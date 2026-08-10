# Rechtenmodel Glistix — gap-analyse en verbeterplan

Vergelijking van de huidige Glistix-implementatie met het onderzochte Attendium-rechtenmodel
(account → locatie → sjabloon → event → gastenlijst, additief capability-model met limieten).
Alle bevindingen hieronder zijn geverifieerd in de broncode van de Base44-app.

## 1. Wat er nu staat

Rechten leven verspreid over **vijf dragers**, elk met eigen veldnamen en semantiek:

| Drager | Niveau | Bijzonderheden |
|---|---|---|
| `OrganizationMember` → `OrganizationRole` | organisatie | ~28 booleans, plus rol-quota `max_total_guests`/`max_free_guests` (per evenement, gehandhaafd) |
| `VenueRole` | locatie | per gebruiker × locatie, 10 booleans |
| `UserRole` → `Role` | event (rol) | benoemde rollen per event |
| `EventUserPermission` | event (direct) | per categorie óf per gastenlijst; heeft limietvelden |
| `Promotor.max_guests_per_event` / `CheckInProfile.allowed_guest_list_ids` | los | aparte eilandjes |

De resolutie zit in `src/hooks/useEffectivePermissions.js`: app-admin en organisatie-eigenaar
kortsluiten naar alles; daarna geldt **first-match-override**: EventUserPermission → event-Role →
VenueRole → OrganizationRole → fallback. Limieten voor gastenlijsten en rol-quota zitten centraal
in `src/lib/guestListLimits.js` (`checkAdditionsAllowed`), aangeroepen door AddGuestPanel,
GuestManager en ImportGuestsDialog.

## 2. Gevonden gaten (belangrijkste eerst)

1. **`EventUserPermission`-limieten worden nooit gehandhaafd.** De velden `max_total_guests`,
   `max_free_guests`, `can_add_from_date` en `can_add_until_date` zijn instelbaar in
   EventPermissions.jsx, maar `checkAdditionsAllowed` leest ze niet. Een gastheer met "max 10
   tickets tot vrijdag 18u" kan nu onbeperkt en altijd toevoegen.
2. **Per-lijst-rechten lekken naar het hele event.** `eventPermissionsToFlags` neemt bij ontbreken
   van een "all"-record de *meest permissieve* vlaggen over álle records. Wie `can_add` op één
   gastenlijst heeft, krijgt daarmee `can_add` op het hele event; AddGuestPanel toont bovendien
   alle lijsten in de dropdown. Alleen *bekijken* wordt per lijst beperkt (`allowedGuestListIds`).
3. **Override in plaats van unie.** Attendium is additief (grants stapelen langs de keten). Bij ons
   *vervangt* een lager record het hogere: wie op locatieniveau alles mag en daarna één klein
   event-recht krijgt, verliest op dat event zijn locatierechten. Dat is verrassend en ondocumenteerd.
4. **Standaard is "bekijken" in plaats van "niets".** `DEFAULT_PERMISSIONS` en het schema-default
   van `can_view_guests` staan op `true`. Iemand zonder enig rechtenrecord die de event-URL kent,
   ziet de volledige gastenlijst. Attendium hanteert default-deny (alleen de master user heeft
   impliciet alles).
5. **Geen "eigen gasten"-regel.** Attendium: `add_guests ⇒ eigen gasten zien/wijzigen/verwijderen`.
   Bij ons bestaat het patroon half (EventDetails toont eigen gasten naast toegestane lijsten),
   maar bewerken/verwijderen van eigen gasten vereist alsnog `can_edit_guests`/`can_delete_guests`,
   en een promotor zonder `can_view` ziet zijn eigen lijst niet in het Check-in Station.
6. **Quota zijn omzeilbaar via verwijderen.** Alle tellers zijn `som over actieve rijen`. Wie zijn
   quotum vol heeft, kan ingecheckte gasten verwijderen en opnieuw toevoegen. Attendium telt
   ingecheckt-en-verwijderd onherroepelijk mee.
7. **Geen objectquotum ("Iedereen in totaal").** GuestList kent `max_capacity`/`max_free_guests`
   per lijst, maar er is geen event-breed totaalplafond onafhankelijk van de som van
   individuele limieten. `Event.capacity` bestaat als veld maar wordt niet als harde poort gebruikt.
8. **Sjablonen dragen geen rechten.** `EventTemplate` heeft geen rechtenkoppeling; rechten voor
   terugkerende events moeten per event opnieuw. Attendium gebruikt sjabloonrechten juist als
   dé beheerroute voor reeksen.
9. **Geen extern-gebruiker-concept.** Grants verwijzen naar e-mail, dus cross-organisatie werkt
   technisch al half (Dashboard toont events met een EventUserPermission), maar er is geen
   markering "extern", geen aparte UI-weergave en geen opzegflow.
10. **Feature-gates ongehandhaafd.** `can_import_guests`/`can_export_guests` bestaan op
    OrganizationRole maar de import/export-knoppen checken ze niet.

## 3. Verbeterplan in drie fasen

### Fase 1 — handhaving repareren (DOORGEVOERD)

- **EventUserPermission-limieten afgedwongen**: `checkEventUserPermLimits` in
  `src/lib/guestListLimits.js` toetst nu bij elke toevoeging (enkel, import, tekst — via het
  centrale `checkAdditionsAllowed`) het persoonlijke tijdslot en max totaal/gratis personen,
  per gastenlijst-record én voor "hele event"-records; platform-admin en organisatie-eigenaar
  vallen erbuiten.
- **Per-lijst-scoping van toevoegen/inchecken**: de hook geeft naast `allowedGuestListIds`
  (bekijken) nu ook `addableGuestListIds` en `checkinGuestListIds` terug. AddGuestPanel toont
  alleen toegestane lijsten in de dropdowns, `checkAdditionsAllowed` weigert toevoegingen
  buiten de scope (dekt ook GuestManager en ImportGuestsDialog), en het Check-in Station plus
  de check-in-knoppen in de tabel respecteren de check-in-scope. Lijst-gebonden `can_admin`
  geeft niet langer event-brede admin.
- **Eigen-gasten-regel**: wie mag toevoegen ziet, bewerkt en verwijdert zijn eigen gasten
  (op `added_by_email`), ook zonder view/edit/delete-recht — in de tabel én de zijbalk.
- **Default-deny**: `DEFAULT_PERMISSIONS` staat op alles-uit, het schema-default van
  `EventUserPermission.can_view` is `false`, nieuw toegevoegde gebruikers in
  Evenement-machtigingen starten zonder rechten, en EventDetails toont een
  "Geen toegang"-melding voor wie geen enkel recht heeft.
- **Import/export-gates**: de import- en exportknoppen in EventDetails controleren nu
  `can_import_guests`/`can_export_guests` van de organisatierol.

Restpunt fase 1: de lijst-dropdowns in GuestManager zijn nog niet visueel gefilterd op de
add-scope (de centrale controle blokkeert wel), en de vlag-berekening bleef verder
first-match-override — de unie komt in fase 2.

### Fase 2 — unie-semantiek (DOORGEVOERD)

`useEffectivePermissions` berekent de effectieve rechten nu als **unie van alle niveaus**
(organisatierol ∪ locatierol ∪ event-rol ∪ directe event-machtigingen) in plaats van
first-match-override: een klein event-recht kan iemands locatie- of organisatierechten dus niet
meer per ongeluk vervangen. De kortsluitingen voor app-admin en organisatie-eigenaar blijven.
De hook geeft daarnaast `flagSources` terug: per vlag de lijst van niveaus die hem toekennen
(basis voor "Van locatie"-herkomstbadges in de UI). De gastenlijst-scopes gelden alleen nog
wanneer géén breder niveau hetzelfde recht al event-breed geeft — dat is ook doorgetrokken in
`checkAdditionsAllowed`, dat event-breed toevoegrecht via org-, locatie- of event-rol laat
voorgaan op de lijst-scope van directe event-records. Limieten stapelen als *meest beperkende
geldige waarde*: lijst-limieten, persoonlijke event-limieten en rol-quota worden allemaal
toegepast; leeg = geen limiet op dat niveau.

### Fase 3 — schema-unificatie

- Eén entiteit `PermissionGrant`: `user_email`, `scope_type`
  (`organization|venue|event_template|event`), `scope_id`, nullable `guest_list_id`, de vlaggen
  (`admin`, `view_guests`, `add_guests`, `edit_guests`, `delete_guests`, `check_in_guests`),
  de limieten (`max_total`, `max_free`, `add_from_at`, `add_until_at`) en `is_external`.
- Eén entiteit `ScopeQuota` voor "Iedereen in totaal": `scope_type`, `scope_id`, nullable
  `guest_list_id`, `max_total`, `max_free`.
- Monotone verbruiksteller: entiteit `QuotaUsage` (`event_id`, `user_email`, nullable
  `guest_list_id`, `consumed_total`, `consumed_free`) die bij toevoegen ophoogt en bij
  verwijderen alleen verlaagt wanneer de gast **nooit ingecheckt** was.
- Rechten op `EventTemplate` die overerven naar events die het sjabloon gebruiken.
- Migratie: VenueRole/EventUserPermission 1-op-1 omzetten naar grants; Role/UserRole en
  OrganizationRole blijven als benoemde-rol-laag die bij toekenning naar grants "uitvouwt"
  of als aparte bron in de unie blijft meedraaien.

## 4. Kanttekening

Alle handhaving is en blijft client-side (Base44-architectuur); wie de API rechtstreeks aanroept,
omzeilt elke controle. Harde afdwinging vergt server-side regels of backend-functies — dat geldt
voor het hele bestaande model en verandert niet door dit plan.
