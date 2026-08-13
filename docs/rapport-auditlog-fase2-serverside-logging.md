# Rapport Fase 2: server-side logging in de mutatiepaden

**Datum:** 13 augustus 2026
**App:** Glistix (Base44, app-ID `69160faa53683c5a49a95c7d`)
**Base44-checkpoint:** `Fase 2 AuditLog: server-side logging — guestCheckIn/guestWrite-functions, alle check-in/add/edit/delete-paden omgelegd` (`6a7daefa11084d384aa6db58`)
**Niet gepubliceerd.** Volledige `vite build` slaagt. Elk gewijzigd bestand is vóór de wijziging volledig gelezen en na de wijziging herlezen.

## Gewijzigde bestanden

**Nieuwe backend functions**
1. `base44/functions/guestCheckIn/entry.ts` — nieuw
2. `base44/functions/guestWrite/entry.ts` — nieuw

**Bestaande backend function**
3. `base44/functions/invitationRSVP/entry.ts` — log-write + `added_via: 'registration_link'`

**Client (omgelegd naar de functions)**
4. `src/hooks/useOfflineCheckIn.js` — `queued_at` (client-tijdstip) op elk queue-item; her-enqueue behoudt het originele tijdstip
5. `src/pages/CheckInStation.jsx` — check-in-mutatie + offline-sync-lus → `guestCheckIn`
6. `src/pages/EventDetails.jsx` — check-in-mutatie, offline-sync-lus en bulk-check-in → `guestCheckIn`; opslaan/aanmaken, verwijderen en bulk-verwijderen → `guestWrite`; client-side `recordGuestDeletion` vervallen (nu server-side)
7. `src/components/guests/GuestSidebar.jsx` — check-in-mutatie → `guestCheckIn`
8. `src/pages/GuestManager.jsx` — single add, bestand-/tekst-import, naam-opslaan, verwijderen → `guestWrite`; check-in-toggle → `guestCheckIn`
9. `src/components/guests/AddGuestPanel.jsx` — bestand-/tekst-importlussen → één `guestWrite`-batch (source: import)
10. `src/components/guests/ImportGuestsDialog.jsx` — `Guest.bulkCreate` → `guestWrite`-batches van 25, met behoud van de custom-field-mapping per record
11. `src/pages/TableManager.jsx` — beide check-in-mutaties → `guestCheckIn`

## Hoe de functions werken

**`guestCheckIn`** — `{ event_id, source: manual|offline_sync, items: [{ guest_id, data, occurred_at?, queued_at? }] }`. Vereist een ingelogde gebruiker (401 zonder). Whitelist van check-in-velden (onder meer `checked_in`, `plus_ones_checked_in`, `check_in_time`, `status`, tafel/stoel/merch/maaltijd/custom-velden); valideert dat de gast bij het event hoort. Per item ná de geslaagde mutatie een AuditLog-record: actie afgeleid uit de tellerovergang (`checked_in` bij 0→n, `checkin_undone` bij n→0, anders `checkin_edited`), `occurred_at` = client-tijdstip (`occurred_at`/`queued_at`), bij offline_sync ook `synced_at` = nu, `meta` = `{ party_size, checked_in_before, checked_in_after }`. Log-fout → `console.error`, mutatie blijft geldig. Retourneert per-item `{ ok, error? }`; de client zet mislukte offline-items terug in de wachtrij mét hun originele `queued_at`.

**`guestWrite`** — drie acties. `create` (single of batch, source manual|import): zet server-side `added_by_user_id`/`added_by_email`/`added_via`, negeert die velden uit client-invoer (protected), logt per gast `guest_added` en retourneert de aangemaakte records (de client heeft de id's nodig voor custom-field-waarden en de deel-dialoog). `update`: protected velden gestript (id, created_*, added_by_* — added_by is dus niet wijzigbaar via update), logt `guest_updated` met een `changes`-diff `{ veld: { from, to } }` (alleen als er echt iets veranderde). `delete` (batch): schrijft eerst server-side het `QuotaConsumption`-record bij ooit-ingecheckt (monotoon verbruik — voorheen client-side en omzeilbaar), verwijdert dan, en logt `guest_deleted` met `guest_name_snapshot` en tickets in `meta` — het permanente spoor waar de quota-telling op kan bouwen.

**`invitationRSVP`** — de gast krijgt `added_via: 'registration_link'` en er wordt een `guest_added`-log geschreven (`source: registration_link`, `actor_email` = gast-e-mail, geen actor_user_id — publiek pad).

## Dekking per Fase 0-pad

| Pad | Status |
|---|---|
| 1 Single add (AddGuestPanel/GuestDialog/GuestManager) | ✅ gelogd via `guestWrite` |
| 2 Bulk/import (wizard, bestand, tekst, ×2 panelen) | ✅ gelogd via `guestWrite` (source: import) |
| 3 Bewerken (EventDetails/GuestDialog, GuestManager-naam) | ✅ gelogd via `guestWrite` met diff |
| 4 Verwijderen (single + bulk, beide panelen) | ✅ gelogd + server-side quota-record |
| 5 Check-in (Station, EventDetails, GuestSidebar, GuestManager) | ✅ gelogd via `guestCheckIn` |
| 6 Offline queue (Station + EventDetails) | ✅ `queued_at` → `occurred_at`, `synced_at`, source `offline_sync` |
| 7 Undo/correctie | ✅ zelfde pad, actie `checkin_undone`/`checkin_edited` |
| 8 Bulk check-in/-uit | ✅ één `guestCheckIn`-batch |
| 9 TableManager check-ins | ✅ gelogd |
| 10 Publieke registratie | ✅ gelogd in `invitationRSVP` |

## Nog niet gelogd (bewust, klein gehouden — voorstel per pad)

1. **Tafel-toewijzingen**: `TableManager.assignTableMutation`, drie `onAssignTable`-callbacks in CheckInStation, `GuestSidebar.tableUpdateMutation`. Voorstel: `guestWrite update` (logt dan automatisch een `table_id`-diff) — kleine follow-up, ±5 call-sites.
2. **+1-namen** (`GuestSidebar.saveAllNames`, schrijft `custom_field_1` bij blur): voorstel idem `guestWrite update`; bewust uitgesteld omdat het per toetsenbord-blur vuurt en de log anders vol ruis komt — eerst debounce toevoegen.
3. **QR-code-backfill** (CheckInStation-effect, zet `qr_code` op gasten zonder code): technisch, geen gebruikersactie. Voorstel: verplaatsen naar `guestCheckIn`/aparte system-write met `source: system`, of gewoon ongelogd laten.

## Openstaande risico's / kanttekeningen

- **Guest blijft client-schrijfbaar** (permissions open). De zeven overgebleven directe writes hierboven zijn daarvan de legitieme gebruikers. Pas wanneer ook die zijn omgelegd kan client-write op Guest dicht — dat is de sluitsteen en een aparte beslissing (zie Fase 0).
- De functions doen een **auth-check maar geen fijnmazige permissiecheck** (kan-inchecken/kan-toevoegen per gastenlijst wordt nog client-side afgedwongen, zoals voorheen). Server-side permissie-afdwinging is een logische verdieping ná het dichtzetten van client-write.
- **Latency**: elke check-in gaat nu via een function-roundtrip (met per-gast fetch + log-write). Bij normale aantallen onmerkbaar; bij zeer grote bulk-imports iets trager dan `bulkCreate`. Offline-sync is juist sneller (één batch i.p.v. één update per item).
- Een offline uitcheck-actie heeft nu wél een correct `occurred_at` (queued_at), wat vóór deze fase onmogelijk was.
- Base44's ingebouwde `created_by` op nieuwe gasten is voortaan de service role, niet meer de gebruiker — daarom vult `guestWrite` altijd `added_by_user_id`/`added_by_email` en is dat het veld om op te bouwen (ook relevant voor de eigen-gasten-regel, die al op `added_by_email` draait).

## Hoe jij het test (met het testevent)

1. **Check-in Station**: check een gast in → in Dashboard → Data → AuditLog verschijnt een `checked_in`-record met jouw e-mail, `source: manual` en `meta.checked_in_before/after`. Undo via de toast → `checkin_undone`. Gedeeltelijke groeps-check-in → `checkin_edited` met de juiste tellers.
2. **Offline**: zet het toestel offline (devtools → Network → Offline), check 2 gasten in, ga online → toast "2 offline check-ins gesynchroniseerd"; de logrecords hebben `source: offline_sync`, `occurred_at` = het moment van de offline tik, `synced_at` = het sync-moment.
3. **Toevoegen**: voeg een gast toe via het zijpaneel → `guest_added`-log en op de gast zelf `added_by_user_id`/`added_by_email`/`added_via: manual`. Importeer via tekst of Excel → idem met `added_via: import`.
4. **Bewerken**: wijzig een naam/e-mail → `guest_updated`-record met `changes: { full_name: { from, to } }`.
5. **Verwijderen**: verwijder een ingecheckte gast → `guest_deleted`-record mét naam-snapshot en tickets in meta, plus een `QuotaConsumption`-record; verwijder een nooit-ingecheckte gast → alleen het logrecord.
6. **Publieke registratie**: accepteer een uitnodiging via de RSVP-link → `guest_added` met `source: registration_link` en `added_via: registration_link` op de gast.
7. **Regressie**: alle bestaande flows (undo-toast, custom-fields-check-in, tafeltoewijzing bij check-in, bulk-acties, importwizard incl. nieuwe custom velden) horen ongewijzigd te werken.

**Status: Fase 2 klaar. Ik wacht op je test en goedkeuring voor Fase 3 (backfill added_by — begint verplicht met een dry-run).**
