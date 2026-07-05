# Glistix — Volledige flow-audit (Base44)

**Datum:** 5 juli 2026
**App:** Glistix (Base44 appId `69160faa53683c5a49a95c7d`)
**Scope:** volledige flow — authenticatie, dashboard, evenementen, gastenbeheer, check-in, tafelplan, rollen/permissies, abonnementen/betalingen (Mollie), uitnodigingen/RSVP, publieke pagina's.

> Statuslegenda: 🔴 kritiek · 🟠 hoog · 🟡 medium · ⚪ laag
> Voor elke bevinding met code staat een exacte fix in `audit/patches/fixes.md`.

---

## Samenvatting

Het platform zit functioneel goed in elkaar, maar bevat **7 kritieke** en **10+ hoge/medium bugs**, geconcentreerd in vier gebieden:

1. **Betalingen & limieten** — de "eenmalige event-unlock"-flow kan geld innen zonder iets te leveren; plan-limieten zijn op 3 plaatsen verschillend gedefinieerd; promocodes worden volledig client-side geactiveerd (manipuleerbaar).
2. **Check-in** — "Uitchecken/ongedaan maken" vanuit het Check-in Station checkt gasten juist opnieuw ín; tafeltoewijzing bij check-in behandelt elke tafel met 1 gast als vol; offline gefaalde check-ins gaan stil verloren.
3. **Data-scheiding & permissies** — het dashboard laadt events van álle organisaties en toont venue-loze events van andere orgs; org-eigenaren zonder platform-admin-rol krijgen nergens beheerknoppen; gastenlijst-restricties gelden niet in het Check-in Station.
4. **RSVP/uitnodigingen** — token-lookup breekt na 500 uitnodigingen; vervaldatum (`expires_at`) wordt nergens gecontroleerd.

---

## 🔴 Kritieke bugs

### K1. Event-unlock-aankoop levert niets (geld betaald, niets ontgrendeld)
- **Waar:** `src/pages/Dashboard.jsx` (UnlockLimitDialog-flow) + `src/components/events/UnlockLimitDialog.jsx` + `base44/functions/mollieWebhook/entry.ts`
- **Wat:** Bij het bereiken van de maandlimiet wordt het event **niet** aangemaakt; `UnlockLimitDialog` krijgt `eventId={null}` en start een Mollie-betaling met `eventId: null` in de metadata. De webhook doet vervolgens `Event.update(null, …)` → faalt. Bovendien wordt de `onUnlocked`-callback **nergens** aangeroepen (de dialog redirect naar Mollie en verlaat de pagina), dus de aanmaaklogica in `onUnlocked` is dood.
- **Gevolg:** klant betaalt €20–€120 en krijgt geen event.
- **Fix:** maak het event éérst aan (status `draft`) vóór de redirect naar Mollie, en geef dat `eventId` mee aan de betaling zodat de webhook het correct kan ontgrendelen. Zie patch P1.

### K2. Uitchecken vanuit Check-in Station checkt gasten opnieuw ín
- **Waar:** `src/pages/EventDetails.jsx` → `handleCheckIn(guest)` i.c.m. `NameSearchCheckIn.jsx` en `QRScanCheckIn.jsx`
- **Wat:** `handleCheckIn` beslist in-/uitchecken op basis van `guest.checked_in` van het **doorgegeven object**. `NameSearchCheckIn` en `QRScanCheckIn` geven bij "Uitchecken"/undo echter een al-gewijzigd object door (`{ ...guest, checked_in: false }`) → de handler denkt dat de gast nog niet binnen is en checkt hem opnieuw in (of opent de custom-fields dialog).
- **Gevolg:** uitchecken/ongedaan maken werkt niet vanuit naamzoek- en QR-check-in; tellingen kloppen niet meer op de avond zelf.
- **Fix:** in `handleCheckIn` altijd het actuele record uit `guests` opzoeken op `id` en dáárop de toggle-beslissing baseren. Zie patch P2.

### K3. RSVP-links breken na 500 uitnodigingen
- **Waar:** `base44/functions/invitationRSVP/entry.ts:10`
- **Wat:** token-lookup via `Invitation.list('-created_date', 500)` + `.find()`. Uitnodiging #501+ (of oudere) wordt niet gevonden → gast krijgt "Uitnodiging niet gevonden".
- **Fix:** `Invitation.filter({ token })`. Zie patch P3.

### K4. Promocode-activatie volledig client-side (manipuleerbaar + race)
- **Waar:** `src/pages/Pricing.jsx` → `handleApplyPromo`
- **Wat:** de browser doet zelf `Organization.update` (plan → professional) en `PromoCode.used_count + 1`. Iedereen met een account kan dit via de console nadoen zonder geldige code; twee gelijktijdige inwisselingen omzeilen `max_uses`.
- **Fix (structureel):** verplaats naar een backend-function (`redeemPromoCode`) met service-role, atomaire validatie en увеличение van `used_count`. Zie patch P4 (backend-function meegeleverd). Zelfde geldt voor "abonnement opzeggen" en "downgrade naar free" (nu ook client-side).

### K5. Plan-limieten op 3 plaatsen verschillend — betalende klanten krijgen minder dan beloofd
- **Waar:** `src/lib/planLimits.js` vs `base44/functions/mollieWebhook/entry.ts` vs `src/components/utils/subscriptionLimits.jsx`
- **Wat:**
  - Pricing/vertalingen beloven: **onbeperkt evenementen** voor lite/starter/professional; lite = 300 gasten.
  - `mollieWebhook` zet na betaling echter `max_events: 5` (lite) en `20` (starter) — in strijd met wat de klant koopt.
  - Pricing-promopad zet `max_events: 10` bij onbeperkte plannen (`?? 10`), webhook gebruikt `?? 999`.
  - `subscriptionLimits.jsx` bevat een derde, afwijkende tabel zonder `lite` (zou crashen) — gelukkig **dode code** → verwijderen.
- **Fix:** één bron van waarheid (`src/lib/planLimits.js`), webhook en promopad daarop aligneren. Zie patch P5.

### K6. Publieke gast-bevestigingspagina maakt gastenlijst opvraagbaar
- **Waar:** `src/pages/GuestConfirmation.jsx` (+ entity-permissies van `Guest`)
- **Wat:** de publieke pagina query't `Guest`/`Event`/`GuestList` rechtstreeks vanuit de browser zonder login. Dat werkt alleen doordat de Guest-entity publiek leesbaar is → **iedere bezoeker kan de volledige gastendatabase opvragen**. De fallback `Guest.filter({ id: token })` maakt bovendien enumeratie via gast-ID's mogelijk.
- **Fix:** (a) verwijder de id-fallback (patch P6); (b) structureel: verplaats de lookup naar een backend-function (zoals `invitationRSVP`) en zet de Guest-entity read-permissie dicht. **Controleer in Base44 → Entities → Guest de permissies** (schema toont `read: true`).

### K7. Cross-org datalek + verkeerde limiettelling op Dashboard
- **Waar:** `src/pages/Dashboard.jsx` → events-query
- **Wat:** bij >1 venue haalt het dashboard **alle events van de hele app** op (`Event.list('-start_date')`) en filtert client-side: `!e.venue_location_id || venueIds.includes(…)`. Gevolg: (a) events zonder venue-koppeling van **andere organisaties** verschijnen in jouw dashboard en tellen mee in jouw maandlimiet; (b) alle event-data van andere orgs komt naar de browser; (c) `Event.list` zonder limiet kan afkappen.
- **Fix:** filter server-side op `organization_id` (+ venue-fallback voor legacy events). Zie patch P7.

---

## 🟠 Hoge bugs

### H1. Org-eigenaren zonder platform-admin zien geen beheerfuncties
- **Waar:** `src/hooks/useEffectivePermissions.js` + `src/pages/EventDetails.jsx` (navItems, Edit-knop, instellingen-menu — alles gegate op `isAdmin` = `User.role === 'admin'`)
- **Wat:** eigenaarschap van de organisatie (`Organization.owner_email`) geeft géén rechten in de permissie-hook; wie zijn org via de NoOrgBanner aanmaakt krijgt een `OrganizationMember` **zonder** `role_id` → valt terug op view-only. Resultaat: een betalende eigenaar kan zijn eigen event niet bewerken, geen tafelplan/machtigingen/tags openen.
- **Fix:** in `useEffectivePermissions` de organisatie ophalen en bij `owner_email === userEmail` FULL_PERMISSIONS geven; in `EventDetails` de nav/knoppen gaten op `userPermissions.can_manage_event`/`can_manage_roles`/`can_manage_tables` i.p.v. alleen `isAdmin`. Zie patch P8.

### H2. Import omzeilt gastenlimiet volledig
- **Waar:** `src/components/guests/AddGuestPanel.jsx` (`handleFileImport`, `handleTextImport`) en `src/components/guests/ImportGuestsDialog.jsx`
- **Wat:** alleen de enkel-gast-flow checkt `checkGuestLimit`; via bestand/tekst kan een free-org (50) duizenden gasten importeren. Import-gasten krijgen ook geen `unique_access_token`/`added_by_email` (bevestigingslink en "toegevoegd door"-filter werken dan niet).
- **Fix:** limiet vóór de import-loop afdwingen + token/e-mail meegeven. Zie patch P9.

### H3. Tafeltoewijzing bij check-in: elke tafel met 1 gast is "vol"
- **Waar:** `src/components/checkin/NameSearchCheckIn.jsx` → `const isFull = assignedGuests > 0 && !isSelected;`
- **Wat:** capaciteit wordt genegeerd; vanaf de eerste toegewezen gast is de tafel niet meer kiesbaar.
- **Fix:** vergelijk met `table.capacity` en tel `party_size` mee. Zie patch P10.

### H4. Offline check-ins kunnen stil verloren gaan
- **Waar:** `src/pages/EventDetails.jsx` → `syncQueue`
- **Wat:** elk queue-item wordt in een lege `catch {}` geprobeerd, daarna wordt de **hele queue gewist** — mislukte updates (bv. door rate limit of tijdelijke fout) zijn definitief weg.
- **Fix:** mislukte items terug in de queue zetten en melden. Zie patch P11.

### H5. Webhook-/betaalpagina-verificatie
- **Waar:** `src/pages/PaymentReturn.jsx` + `base44/functions/mollieCreatePayment/entry.ts`
- **Wat:** (a) PaymentReturn toont "Betaling geslaagd!" puur op basis van het `?status=success` URL-param — Mollie stuurt ook mislukte/verlopen betalingen naar de redirectUrl. (b) `mollieCreatePayment` controleert niet of de gebruiker bij `organizationId`/`eventId` hoort.
- **Fix:** neutrale copy ("wordt verwerkt … activatie volgt na bevestiging") — patch P12; structureel: status-verificatie-endpoint + ownership-check in de function.

### H6. Vervallen uitnodigingen blijven bruikbaar
- **Waar:** `base44/functions/invitationRSVP/entry.ts`
- **Wat:** `expires_at` wordt nergens gecheckt; status `expired` wordt nooit gezet.
- **Fix:** onderdeel van patch P3.

### H7. `syncVenueRoleToEvents` schrijft veld buiten schema + kan rechten escaleren
- **Waar:** `base44/functions/syncVenueRoleToEvents/entry.ts`
- **Wat:** (a) schrijft `_venue_synced`, geen schemaveld van `EventUserPermission`; (b) `can_manage_roles || can_manage_event` ⇒ `can_admin: true` ⇒ via `useEffectivePermissions` FULL_PERMISSIONS incl. rollenbeheer; (c) bij verwijdering van een VenueRole blijven de gesynchroniseerde event-permissies bestaan.
- **Fix:** `_venue_synced` verwijderen (patch P13); delete-sync en fijnmaziger mapping als verbetering.

---

## 🟡 Medium

| # | Waar | Wat | Patch |
|---|------|-----|-------|
| M1 | `Dashboard.jsx` | Kapotte admin-check: `orgMembers[0]?.role` bestaat niet in het schema (wel `role_id`); alleen NoOrgBanner schrijft een los `role`-veld | P7 (deels), advies A3 |
| M2 | `Dashboard.jsx` | Maandlimiet telt sjablonen mee; headerteller telt unlocked events wél mee (inconsistent met de echte check) | P7 |
| M3 | `Dashboard.jsx` | `getGuestStats` leest niet-bestaand veld `rsvp_status`; "totaal gasten"-statistiek telt alleen de eerste 8 zichtbare events | P7 |
| M4 | `EventDetails.jsx` | Gastenlijst-restricties (`allowedGuestListIds`) gelden niet in het Check-in Station → beperkte gebruikers zien/checken alle gasten | P2 |
| M5 | `EventDetails.jsx` | Bulk-inchecken zet `plus_ones_checked_in` niet → gasten blijven "gedeeltelijk" na bulk-actie | P2 |
| M6 | `EventDetails.jsx` | QR-backfill vuurt bij openen Check-in Station 1 update per gast zonder QR (rate-limit risico bij grote lijsten) | advies |
| M7 | `InvitationManager.jsx` | `createMutation` heeft geen `onError` (stille fout); verwijderen zonder bevestiging | P14 |
| M8 | `QRScanCheckIn.jsx` | Toont niet-bestaand `rsvp_status`-veld als badge ("undefined"); volledig Engels terwijl de rest NL is | P15 |
| M9 | `AddGuestPanel.jsx` | Fout bij opslaan enkel-gast wordt niet getoond (try/finally zonder catch); verplichte custom fields niet afgedwongen | P9 |
| M10 | `Pricing.jsx` | Opzeggen/downgraden client-side (zelfde patroon als K4) | A2 |
| M11 | Diverse | `Guest.status` (invited/checked_in/no_show) wordt nergens bijgewerkt — dubbel met `checked_in`; kies één | A4 |

## ⚪ Laag / opruimen

- **Dode code:** `src/components/utils/subscriptionLimits.jsx` (nergens geïmporteerd, zou crashen op 'lite') en `src/pages/Passes.jsx` (0 bytes) → verwijderen.
- `QRCodeDisplay.jsx` toont geen echte QR-code (alleen een icoon + tekst) — gebruik bv. `qrcode.react`.
- Vertalingen: `de` (Duits) bestaat in `translations.jsx` maar niet in `UserPreferences.language`-enum; Login/Register/QRScan zijn hardcoded Engels, veel andere pagina's hardcoded Nederlands (bypassen `t()`).
- Features-teksten vs `planLimits.js`: lite "2 gebruikers" vs code 5; starter "5" vs code 10 — gelijktrekken.
- `Event.date` is deprecated in het schema; controleer of nergens meer gebruikt (Dashboard/EventDialog gebruiken start_date — ok).
- `AppHeader` haalt de user zelf op i.p.v. via `AuthContext`; geen organisatie-switcher in de UI hoewel het datamodel multi-org ondersteunt.

---

## Verbetervoorstellen (aanbevolen roadmap)

**A1. Betaal-flow robuust maken (hoogste prioriteit)**
Webhook-idempotentie (paymentId opslaan), ownership-check in `mollieCreatePayment`, verificatie-endpoint voor PaymentReturn, en de event-unlock-flow uit K1/P1. Overweeg Mollie Subscriptions i.p.v. handmatige verlengingsdatums (nu verlengt er feitelijk niets automatisch — `next_renewal_date` is puur cosmetisch, na een maand verloopt niets en blijft het plan actief zonder betaling).

**A2. Autorisatie serverzijdig afdwingen**
Alle plan-/abonnementsmutaties (promo, cancel, downgrade) en gevoelige reads (GuestConfirmation) naar backend-functions; entity-permissies in Base44 aanscherpen (Guest/Organization/PromoCode niet publiek/lezend-schrijfbaar voor iedere user). Dit is belangrijker dan welke UI-fix ook: de UI-permissies (useEffectivePermissions) zijn nu **alleen** client-side.

**A3. Eén rolmodel voor organisaties**
`OrganizationMember.role_id` consequent gebruiken (NoOrgBanner schrijft nu een niet-bestaand `role`-veld), een systeemrol "Owner" aanmaken bij org-creatie, en `owner_email` in `useEffectivePermissions` respecteren (P8).

**A4. Check-in datamodel opschonen**
`checked_in` (bool) + `status` (enum) + `plus_ones_checked_in` (die feitelijk "personen ingecheckt" betekent) lopen door elkaar. Voorstel: `persons_checked_in` als enige teller, `checked_in` afgeleid, `status` overal bijwerken of schrappen; no-show-markering toevoegen na afloop event.

**A5. Performance & schaal**
- `Event.list()`/`Guest.filter()` zonder limiet: controleer de server-side default en pagineer (>500 gasten per event is realistisch bij starter/professional).
- Dashboard doet nu 10+ queries per load; overweeg één backend-function "dashboardSummary".
- Custom field values & guest tags worden per gast opgehaald (N queries, in chunks van 50) — één `filter({ event_id })` als het datamodel een event_id op die records krijgt.

**A6. UX-quick wins**
Organisatie-switcher in de header; bevestigingsdialogen consistent (nu mix van `confirm()` en niets); toasts i.p.v. `alert()`; check-in station: partial check-in zichtbaar in "Alle"-pill-telling; duplicaat-event kopieert ook CheckInConfig/CategoryColors/CustomFieldDefinitions.

---

## Toepassing van de fixes

Alle patches staan als exacte oud→nieuw-blokken in **`audit/patches/fixes.md`**, klaar om toe te passen op de Base44-app (via de Base44 editor, of door Claude in een nieuwe sessie met werkende schrijfrechten — de tool-permissies stonden in deze sessie geblokkeerd door een verbroken permission-stream, vandaar deze route).
