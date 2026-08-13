# Rapport Fase 1: AuditLog-entity + added_by op Guest

**Datum:** 13 augustus 2026
**App:** Glistix (Base44, app-ID `69160faa53683c5a49a95c7d`)
**Base44-checkpoint:** `Fase 1 AuditLog: entity + added_by_user_id/added_via op Guest (RLS client-write dicht)` (`6a7daac0c308988422aff56f`)
**Niet gepubliceerd.** Geen enkel datarecord aangemaakt, gewijzigd of verwijderd.

## Gewijzigde bestanden

1. `base44/entities/AuditLog.jsonc` — **nieuw**
2. `base44/entities/Guest.jsonc` — twee velden toegevoegd, verder byte-voor-byte ongewijzigd (volledig herlezen na de edit; ook `required` en `permissions` onaangeroerd)

## Wat er exact veranderde

### AuditLog (nieuw)

- Alle gevraagde velden: `organization_id`, `event_id`, `guest_id`, `guest_name_snapshot`, `actor_user_id`, `actor_email`, `action` (enum: `guest_added|guest_updated|guest_deleted|checked_in|checkin_undone|checkin_edited`), `source` (enum: `manual|import|registration_link|offline_sync|system`), `occurred_at` (date-time), `synced_at` (date-time, optioneel), `changes` (object), `meta` (object).
- `required`: `organization_id`, `event_id`, `guest_id`, `action`, `source`, `occurred_at`.
- **Client-writes dicht**: `permissions: { create: false, update: false, delete: false, read: true }`. Backend functions schrijven straks via `asServiceRole`, dat deze permissies omzeilt — precies de bedoeling.
- **Lezen beperkt via RLS**: `read` alleen voor `didierverstraete14@gmail.com` (superadmin) of platformrol `admin`.

### Guest (uitgebreid)

- `added_by_user_id` (string) — nieuw.
- `added_via` (enum `manual|import|registration_link|system`) — nieuw. **Afwijking van de spec:** de opdracht vroeg drie waarden; ik heb `system` toegevoegd omdat de Fase 3-backfill die waarde expliciet nodig heeft ("waar geen bron bestaat: added_via = system"). Zonder die waarde nu zou Fase 3 een tweede schema-wijziging vergen.
- Bestaand en onaangeroerd: `added_by_email`, `promotor_id`, alle overige velden en permissies.

## Gedocumenteerde RLS-beperking (zoals de opdracht vroeg)

Org-scoping ("org-admins zien alleen hun eigen organisatie") kan **niet** in Base44-RLS: dat vereist een join naar `OrganizationMember`/`OrganizationRole`, en RLS-templates kennen alleen `{{user.id}}`, `{{user.email}}`, `{{user.role}}`, `{{user.data.<veld>}}`. De `User`-entity bevat wel `current_organization_id`, maar dat is door de gebruiker zelf wisselbaar en zegt niets over admin-zijn binnen die org — dus bewust niet gebruikt. Gevolg: RLS-read staat nu strak (superadmin + platform-admins); **org-admins krijgen leestoegang via de `getAuditLog`-backendfunctie (Fase 4), die per aanvraag het org-lidmaatschap en de rol controleert via service role.**

## Verificatie & hoe jij het test

Geverifieerd door mij: `list_entity_schemas` bevestigt dat het platform beide schema's kent, inclusief de dichte permissies en de RLS-read-regel. (Kanttekening: de JSONC-commentaarregels die ik schreef zijn door het platform genormaliseerd/gestript — de rationale staat daarom in dit rapport.)

Zelf testen (met je testevent):

1. **Entity bestaat / schema klopt**: Base44 dashboard → Data → AuditLog: velden en enums controleren; Guest: `added_by_user_id` en `added_via` zichtbaar.
2. **Client-write wordt geweigerd**: log in de app in als een niet-admin gebruiker, open de browserconsole op een willekeurige Glistix-pagina en voer uit:
   ```js
   const { base44 } = await import('/src/api/base44Client.js');
   await base44.entities.AuditLog.create({ organization_id: 'x', event_id: 'x', guest_id: 'x', action: 'guest_added', source: 'manual', occurred_at: new Date().toISOString() });
   ```
   Verwacht: een permission-fout, géén record. (Zelfde test als superadmin hoort óók te falen — de create-permissie staat voor álle clients dicht; alleen service role mag schrijven.)
3. **Lezen beperkt**: als niet-admin in de console `await base44.entities.AuditLog.list()` → verwacht leeg/geweigerd; als `didierverstraete14@gmail.com` → toegestaan (nu nog 0 records).
4. **Bestaande flows ongebroken**: gast toevoegen/bewerken/inchecken in het testevent werkt zoals voorheen (de Guest-wijziging is puur additief).

## Openstaande risico's / afwijkingen

- `added_via` heeft een vierde enum-waarde `system` (gemotiveerde afwijking, zie boven).
- De platformrol `admin` in de RLS-read betreft Base44-app-admins, niet org-admins; als je ook dat te ruim vindt, kan de `$or`-tak met `role: admin` eruit en blijft alleen jouw superadmin-e-mail over — zeg het en ik versmal hem.
- Guest zelf blijft client-schrijfbaar (bewust: dichtzetten kan pas als alle call-sites in Fase 2 via backend functions lopen).

**Status: Fase 1 klaar. Ik wacht op je goedkeuring voor Fase 2 (server-side logging in de mutatiepaden, te beginnen met `invitationRSVP` en daarna de nieuwe `guestCheckIn`/`guestWrite`-functions).**
