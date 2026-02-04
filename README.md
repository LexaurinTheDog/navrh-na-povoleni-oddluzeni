# Návrh na povolení oddlužení

Claude Code skill pro automatizovaný sepis insolvenčního návrhu spojeného s návrhem na povolení oddlužení podle zákona č. 182/2006 Sb. (insolvenční zákon).

## Popis

Tento skill automatizuje přípravu kompletní dokumentace pro oddlužení fyzických osob (osobní bankrot) v České republice:

- **NNPO_data.xml** - data pro import do PDF formuláře NNPO
- **Plná moc** - pro sepis a podání návrhu
- **Poučení dlužníka** - o povinnostech v oddlužení
- **Seznam majetku** - dle § 103 a § 104 IZ
- **Seznam zaměstnanců** - dle § 103 a § 104 IZ
- **Seznam závazků** - dle § 103 a § 104 IZ

## Instalace

Zkopírujte obsah tohoto repozitáře do složky `~/.claude/skills/oddluzeni/`.

## Použití

V Claude Code spusťte skill příkazem:

```
/oddluzeni [cesta-k-dokumentaci-dluznika]
```

Nebo použijte triggery:
- `/oddluzeni`
- `insolvenční návrh`
- `návrh na oddlužení`
- `osobní bankrot`

## Sepisovatel

Před použitím skillu je nutné nastavit údaje sepisovatele v souborech:
- `SKILL.md` - hlavní definice
- `templates/*.md` - šablony dokumentů
- `reference/xml_template.xml` - XML šablona

Placeholdery k nahrazení:
- `[JMENO_ADVOKATA]` - jméno advokáta s tituly
- `[NAZEV_KANCELARE]` - název advokátní kanceláře
- `[ICO_KANCELARE]` - IČO kanceláře
- `[SIDLO_KANCELARE]` - sídlo kanceláře
- `[ID_DATOVE_SCHRANKY]` - ID datové schránky advokáta
- `[EV_CISLO_CAK]` - evidenční číslo ČAK

## Reference

- Formulář NNPO verze 1.5.4 (17. 9. 2025)
- Pokyny k vyplnění verze v1.5.0 (1. 10. 2024)
- Zákon č. 182/2006 Sb., o úpadku a způsobech jeho řešení (insolvenční zákon)
- ARES REST API: https://ares.gov.cz/swagger-ui/

## Licence

MIT License - viz soubor LICENSE
