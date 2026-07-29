# Hoe de Glistix-app nu werkt

*Stand: 29 juli 2026 — na de uitbreiding met rol-quota, gastenlijst-limieten en tijdslots. Alle wijzigingen staan in de Base44-sandbox (appId `69160faa53683c5a49a95c7d`); de build is groen. Live zetten gebeurt met de **Publish**-knop in de Base44-editor.*

---

## 1. Wat de app is

Glistix is een gastenlijst- en evenementenbeheerplatform, gebouwd op Base44 (React + Vite, data via Base44-entities). Organisaties beheren er hun evenementen, gastenlijsten, gasten, check-in, tafelplannen en team met rollen en machtigingen.

## 2. Accounts en organisaties

- Gebruikers loggen in via Base44-auth. Elke gebruiker heeft een `current_organization_id` die bepaalt in welke organisatie hij werkt.
- Een **Organization** heeft een eigenaar (`owner_email`) en een abonnement (`subscription_plan`) dat limieten bepaalt (aantal evenementen, gasten per evenement, enz. — zie `src/lib/planLimits.js`).
- Teamleden zijn **OrganizationMember**-records met een e-mailadres en optioneel een `role_id` die naar een **OrganizationRole** verwijst.
- **Platform-admins** (herkend via `isPlatformAdmin`) hebben overal onbeperkte toegang.

## 3. Rollen en machtigingen

### Rollen aanmaken en bewerken
Op de pagina **Rollenbeheer** (`RoleManagement`, alleen zichtbaar met de machtiging `can_manage_roles` of als eigenaar) kan een organisatie eigen rollen aanmaken. Per rol zijn er 26 aan/uit-machtigingen, gegroepeerd in: Organisatie, Gebruikers & Rollen, Evenementen, Gastenlijsten, Gasten, en Overig (locaties, tafelplan, Table Manager, analytics, promotors, tags, aangepaste velden).

### Nieuw: gastenlimiet per rol
Elke rol heeft nu ook een optionele **gastenlimiet**: *Max. personen* (`max_total_guests`) en *Waarvan gratis* (`max_free_guests`). Dit is het maximum dat een lid met deze rol **per evenement** mag toevoegen, geteld per persoon. Leeg = onbeperkt. Platform-admins en de organisatie-eigenaar vallen nooit onder een rol-quota.

### Nieuw: standaardrollen met één klik
De knop **"Standaardrollen toevoegen"** op Rollenbeheer maakt vijf voorgedefinieerde rollen aan (bestaande namen worden overgeslagen, dus dubbel klikken geeft geen duplicaten):

| Rol | Toegang |
|---|---|
| **Beheerder** | Alles |
| **Event Manager** | Evenementen, gastenlijsten, gasten, check-in, tafels, analytics, tags, velden |
| **Check-in medewerker** | Gasten bekijken, inchecken, Table Manager |
| **Promotor** | Gasten bekijken/toevoegen + gastenlijsten bekijken, met limiet 25 personen waarvan 10 gratis |
| **Staff** | Alleen-lezen op gasten en gastenlijsten |

### Rollen toewijzen
In **Organisatie-instellingen → Leden** staat naast elk lid een **inline rol-dropdown**: kies direct een rol (of "geen rol") en de wijziging wordt meteen opgeslagen. De eigenaar is beschermd en houdt altijd volledige toegang. Daarnaast blijft de uitgebreide bewerk-dialog beschikbaar.

## 4. Evenementen

Evenementen worden aangemaakt en beheerd binnen de organisatie (machtigingen `can_create_events` / `can_edit_events` / `can_delete_events`). Het abonnement bepaalt het maximum aantal gasten per evenement; een evenement kan daarnaast een individuele verhoging hebben (`max_guests_override`, te koop via de unlock-dialog).

## 5. Gastenlijsten

Per evenement kunnen meerdere gastenlijsten bestaan (bijv. "VIP", "Gastenlijst", "Pers"). Per lijst is instelbaar:

- **Max. capaciteit** (`max_capacity`) — maximaal aantal **personen** op de lijst;
- **Max. gratis** (`max_free_guests`) — maximaal aantal **gratis personen**;
- **Nieuw: tijdslot** (`add_from_date` / `add_until_date`) — het venster waarin aan de lijst toegevoegd mag worden.

De lijstkaarten tonen nu de bezetting in personen (niet in records), het aantal gratis en het tijdslot.

## 6. Gasten toevoegen — en hoe de limieten worden afgedwongen

Een gast-record heeft o.a. `party_size` (aantal personen) en `plus_ones_allowed` (waarvan gratis). Er zijn vier toevoegroutes, en **alle vier** passeren dezelfde controles:

1. **Snel toevoegen** (AddGuestPanel: één gast, bestand-import, tekst-import);
2. **Import-wizard** (ImportGuestsDialog: Excel/CSV met kolom-mapping, preview, dedup en batches);
3. **Gast-dialog** (uitgebreid formulier);
4. **Gasten toevoegen-paneel** op de GuestManager-pagina (enkel, bestand, tekst).

Alle importroutes registreren nu ook wie de gasten toevoegde (`added_by_email`), zodat de rol-quota ook bij imports correct meetelt.

**Basisregel: het aantal gratis personen kan nooit groter zijn dan het totaal aantal personen.** Bij handmatig invoeren (Snel toevoegen, Gast-dialog — ook bij bewerken) geeft de app een foutmelding; bij imports wordt het aantal gratis per rij automatisch begrensd op het aantal personen van die rij.

Volgorde van controles bij het opslaan:

1. **Abonnement/evenement-limiet** — totaal aantal gasten van het evenement versus planlimiet of `max_guests_override` (bestond al);
2. **Tijdslot van de lijst** — buiten het venster wordt toevoegen geblokkeerd met een duidelijke melding ("kan pas vanaf …" / "is gesloten sinds …");
3. **Lijstcapaciteit** — som van `party_size` op de lijst + nieuwe personen mag `max_capacity` niet overschrijden;
4. **Gratis op de lijst** — som van `plus_ones_allowed` + nieuwe gratis mag `max_free_guests` niet overschrijden;
5. **Rol-quota** — de personen/gratis die déze gebruiker al aan dit evenement toevoegde (via `added_by_email`) + de nieuwe toevoeging mag de `max_total_guests`/`max_free_guests` van zijn rol niet overschrijden.

Belangrijk:
- De **lijstregels (2–4) gelden voor iedereen**, ook voor de eigenaar en platform-admins.
- De **rol-quota (5) geldt niet** voor platform-admins en de organisatie-eigenaar.
- Bij imports wordt de **hele batch vooraf** getoetst: past de import niet, dan wordt niets geïmporteerd.
- De logica staat centraal in `src/lib/guestListLimits.js` (`checkGuestListLimit`, `checkRoleGuestQuota`), zodat alle routes identiek rekenen.

Daarnaast bestaan er nog per-gebruiker-per-evenement limieten (`EventUserPermission`, ingesteld via Evenement-machtigingen) — die werken zoals voorheen en staan los van de nieuwe rol-quota.

## 7. Check-in en verder

- **Check-in**: elke gast krijgt een uniek toegangstoken/QR-code; bij de deur worden personen per gast afgevinkt, met onderscheid gratis/betaald.
- **Tafelplan / Table Manager**: bij locaties met tafelplan kunnen gasten aan tafels worden gekoppeld.
- **Tags en aangepaste velden**: gasten kunnen getagd worden en per evenement kunnen extra invoervelden gedefinieerd worden.
- **Analytics**: statistieken per evenement voor wie `can_view_analytics` heeft.

## 8. Technische kanttekeningen

- Alle handhaving is **client-side** (zoals de rest van de app). Wie de API rechtstreeks aanroept, omzeilt de controles; harde afdwinging zou server-side regels in Base44 vergen.
- Schema-uitbreidingen zijn doorgevoerd op `GuestList` (`add_from_date`, `add_until_date`) en `OrganizationRole` (`max_total_guests`, `max_free_guests`).
- `npm run build` is groen na alle wijzigingen.
- Wijzigingen staan in de Base44-sandbox; **publiceren naar de live site gebeurt met de Publish-knop in de Base44-editor**.
