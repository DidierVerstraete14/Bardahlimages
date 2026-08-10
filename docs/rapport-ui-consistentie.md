# Rapport: UI-consistentiecontrole (taal, datums, navigatie)

**Datum:** 10 augustus 2026
**Scope:** alle pagina's en componenten in de Glistix-app (Base44, app-ID `69160faa53683c5a49a95c7d`)
**Checkpoint na doorvoering:** `UI-consistentie: Nederlandse datums (locale nl), nl-BE overal, knoplabels gelijkgetrokken`

## Samenvatting

De app oogde grotendeels consistent, maar datumweergave was dat niet: het merendeel van de `date-fns`-aanroepen gebruikte **geen Nederlandse locale**, waardoor maand- en dagnamen in het Engels verschenen ("Mar 2026", "Monday" in plaats van "mrt 2026", "maandag"). Daarnaast liep de browser-locale door elkaar (`nl-NL` vs `nl-BE`) en verschilde de hoofdlettering van terug-knoppen per pagina. Alles is gelijkgetrokken en doorgevoerd.

## Bevindingen en fixes

### 1. Engelse maand-/dagnamen in datums (opgelost)

Slechts 6 van de ±17 gebruikersgerichte `format()`-aanroepen gaven `{ locale: nl }` mee. De rest toonde Engelse namen. Gerepareerd (import van `nl` uit `date-fns/locale` toegevoegd waar die ontbrak):

| Bestand | Was | Nu |
|---|---|---|
| `pages/EventDetails.jsx` | `'d MMM yyyy'` zonder locale | met `locale: nl` |
| `pages/InvitationManager.jsx` (2×) | `'dd MMM yyyy HH:mm'` zonder locale | `'d MMM yyyy HH:mm'` met `locale: nl` |
| `components/events/EventCard.jsx` | `'PPP'` zonder locale (→ "August 10th, 2026") | `'d MMM yyyy'` met `locale: nl` |
| `components/analytics/AttendanceChart.jsx` | `'MMM dd'` zonder locale | `'d MMM'` met `locale: nl` |
| `components/guests/GuestRow.jsx` (2×) | `'PPp'` zonder locale | `'d MMM yyyy HH:mm'` met `locale: nl` |

Meteen ook het datumpatroon geüniformeerd naar `d MMM yyyy (HH:mm)` — hetzelfde patroon dat GuestSidebar, GuestConfirmation en RSVPPage al gebruikten. Technische patronen (`yyyy-MM-dd` voor invoervelden, `HH:mm` voor tijden) zijn locale-neutraal en ongemoeid gelaten.

### 2. `nl-NL` vs `nl-BE` door elkaar (opgelost)

`toLocaleString`/`toLocaleDateString` gebruikte 6× `nl-BE` en 4× `nl-NL`; de taalkaarten in UserManagement, OrganizationSettings en Pricing mapten `nl` bovendien op `nl-NL`. Alles gestandaardiseerd op **`nl-BE`** (Vlaamse notatie, de meerderheid in de code en passend bij de doelgroep). Aangepast in: `Dashboard.jsx`, `GuestLists.jsx`, `EventDetails.jsx` (CSV-export), `UserManagement.jsx`, `OrganizationSettings.jsx`, `Pricing.jsx`.

### 3. Terug-knoppen: hoofdlettering verschilde per pagina (opgelost)

"Terug naar Dashboard" naast "Terug naar dashboard" en "Terug naar Evenement" naast "Terug naar evenement". Gestandaardiseerd op kleine letter (Nederlandse zinsstijl, de meerderheid): `CustomFieldsSettings.jsx`, `UserManagement.jsx`, `VenueManagement.jsx`.

## Gecontroleerd en in orde bevonden

- **Check-in Station**: rendert correct fullscreen (`fixed inset-0 z-50`) over de app-schil heen; geen dubbele header.
- **Laadstaten**: 21 pagina's gebruiken hetzelfde spinner-patroon (`animate-spin`); uniform, geen actie nodig.
- **Kale "Terug"-knoppen** op PromoCodeAdmin en VenueUsers: bewust kaal gelaten — die pagina's zijn vanuit meerdere plekken bereikbaar, dus een vaste bestemming in het label zou misleiden.

## Verificatie

Alle 12 gewijzigde bestanden zijn na de aanpassing door esbuild gehaald (JSX-parse): geen fouten, geen dubbele imports. `nl-NL` komt niet meer voor in de codebase.
