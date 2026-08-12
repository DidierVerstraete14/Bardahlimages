# Rapport: gap-analyse Glistix t.o.v. Attendium-referentiedocument

**Datum:** 12 augustus 2026
**Bron:** `referentie-attendium-design-ux.md` (design- & UX-analyse van Attendium, aangeleverd door de gebruiker)
**Scope:** de "concrete opdracht-set" (sectie 14) en de 10 principes (sectie 12) afgezet tegen de huidige Glistix-app (Base44, app-ID `69160faa53683c5a49a95c7d`)
**Checkpoint na doorvoering:** `Attendium-referentie fase A+B: toetsenbordnavigatie, undo-toast, bredere zoek, tekst-import met live preview`

## Samenvatting

Glistix dekte vooraf al opvallend veel van de opdracht-set — op meerdere punten zelfs breder dan Attendium zelf (getypeerde aangepaste velden, bulk in-/uitchecken, zoeken buiten de naam, offline modus gratis). De echte gaten waren precies de vier punten die het referentiedocument als "Attendium-zwaktes / jouw kans" aanmerkt: toetsenbordnavigatie, undo-toast, live parse-preview bij tekst-import en de uitgebreide plak-syntax. Die vier zijn in deze ronde doorgevoerd. Het volledige zwart-canvas-herontwerp (sectie 2/14) is bewust **niet** uitgevoerd — dat is een identiteitswissel die een aparte beslissing verdient.

## 1. Wat Glistix al had (per punt van de opdracht-set)

| Opdracht-set-punt | Status vóór deze ronde |
|---|---|
| Permanente `ingecheckt / totaal`-teller in header | ✅ In Check-in Station-header (voortgangsbalk + % + n/m), maar verborgen op mobiel |
| Gedimde/gemarkeerde ingecheckte rijen + gekleurde lijst-labels | ✅ Groen gedimd met doorhaling; gastenlijstkleur als label in elke rij |
| Instant client-side zoeken vanaf eerste letter | ✅ Naam, e-mail, telefoon, bedrijf (Attendium zoekt alleen op naam) |
| Detailpaneel in-place naast de lijst (geen modal) | ✅ `InlineGuestPanel` als vast zijpaneel (desktop) / overlay (mobiel) |
| `Inchecken n / m`-breukmodel voor groepen | ✅ Gedeeltelijke groeps-check-in met `plus_ones_checked_in` |
| Drie invoermodi: enkel / bestand / tekst-plakken | ✅ `AddGuestPanel` met drie tabs, syntax-uitleg permanent onder het veld |
| Aangepaste velden met type en validatie | ✅ text/number/email/phone/date/select/checkbox/paragraph + verplicht-vlag — Attendium heeft dit niet |
| Bulk-acties | ✅ Bulk in-/uitchecken, exporteren, verwijderen (Attendium: alleen verwijderen). Verplaatsen/taggen ontbreekt nog |
| Check-ins per uur van de dag | ✅ `CheckInTimeline` (staafdiagram per uur, VIP vs. regulier) in Statistieken |
| Filters-popover (ingecheckt / lijsten / toegevoegd door, Alle/Geen) | ✅ In het zoekpaneel van het Check-in Station |
| Offline modus + wachtrij + sync | ✅ Gratis inbegrepen (bij Attendium betaald) |
| QR: camera-scanner + USB-scanner + echte QR-codes | ✅ Sinds de vorige ronde (jsQR + getUserMedia) |
| Donkere modus met licht-optie | ✅ Check-in Station donker met `ThemeToggle` |

## 2. Doorgevoerd in deze ronde

### Fase A — check-in-ergonomie (`NameSearchCheckIn.jsx`, `CheckInStation.jsx`)

1. **Toetsenbordnavigatie** in het Check-in Station, exact zoals het document vraagt: `↑`/`↓` selecteert een rij (zichtbare indigo focus-ring, scrollt mee), `Enter` checkt de gemarkeerde gast direct in (alleenstaande gast) of opent het detailpaneel (groep of al ingecheckt — bewust géén blinde checkout op Enter), `/` springt naar het zoekveld, `Esc` wist de zoekopdracht of sluit het paneel. Enter werkt ook vanuit het zoekveld wanneer er precies één resultaat is — dat maakt "naam typen + Enter" en USB-scanners snel. Een discrete kbd-hint onder de zoekbalk (alleen desktop) documenteert de sneltoetsen in context (principe 7).
2. **Undo-toast (8 s)** na elke check-in, gedeeltelijke check-in en checkout: "Ongedaan maken" herstelt de volledige vóór-status (`checked_in`, `check_in_time`, `plus_ones_checked_in`). Werkt ook offline (gaat door dezelfde wachtrij). Meteen een **bug** gerepareerd: een gedeeltelijke groeps-check-in (bv. 2 van 5) toonde voorheen de toast "uitgecheckt"; nu toont hij "✓ ingecheckt (2/5)".
3. **Zoeken uitgebreid** naar tafelnummer en aangepaste-veldwaarden (via memo-indexen, dus zonder per-toetsaanslag-kosten) — het document noemt dit expliciet als Attendium-zwakte.
4. **Teller ook op mobiel**: het `n/m`-cijfer en percentage staan nu op elk schermformaat in de header; alleen de voortgangsbalk blijft desktop-only.

### Fase B — tekst-import met live preview (`AddGuestPanel.jsx`)

5. **Volledige Attendium-syntax** ondersteund, bovenop de bestaande komma-vorm: `John Smith +2`, `Katy Perry +1 (2)` (gratis tickets tussen haakjes), `Hugo Smith (gratis)` (hele groep gratis), `Jon Snow +2 (3) : King of the north` (notitie na dubbele punt), `John Smith john@attendium.com` (e-mail wordt overal op de regel herkend). De bestaande vorm `Naam, tickets, gratis, notities, email` blijft ongewijzigd werken.
6. **Live parse-preview** — de uitbreiding die het document zelf voorstelt en die Attendium mist: onder het plakvak verschijnt direct per regel de herkende gast met badges voor +X, gratis, e-mail en notitie; niet-parsebare regels (bv. zonder letters) worden rood gemarkeerd met een teller "X regels niet herkend". De importknop toont het aantal ("14 gasten importeren") en importeert alleen de geldige regels.
7. Syntax-uitleg onder het veld bijgewerkt naar de nieuwe vormen; parser geverifieerd tegen alle voorbeelden uit het referentiedocument (node-test) en de volledige `vite build` slaagt.

## 3. Bewust niet (nu) gedaan — met aanbeveling

1. **Zwart-canvas-herontwerp** (drie kolommen op `#000`, lime-accent, Inter, 4–6px radius): dit is een merkidentiteitswissel — Glistix is nu indigo-op-licht met een donker Check-in Station. De deur-context (donker, kleur = betekenis) is in het station al gevolgd. Volledige overname verdient een expliciete go/no-go van de eigenaar; technisch zou het neerkomen op een token-laag (CSS-variabelen) + herstyling van ±40 pagina's. **Aanbeveling: eerst beslissen of Glistix het Attendium-uiterlijk wil spiegelen of een eigen identiteit houdt.**
2. **Sidebar-swap-navigatie** (niveau-wissel in dezelfde kolom): forse IA-refactor van alle event-schermen; zinvol om te combineren mét punt 1 als dat doorgaat.
3. **Skeleton-states per route** (nooit >300 ms leeg scherm): goede volgende fase; de `Skeleton`-component bestaat al in de ui-map, het is vooral 21 spinner-pagina's omzetten.
4. **Undo na verwijderen** (gast heraanmaken vanuit de toast): kandidaat volgende fase; vraagt her-creatie van het record inclusief gekoppelde custom-field-waarden en tags.
5. **Bulk verplaatsen naar andere gastenlijst + bulk taggen**: kandidaat volgende fase; de selectie-infrastructuur in EventDetails bestaat al.
6. **Virtueel scrollen** van de gastenlijst: pas relevant boven ±1.000 zichtbare rijen; nu al gedempt doordat zoekresultaten op 12 worden afgekapt.
7. **Analytics: per-kaart inline filters + refresh-timestamp**: laag risico, medium waarde; per-uur-grafiek bestaat al.

## 4. Verificatie

- Parser getest met node tegen alle syntaxvoorbeelden uit het referentiedocument (incl. randgevallen: lege regel, regel zonder letters) — alle uitkomsten correct.
- Volledige `vite build` zonder fouten; toetsenbord-hint, undo-toast en preview aantoonbaar aanwezig in de productiebundel.
- Base44-checkpoint aangemaakt vóór deze rapportcommit.
