---
name: oddluzeni
description: "Sepis insolvencniho navrhu spojeneho s navrhem na povoleni oddluzeni. Generuje XML data pro vyplneni PDF formulare NNPO a pripravuje prilohy (plna moc, pouceni dluznika, seznam majetku, seznam zamestnancu, seznam zavazku). Triggers: /oddluzeni, insolvencni navrh, navrh na oddluzeni, osobni bankrot."
allowed-tools: Read, Glob, Grep, Write, Bash, Task, WebFetch
argument-hint: "[cesta-k-dokumentaci-dluznika]"
---

# Skill: Insolvencni navrh spojeny s navrhem na povoleni oddluzeni

Tento skill automatizuje sepis insolvencniho navrhu spojeneho s navrhem na povoleni oddluzeni podle zakona c. 182/2006 Sb. (insolvencni zakon).

Sepisovatelem navrhu je:
- [JMENO_ADVOKATA], advokat
- [NAZEV_KANCELARE], ICO: [ICO_KANCELARE]
- Sidlo: [SIDLO_KANCELARE]
- ID datove schranky advokata: [ID_DATOVE_SCHRANKY]

---

## Workflow

### Krok 1: Ziskani dokumentace dluznika

Pokud uzivatel zadal cestu k dokumentaci jako argument ($ARGUMENTS), precti vsechny soubory v teto slozce. Jinak se zeptej:

```
Pro sepis insolvencniho navrhu potrebuji dokumentaci dluznika.

Uvedte cestu ke slozce s dokumenty dluznika, nebo mi postupne poskytnete tyto udaje:

A. IDENTIFIKACE DLUZNIKA
   - Jmeno, prijmeni, titul
   - Rodne cislo / datum narozeni
   - Adresa trvaleho bydliste
   - Koresponencni adresa (pokud se lisi)
   - Email, telefon, ID datove schranky
   - Rodinny stav
   - ICO (pokud je podnikatel)
   - Statni prislusnost (pokud neni CR)

B. VYZIVOVACIPOVINNOSTI
   - Pocet vyzivovanych osob (pro nezabavitelnou castku)
   - Vyse mesicniho vyzivneho stanoveneho soudem
   - Pocet osob, kterym je vyzivne stanoveno soudem
   - Vyse dluzneho vyzivneho

C. PRIJMY ZA POSLEDNICH 12 MESICU
   - Celkova vyse
   - Popis zdroju (zamestnani, duchod, OSVC, davky)
   - Doklady (potvrzeni od zamestnavatele, danove priznani)

D. OCEKAVANE PRIJMY V NASLEDUJICICH 12 MESICICH
   - Celkova ocekavana vyse
   - Zdroj prijmu (typ: zamestnani/OSVC/duchod/davky/jine)
   - Platce (nazev, adresa, ICO, vyse mesicniho prijmu)
   - Zavazek treti osoby (darovaci smlouva, smlouva o duchodu)

E. SCHOPNOSTI A MOZNOSTI VYDELEVNE CINNOSTI
   - Nejvyssi dosazene vzdelani (1=zakladni, 2=stredni, 3=vyssi odborne, 4=vysokoskolske)
   - Nazev skoly, obor
   - Dovednosti (ridicsky prukaz, jazyky, certifikaty)
   - Nejvyznamnejsi predchozi zamestnani
   - Schopnost dojizdeni do zamestnani
   - Dalsi faktory ovlivnujici prijmovy potencial

F. ZAVAZKY (DLUHY)
   - Pro kazdy zavazek: veritel, pravni duvod, datum splatnosti, vyse
   - Minimalne 2 veritele s pohledavkami 30+ dnu po splatnosti

G. MAJETEK
   - Nemovitosti
   - Vozidla
   - Bankovni ucty a zustatky
   - Ceniny, sperky, elektronika
   - Pohledavky (vlastni dluznici)
   - Podily ve spolecnostech
   - Majetek ve spolecnem jmeni manzelu

H. DALSI UDAJE
   - Drivejsi insolvencni rizeni (sp.zn., zahranicni?)
   - Osvobozeni od placeni pohledavek v poslednich 20 letech?
   - Zamitnuty navrh pro nepoctivy zamer v poslednich 5 letech?
   - Vzeti navrhu zpet v poslednich 3 mesicich?
   - Zahranicni veritele z EU?
   - Spolecny navrh manzelu?
   - Navrhuje jinou nez zakonnou splatku?
```

---

### Krok 1b: Overeni v ARES (Administrativni registr ekonomickych subjektu)

Po ziskani dokumentace proved automaticke overeni vsech subjektu s ICO pres ARES REST API.

**ARES REST API endpointy:**

Zakladni URL: `https://ares.gov.cz/ekonomicke-subjekty-v-be/rest`

| Endpoint | Popis | Metoda |
|---|---|---|
| `/ekonomicke-subjekty/{ico}` | Zakladni udaje subjektu | GET |
| `/ekonomicke-subjekty-rzp/{ico}` | Zivnostensky rejstrik (zivnosti, provozovny) | GET |
| `/ekonomicke-subjekty-res/{ico}` | Registr ekonomickych subjektu (CZ-NACE, statistika) | GET |
| `/ekonomicke-subjekty-vr/{ico}` | Verejny rejstrik (OR, spolky) | GET |
| `/ekonomicke-subjekty-ceu/{ico}` | Centralni evidence upadcu | GET |

**Postup overeni:**

1. **Overeni dluznika** (pokud ma ICO):
   ```bash
   curl -s -H "Accept: application/json" \
     "https://ares.gov.cz/ekonomicke-subjekty-v-be/rest/ekonomicke-subjekty/{ICO}"
   ```
   Z odpovedi ziskej:
   - `obchodniJmeno` → overeni jmena
   - `bydliste.textovaAdresa` → overeni adresy trvaleho bydliste (NIKOLI `sidlo`, ktere je adresou podnikani)
   - `bydliste.kodKraje` / `bydliste.nazevKraje` → urceni prislusneho krajskeho soudu
   - `dic` → overeni rodneho cisla (format CZrrmmddxxxx)
   - `pravniForma` → "101" = FO podnikajici
   - `datumVzniku` → datum zahajeni podnikani
   - `seznamRegistraci.stavZdrojeCeu` → "NEEXISTUJICI" = zadne drivejsi insolvencni rizeni

2. **Ziskani datove schranky a zivnosti** (ROS zobrazeni):
   Webova stranka `https://ares.gov.cz/ekonomicke-subjekty/ros/{ICO}` zobrazuje:
   - ID datove schranky dluznika (dulezite pro kontaktni udaje v NNPO)
   - Seznam zivnosti (aktivni/zaniklych)
   - Seznam provozoven

3. **Overeni CEU** (Centralni evidence upadcu):
   ```bash
   curl -s -H "Accept: application/json" \
     "https://ares.gov.cz/ekonomicke-subjekty-v-be/rest/ekonomicke-subjekty-ceu/{ICO}"
   ```
   - Odpoved `NENALEZENO` = zadny zaznam o upadku (dobre)
   - Pokud nalezen zaznam → overit, zda nespadaji do § 395 IZ (nepoctivy zamer)

4. **Overeni veritelu** (pro kazdeho veritele s ICO):
   ```bash
   curl -s -H "Accept: application/json" \
     "https://ares.gov.cz/ekonomicke-subjekty-v-be/rest/ekonomicke-subjekty/{ICO_VERITELE}"
   ```
   Z odpovedi overit:
   - `obchodniJmeno` → spravny nazev veritele
   - `sidlo.textovaAdresa` → spravna adresa sidla
   - `ico` → potvrzeni ICO
   - `pravniForma` → typ subjektu (s.r.o., a.s., n.p., obec atd.)

**Vysledky overeni zobraz uzivateli ve forme tabulky:**

```
ARES OVERENI - vysledky
========================

DLUZNIK:
  Jmeno:           [ares] vs [dokumenty] → OK / NESOULAD
  Trvale bydliste: [ares bydliste] vs [dokumenty] → OK / NESOULAD
  Datova schranka: [ID z ARES]
  Stav v CEU:      Zadny zaznam / ZAZNAM NALEZEN (!)
  Aktivni zivnosti: X
  Zaniklych zivnosti: Y

VERITELE:
  1. [nazev] ICO [ico] → OK (adresa overena)
  2. [nazev] ICO [ico] → OK (adresa overena)
  ...
```

**Data z ARES pouzij automaticky:**
- Datovou schranku dluznika → vyplnit do XML (`<dat_schr>`)
- Overene adresy veritelu → pouzit v seznamu zavazku
- Kraj z trvaleho bydliste dluznika (`bydliste.kodKraje`) → potvrdit/urcit prislusny krajsky soud
- Stav v CEU → nastavit `<drivejsi_rizeni switch="0">` nebo `"1"`

---

### Krok 2: Analyza a validace dat

Po ziskani vsech udaju a overeni v ARES proved:

1. **Overeni podminek oddluzeni:**
   - Minimalne 2 veritele s pohledavkami 30+ dnu po splatnosti
   - Schopnost splacet minimalni mesicni splatku (odmena IS + hotove vydaje + stejna castka veritelum)
   - Zadny priznak nepoctiveho zameru

2. **Urceni prislusneho soudu** podle trvaleho bydliste dluznika (potvrdit krajem z ARES pole `bydliste.kodKraje`):

   | kodKraje (ARES) | Kraj | Kod soudu |
   |---|---|---|
   | 19 | Praha | MSPH |
   | 27 | Stredocesky | KSPH |
   | 35 | Jihocesky | KSCB |
   | 43 | Plzensky | KSPL |
   | 51 | Karlovarsky | KSPL |
   | 60 | Ustecky | KSUL |
   | 78 | Liberecky | KSLB |
   | 86 | Kralovehradecky | KSHK |
   | 94 | Pardubicky | KSPA |
   | 108 | Vysocina | KSBR |
   | 116 | Jihomoravsky | KSBR |
   | 124 | Olomoucky | KSOL |
   | 132 | Zlinsky | KSBR |
   | 140 | Moravskoslezsky | KSOS |

3. **Vypocet predpokladaneho plneni veritelum** na zaklade prijmu a srazek.

---

### Krok 3: Generovani XML dat pro formular NNPO

Vygeneruj XML soubor `NNPO_data.xml` ve formatu XFA data podle schematu v [reference/xml_template.xml](reference/xml_template.xml).

Struktura XML:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<xfa:datasets xmlns:xfa="http://www.xfa.org/schema/xfa-data/1.0/">
<xfa:data>
<NNPO verze="1.5.4">
  ... vsechna data podle schematu ...
</NNPO>
</xfa:data>
</xfa:datasets>
```

Pravidla pro XML:
- Prazdne elementy zapisuj jako `<element/>` (self-closing)
- Switch atributy: "1" = Ano, "0" = Ne, "" = nevyplneno
- Datumy ve formatu YYYY-MM-DD
- Castky jako cisla s max. 2 desetinnymi misty (tecka jako separator)
- Kody soudu: MSPH, KSPH, KSCB, KSPL, KSUL, KSLB, KSHK, KSPA, KSBR, KSOS, KSOL
- Rejstrik je vzdy "INS"
- Verze formulare: "1.5.4"

Kompletni strukturu XML viz [reference/xml_template.xml](reference/xml_template.xml).

**DULEZITE:** XML soubor uloz do slozky s dokumentaci dluznika jako `NNPO_data.xml`. Uzivatel pak importuje data do PDF formulare pres Adobe Acrobat: Formular → Importovat data.

---

### Krok 4: Generovani priloh

Pro kazdy z 5 priloh pouzij Task tool s subagent_type `legal-document-formatter` a vygeneruj .docx soubor.

#### Priloha 1: Plna moc
Viz sablona v [templates/plna_moc.md](templates/plna_moc.md).
Uloz jako `Priloha_1_Plna_moc.docx`.

#### Priloha 2: Pouceni dluznika
Viz sablona v [templates/pouceni_dluznika.md](templates/pouceni_dluznika.md).
Uloz jako `Priloha_2_Pouceni_dluznika.docx`.

#### Priloha 3: Seznam majetku
Viz sablona v [templates/seznam_majetku.md](templates/seznam_majetku.md).
Uloz jako `Priloha_3_Seznam_majetku.docx`.

#### Priloha 4: Seznam zamestnancu
Viz sablona v [templates/seznam_zamestnancu.md](templates/seznam_zamestnancu.md).
Uloz jako `Priloha_4_Seznam_zamestnancu.docx`.

#### Priloha 5: Seznam zavazku
Viz sablona v [templates/seznam_zavazku.md](templates/seznam_zavazku.md).
Uloz jako `Priloha_5_Seznam_zavazku.docx`.

Vsechny prilohy uloz do slozky s dokumentaci dluznika.

---

### Krok 5: Kontrolni seznam a odeslani

Po vygenerovani vsech dokumentu zobraz kontrolni seznam:

```
KONTROLNI SEZNAM - Insolvencni navrh na povoleni oddluzeni

Vygenerovane dokumenty:
[x] NNPO_data.xml - data pro import do PDF formulare
[x] Priloha_1_Plna_moc.docx
[x] Priloha_2_Pouceni_dluznika.docx
[x] Priloha_3_Seznam_majetku.docx
[x] Priloha_4_Seznam_zamestnancu.docx
[x] Priloha_5_Seznam_zavazku.docx

Dalsi prilohy k doplneni rucne:
[ ] Potvrzeni o prijmech za 12 mesicu
[ ] Vypis z rejstriku trestu (ne starsi 3 mesice)
[ ] Darovaci smlouva / smlouva o duchodu (pokud existuje)
[ ] Cestne prohlaseni manzelu o SJM (pri spolecnem navrhu)

Dalsi kroky:
1. Otevrete formular NNPO v Adobe Acrobat Reader
2. Importujte data: Formular → Importovat data → vyberte NNPO_data.xml
3. Zkontrolujte vyplneny formular
4. Pripojte elektronicky podpis (konverze plne moci)
5. Podejte pres ISIR nebo datovou schrankou na prislusny krajsky soud
```

---

## Reference

- Kompletni XML sablona: [reference/xml_template.xml](reference/xml_template.xml)
- XSD schema: stahnout z https://isir.justice.cz (verze 1.5.2)
- Pokyny k vyplneni: verze v1.5.0 od 1. 10. 2024
- Formular PDF: verze 1.5.4 ze 17. 9. 2025
- Zakon: c. 182/2006 Sb., o upadku a zpusobech jeho reseni (insolvencni zakon)
- Odmena sepisovatele: max. 4 000 Kc bez DPH (jednotlivec), 6 000 Kc bez DPH (manzele)
- ARES REST API: https://ares.gov.cz/ekonomicke-subjekty-v-be/rest/ekonomicke-subjekty/{ico}
- ARES Swagger UI: https://ares.gov.cz/swagger-ui/
- ARES webove rozhrani: https://ares.gov.cz/ekonomicke-subjekty
- ARES ROS detail (s datovou schrankou): https://ares.gov.cz/ekonomicke-subjekty/ros/{ico}
