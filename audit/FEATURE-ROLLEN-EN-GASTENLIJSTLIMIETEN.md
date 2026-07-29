# Glistix — Feature: rolbeheer-uitbreiding + gastenlijst-limieten

**Datum:** 29 juli 2026
**App:** Glistix (Base44 appId `69160faa53683c5a49a95c7d`)
**Status:** doorgevoerd in de Base44-sandbox, `npm run build` groen. Live zetten = **Publish** in de Base44-editor.

## Wat is er gebouwd

### 1. Gastenlijst-limieten (afgedwongen)
- **Schema `GuestList`** uitgebreid met `add_from_date` en `add_until_date` (tijdslot); `max_capacity` en `max_free_guests` bestonden al maar werden nergens afgedwongen.
- **Nieuw `src/lib/guestListLimits.js`**: telt per persoon (totaal = Σ `party_size`, gratis = Σ `plus_ones_allowed`) en levert `checkGuestListLimit` (capaciteit + gratis + tijdslot) en `checkRoleGuestQuota` (rol-quota).
- **`src/pages/GuestLists.jsx`**: tijdslot-velden (van/tot) in de aanmaak-/bewerkdialog; kaartweergave toont personen-telling, gratis-limiet en tijdslot.
- **Handhaving** in álle toevoeg-flows — `AddGuestPanel.jsx` (los toevoegen, bestand-import, tekst-import) en `GuestDialog.jsx` (nieuw toevoegen): overschrijding van capaciteit/gratis of toevoegen buiten het tijdslot wordt geblokkeerd met een NL-foutmelding. Geldt voor **iedereen**, ook eigenaar/admin (bewuste keuze).

### 2. Gasten-quota per rol
- **Schema `OrganizationRole`** uitgebreid met `max_total_guests` en `max_free_guests`.
- **`RoleDialog.jsx`**: sectie "Gastenlimiet (optioneel)" met beide velden; leeg = onbeperkt.
- **Handhaving**: in dezelfde toevoeg-flows wordt geteld hoeveel personen/gratis de gebruiker (op `added_by_email`) al aan het evenement toevoegde. Platform-admins en de organisatie-eigenaar zijn uitgezonderd.

### 3. Betere rol-toewijzing
- **`OrganizationSettings.jsx`** (leden-tab): inline rol-dropdown per lid — direct wisselen zonder dialog. Het eigenaar-lid houdt een vaste badge (niet wijzigbaar).

### 4. Standaardrollen-presets
- **Nieuw `src/lib/presetRoles.js`**: Beheerder (alles), Event Manager, Check-in medewerker, Promotor (incl. quota 25/10), Staff (alleen-lezen).
- **`RoleManagement.jsx`**: knop **"Standaardrollen toevoegen"** — idempotent (bestaande namen worden overgeslagen).

## Gewijzigde bestanden (Base44-sandbox)
| Bestand | Wijziging |
|---|---|
| entity `GuestList` | + `add_from_date`, `add_until_date` |
| entity `OrganizationRole` | + `max_total_guests`, `max_free_guests` |
| `src/lib/guestListLimits.js` | nieuw — telling + checks |
| `src/lib/presetRoles.js` | nieuw — presets |
| `src/pages/GuestLists.jsx` | tijdslot-UI + personen-telling |
| `src/components/guests/AddGuestPanel.jsx` | handhaving (3 paden) + rol-quota |
| `src/components/guests/GuestDialog.jsx` | handhaving + rol-quota bij nieuw |
| `src/components/roles/RoleDialog.jsx` | quota-velden |
| `src/pages/RoleManagement.jsx` | presets-knop |
| `src/pages/OrganizationSettings.jsx` | inline rol-dropdown |

## Kanttekening
De handhaving is — net als de rest van de app — **client-side**. Wie de API rechtstreeks aanroept kan de limieten omzeilen; harde afdwinging vergt server-side regels (Base44 backend functions/entity rules). Aanbevolen als vervolgstap.
