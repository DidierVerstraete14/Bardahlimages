# Rapport Fase 0: inventarisatie gast-mutatiepaden (AuditLog + added_by)

**Datum:** 13 augustus 2026
**Scope:** alleen-lezen inventarisatie; er is in deze fase **niets gewijzigd, gepubliceerd of verwijderd**.
**App:** Glistix (Base44, app-ID `69160faa53683c5a49a95c7d`)

## 1. Wat Guest en de platformbasis al bijhouden

- `Guest` (schema `base44/entities/Guest.jsonc`) heeft al: **`added_by_email`** (string) en `promotor_id`. **Ontbreekt: `added_by_user_id` en `added_via`.**
- Base44 houdt op elk record automatisch bij: **`created_by`** (e-mail van de maker), `created_date`, `updated_date`. Bewijs in de code: `EventDetails.saveGuestMutation` stript `created_by/created_date/updated_date` vóór een update, en de gastpanelen tonen `created_date` als fallback wanneer `added_by_email` leeg is.
- `added_by_email` wordt op **vrijwel elk create-pad al gevuld** (via `checkAdditionsAllowed` → `base44.auth.me()`), behalve bij de publieke registratie (daar is geen ingelogde gebruiker).
- ⚠️ **Kritieke bevinding:** `Guest.jsonc` heeft `permissions: { create: true, read: true, update: true, delete: true }` — elke ingelogde client mag rechtstreeks via de SDK schrijven. `QuotaConsumption.jsonc` heeft geen permissions-blok (platformdefault, in de praktijk ook client-schrijfbaar). Zolang dat zo blijft, is élke server-side log omzeilbaar door de SDK rechtstreeks aan te roepen. Server-side logging wordt pas fraudebestendig als client-write op `Guest` (en `QuotaConsumption`) wordt dichtgezet en alle writes door backend functions lopen. Dit raakt ~9 bestanden en hoort m.i. bij Fase 2 als expliciete beslissing.

## 2. Bestaande backend functions

`base44/functions/`: `sendInvitation`, `mollieWebhook`, `syncVenueRoleToEvents`, `mollieCreatePayment`, `redeemPromoCode`, `manageSubscription`, `invitationRSVP`. **Alleen `invitationRSVP` muteert Guest** (server-side create via `asServiceRole` bij een geaccepteerde RSVP). Alle overige gastmutaties lopen client-side via de SDK.

## 3. Inventaris per pad

| # | Pad | Bestand + functie | Backend of client-SDK? | Actor-registratie nu | Opmerkingen |
|---|---|---|---|---|---|
| 1a | Gast toevoegen (single, zijpaneel) | `src/components/guests/AddGuestPanel.jsx` → `handleSingleSave` → `onSave`-prop → `src/pages/EventDetails.jsx` → `saveGuestMutation` → `Guest.create` | **Client** | `added_by_email` (uit `auth.me()`), platform-`created_by` | Schrijft daarna client-side `CustomFieldValue.create` per veld |
| 1b | Gast toevoegen (single, GuestManager) | `src/pages/GuestManager.jsx` → `handleAdd` → `Guest.create` | **Client** | `added_by_email` (uit `checkAdditionsAllowed`), `created_by` | |
| 1c | Gast toevoegen (dialoog) | `src/components/guests/GuestDialog.jsx` → `onSave` → `EventDetails.saveGuestMutation` | **Client** | idem 1a | Zelfde mutatie als 1a |
| 2a | Bulk import (wizard) | `src/components/guests/ImportGuestsDialog.jsx` (~r. 435–475) → `Guest.bulkCreate` in batches van 25 | **Client** | `added_by_email` op elk record, `created_by` | Plus `CustomFieldValue.bulkCreate` |
| 2b | Bestand-import (zijpaneel) | `AddGuestPanel.jsx` → `handleFileImport` → `Guest.create` per record | **Client** | `added_by_email`, `created_by` | |
| 2c | Tekst-import (zijpaneel) | `AddGuestPanel.jsx` → `handleTextImport` → `Guest.create` per regel | **Client** | `added_by_email`, `created_by` | |
| 2d | Bestand/tekst-import (GuestManager) | `GuestManager.jsx` → `handleFileImport` / `handleTextImport` → `Guest.create` | **Client** | `added_by_email`, `created_by` | |
| 3a | Gast bewerken (volledig) | `EventDetails.jsx` → `saveGuestMutation` (update-tak) → `Guest.update` | **Client** | Alleen platform-`updated_date`; **geen actor** | |
| 3b | Gast bewerken (naam) | `GuestManager.jsx` → `GuestDetailPanel.handleSave` → `Guest.update` | **Client** | Geen actor | |
| 3c | Gast bewerken (+1-namen, tafel, overig) | `src/components/guests/GuestSidebar.jsx` → `saveAllNames` (custom_field_1), `tableUpdateMutation`, `updateMutation` (r. 126) → `Guest.update` | **Client** | Geen actor | |
| 3d | QR-code-backfill | `src/pages/CheckInStation.jsx` r. ~247 (effect) → `Guest.update({qr_code})` | **Client** | Geen actor | Automatisch, geen gebruikersactie — kandidaat `source: system` |
| 4a | Gast verwijderen (single) | `EventDetails.jsx` → `deleteGuestMutation` → `recordGuestDeletion` + `Guest.delete` | **Client** | Geen actor op de delete; `QuotaConsumption` bewaart `user_email` = added_by van de gast (niet de verwijderaar!), naam, aantallen | `recordGuestDeletion` (`src/lib/quotaConsumption.js`) is client-side en dus ook omzeilbaar |
| 4b | Gast verwijderen (GuestManager) | `GuestManager.jsx` → `GuestDetailPanel.handleDelete` → idem | **Client** | idem 4a | |
| 5a | Check-in (Check-in Station) | `CheckInStation.jsx` → `checkInMutation` → `Guest.update` (online-tak) | **Client** | Geen actor; alleen `check_in_time` op het record | Alle station-UI's (NameSearch, QR-scan, InlineGuestPanel, undo-toast) monden hierin uit |
| 5b | Check-in (EventDetails) | `EventDetails.jsx` → `checkInMutation` → `Guest.update` | **Client** | Geen actor | Parallel, bijna identiek aan 5a |
| 5c | Check-in (gastfiche) | `GuestSidebar.jsx` → `checkInMutation` (r. 126) | **Client** | Geen actor | |
| 5d | Check-in-toggle (GuestManager) | `GuestManager.jsx` → `handleCheckInToggle` → `Guest.update` | **Client** | Geen actor | |
| 6 | Offline wachtrij (station + EventDetails) | `src/hooks/useOfflineCheckIn.js` → `enqueue` (localStorage) → sync-lus in `CheckInStation.jsx` (~r. 219) en `EventDetails.jsx` (~r. 308) → `Guest.update` per item | **Client** | Geen actor. **Client-tijdstip zit niet expliciet in het queue-item** — alleen impliciet in `item.id` (`Date.now()_random`) en in `data.check_in_time` bij check-ins; bij offline uitchecken (`check_in_time: null`) ontbreekt elk client-tijdstip | Fase 2 moet `queuedAt` aan `enqueue()` toevoegen |
| 7 | Check-in ongedaan maken / corrigeren | Zelfde mutaties als 5a–5d met `checked_in:false` (o.a. undo-toast in `CheckInStation.onSuccess`, `handleCheckOut` in panelen) | **Client** | Geen actor | Geen apart pad; onderscheid alleen af te leiden uit de data |
| 8 | Bulk check-in/-uit | `EventDetails.jsx` → `handleBulkCheckIn` → parallelle `Guest.update`; bulk delete: `handleBulkDelete` → `recordGuestDeletion` + `Guest.delete` per id | **Client** | Geen actor | |
| 9 | TableManager | `src/pages/TableManager.jsx` → `checkInMutation`, `assignTableMutation`, `sidebarCheckInMutation` → `Guest.update`; plus drie inline `onAssignTable`-callbacks in `CheckInStation.jsx` (r. ~489/507/527) en `GuestSidebar.tableUpdateMutation` | **Client** | Geen actor | Tafeltoewijzing = `guest_updated`-kandidaat |
| 10 | Publieke registratie | `base44/functions/invitationRSVP/entry.ts` → `asServiceRole.entities.Guest.create` (bij accept) + `Invitation.update`; client `src/pages/RSVPPage.jsx` roept alleen de functie aan | **✅ Backend function** | Geen `added_by_*` (publiek, geen user); `created_by` = service role | **Enige pad dat vandaag zonder refactor server-side logbaar is** (`source: registration_link`, `added_via: registration_link`) |

Niet gevonden als mutatiepad (gecontroleerd): `GuestConfirmation.jsx` (alleen-lezen + QR), `TablePlan.jsx` (muteert `Table`/zones, geen `Guest`), `CheckInGuestSidebar.jsx` / `InlineGuestPanel.jsx` / `NameSearchCheckIn.jsx` / `QRScanCheckIn.jsx` (delegeren via `onCheckIn`-props naar 5a/9).

## 4. Conclusie

**Zonder refactor logbaar (nu):** alleen pad 10, `invitationRSVP` — de log-write kan direct in de bestaande backend function.

**Vereist eerst migratie client-SDK → backend function (alle overige paden 1–9).** Voorstel voor Fase 2, in oplopende volgorde van risico:

1. **`guestCheckIn`** (nieuwe backend function): single, bulk én offline-batch check-in/checkout. Payload per item: `guest_id`, gewenste velden, `occurred_at` (client-tijdstip; voor de offline queue eerst `queuedAt` toevoegen aan `enqueue()`). Dekt paden 5–8 — de fraudegevoeligste categorie — en schrijft `checked_in` / `checkin_undone` / `checkin_edited`-logs met `source: manual|offline_sync`.
2. **`guestWrite`** (nieuwe backend function): create (single + batch), update, delete. Dekt paden 1–4 en 9; vult `added_by_user_id/added_by_email/added_via` server-side in en schrijft `guest_added` / `guest_updated` (met diff) / `guest_deleted` (met naam-snapshot en tickets in `meta`).
3. **Client-write op `Guest` dichtzetten** (permissions in `Guest.jsonc`) zodra alle call-sites via de functions lopen — anders blijft de log omzeilbaar. Zelfde beslissing nodig voor `QuotaConsumption` (kan dan meteen mee de function in).

**Aandachtspunten voor Fase 1 (schema):**
- `added_by_email` bestaat al en is gevuld op de meeste creates; alleen `added_by_user_id` en `added_via` zijn nieuw op `Guest`.
- Base44 RLS-templates kunnen geen org-scoping via joins; voorstel: AuditLog-read in RLS beperken tot rol/e-mail (superadmin `didierverstraete14@gmail.com`) en org-scoping afdwingen in een `getAuditLog`-backend function.
- De offline queue mist vandaag een expliciet client-tijdstip (zie pad 6) — zonder die toevoeging kan `occurred_at` bij offline sync niet correct gevuld worden.

## 5. Status

Geen code of data gewijzigd. Wacht op goedkeuring om Fase 1 (AuditLog-entity + `added_by_user_id`/`added_via` op Guest + RLS) te starten.
