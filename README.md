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

- JUDr. Vojtěch Říha, Ph.D., advokát
- RIHA legal advokátní kancelář s.r.o., IČO: 21006601
- Sídlo: Moulíkova 2239/3, Smíchov, 150 00 Praha
- ID datové schránky advokáta: y2btkj3

## Reference

- Formulář NNPO verze 1.5.4 (17. 9. 2025)
- Pokyny k vyplnění verze v1.5.0 (1. 10. 2024)
- Zákon č. 182/2006 Sb., o úpadku a způsobech jeho řešení (insolvenční zákon)
- ARES REST API: https://ares.gov.cz/swagger-ui/

## Licence

Proprietární - pouze pro interní použití RIHA legal advokátní kancelář s.r.o.
