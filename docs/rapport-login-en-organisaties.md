# Rapport — Login, organisatielidmaatschap en machtigingen

*Stand: 29 juli 2026, op basis van de actuele code in de Base44-sandbox (appId `69160faa53683c5a49a95c7d`).*

Dit rapport beschrijft (1) hoe inloggen werkt, (2) hoe mensen aan een organisatie worden toegevoegd, en (3) toetst het gewenste model: *"iedereen is admin binnen zijn eigen account (volgens abonnement); wie aan een andere organisatie wordt toegevoegd valt onder de machtigingen van die organisatie, maar blijft zijn eigen events zien."*

---

## 1. Hoe de login werkt

**Routes en flow**
- Niet-ingelogde bezoekers zien de **landingspagina**; de knoppen "Inloggen"/"Aanmelden" sturen naar de Base44-login (`base44.auth.redirectToLogin`) of naar de eigen `/login`-pagina.
- De **/login-pagina** ondersteunt twee methoden: e-mail + wachtwoord (`base44.auth.loginViaEmailPassword`) en **Google** (`base44.auth.loginWithProvider('google')`). Er zijn ook `/register`, `/forgot-password` en `/reset-password` pagina's.
- Alle app-routes staan achter een **ProtectedRoute**: wie niet is ingelogd wordt naar `/login` gestuurd. De `AuthContext` toont het loginscherm alleen bij een echte 401/403 (niet bij netwerkstoringen).

**Sessie/token**
- Het toegangstoken staat in `localStorage` (`base44_access_token`). `base44Client.js` leest het live token bij het opstarten en synchroniseert het opnieuw wanneer het tabblad focus krijgt — zo overleeft de sessie een pagina-refresh en token-vernieuwing.

**Eerste keer inloggen (onboarding)**
- Na de eerste login komt de gebruiker op **Onboarding**: gebruikersnaam kiezen (kleine letters/cijfers/underscores).
- Als de gebruiker al **lid is van een organisatie** (iemand voegde zijn e-mailadres eerder toe), slaat onboarding dat scherm over: de eerste gevonden organisatie wordt meteen de **actieve organisatie** (`current_organization_id`) en de gebruiker landt op het dashboard.
- Bij elke dashboard-load wordt `last_online` bijgewerkt.

## 2. Eigen account = eigen organisatie (admin volgens abonnement)

- Een nieuw account heeft **geen** organisatie. Het dashboard toont dan de banner **"Maak je eigen organisatie aan"** (gratis, geen creditcard).
- Bij het aanmaken gebeurt in één transactie (met rollback bij fouten):
  1. `Organization` met de gebruiker als `owner_email`, plan **free** (1 event, 50 gasten, 1 locatie);
  2. systeemrol **"Eigenaar"** met alle machtigingen;
  3. `OrganizationMember`-record dat de gebruiker aan die rol koppelt;
  4. de eigen organisatie wordt de actieve organisatie.
- **Binnen de eigen organisatie is de eigenaar altijd volledig admin**: `useOrgAccess.can()` geeft de eigenaar (en platform-admins) onvoorwaardelijk `true` voor elke machtiging. Het **abonnement** begrenst alleen de volumes: events per maand, gasten per event, locaties en het aantal leden (`getLimits`/`PLAN_LIMITS`).

✔ Dit deel van het gewenste model klopt dus: iedereen is admin binnen zijn eigen organisatie, binnen de grenzen van zijn abonnement.

## 3. Mensen aan een organisatie toevoegen

**Waar**: Organisatie-instellingen → tab **Leden** (zichtbaar met `can_view_members`; beheren vereist `can_manage_members`/`can_invite_members` of eigenaarschap).

**Hoe**
- **"Lid toevoegen"** opent een dialog: e-mailadres invullen (of een bestaande gebruiker zoeken — suggesties zijn beperkt tot eigen organisatieleden, geen cross-tenant bladeren), naam, en een **rol** kiezen uit de organisatierollen.
- Opslaan maakt direct een `OrganizationMember`-record aan (`user_email` + `role_id`). Er is **geen uitnodigings-/acceptatiestap**: het lidmaatschap bestaat meteen. Heeft de persoon nog geen account, dan "wacht" het lidmaatschap tot diegene zich met dat e-mailadres registreert (onboarding pakt het dan automatisch op).
- De **ledenlimiet van het abonnement** wordt afgedwongen bij het toevoegen (platform-admin uitgezonderd).
- In de ledenlijst kan de rol per lid ook **inline** gewisseld worden via een dropdown (direct opgeslagen). De **eigenaar** is beschermd: niet verwijderbaar en houdt altijd volledige toegang.
- Rollen zelf (machtigingen + gastenquota) worden beheerd op de pagina **Rollenbeheer**, inclusief de knop "Standaardrollen toevoegen".

## 4. Machtigingen wanneer je bij een andere organisatie hoort

De autorisatie loopt centraal via `useOrgAccess` en werkt per **actieve organisatie**:

1. **Platform-admin** (User.role = admin) → alles toegestaan, overal.
2. **Eigenaar van de actieve organisatie** → alles toegestaan binnen die organisatie.
3. **Anders**: de aan/uit-machtigingen van de `OrganizationRole` die aan jouw lidmaatschap hangt (`OrganizationMember.role_id`), plus de nieuwe gastenquota (`max_total_guests`/`max_free_guests`) van die rol.

✔ Ook dit klopt met het gewenste model: word je aan een andere organisatie toegevoegd, dan gelden **daar** de machtigingen die die organisatie jou via een rol geeft — je bent daar dus géén admin, ongeacht je eigen abonnement.

Daarnaast bestaan er fijnmazigere rechten die per **locatie** (`VenueRole`) of per **event** (`EventUserPermission`, met eigen limieten en tijdslot) kunnen worden toegekend, los van organisatierollen.

## 5. Eigen events blijven zien — hoe het nu werkt

Het dashboard bouwt de eventlijst uit drie bronnen:

1. **Events van de actieve organisatie** — een org-admin/eigenaar ziet ze allemaal; een gewoon lid alleen de events waar hij expliciete toegang heeft (event-rol, event-permissie of locatierol).
2. **Events van andere organisaties waar je lid bent** — maar **alleen** als je daar expliciete event-/locatierechten hebt (`hasAccess`-filter).
3. **Direct toegewezen events** (event-permissies/rollen), ongeacht organisatie.

## 6. Toets aan het gewenste model — drie afwijkingen

Het gewenste model klopt op de punten "admin in eigen organisatie" en "onder de machtigingen van de andere organisatie vallen". Op het derde punt — *"maar die blijft wel zijn eigen events zien"* — wijkt de praktijk af. Drie concrete gaten:

**Gap A — Er is geen organisatie-wisselaar.**
`current_organization_id` wordt alleen gezet bij onboarding (eerste lidmaatschap) en bij het aanmaken van een eigen organisatie. Wie lid is van meerdere organisaties kan nergens in de UI wisselen van actieve organisatie. De machtigingen van "de andere organisatie" worden dus pas echt actief als die organisatie toevallig je actieve organisatie is.

**Gap B — Wie eerst aan andermans organisatie wordt toegevoegd, kan geen eigen organisatie meer starten.**
De banner "Maak je eigen organisatie aan" verschijnt alleen als er **geen** actieve organisatie is. Word je eerst als lid toegevoegd bij iemand anders, dan ís er een actieve organisatie en verschijnt de banner nooit — er is dan geen UI-pad meer om je eigen organisatie (met eigen abonnement) aan te maken. Dit botst met "iedereen is admin binnen zijn eigen account".

**Gap C — Eigen events zijn niet gegarandeerd zichtbaar vanuit een andere organisatie.**
De `hasAccess`-check voor events van niet-actieve organisaties kijkt alleen naar expliciete event-rollen, event-permissies en locatierollen — **niet** naar eigenaarschap of lidmaatschap van die andere organisatie. Concreet: staat je actieve organisatie op "organisatie B" (waar je bent toegevoegd), dan zie je de events van je **eigen** organisatie A alleen als je daar toevallig expliciete event-/locatierechten hebt — eigenaarschap alleen is niet genoeg. Het gewenste "je blijft je eigen events zien" werkt dus nu alleen via die omweg.

**Kleinere observatie:** leden worden zonder uitnodiging/acceptatie toegevoegd — het e-mailadres is meteen lid. Dat is snel, maar de betrokkene merkt er niets van tot hij inlogt, en een typefout in het e-mailadres maakt stilletjes een "spooklid" aan.

## 7. Aanbevelingen (niet uitgevoerd)

1. **Organisatie-wisselaar** in de app-header: dropdown met alle organisaties waar je lid van bent; wisselen zet `current_organization_id` en herlaadt de context. Lost gap A op en maakt het model "daar gelden hun machtigingen, thuis ben ik admin" echt bruikbaar.
2. **Banner-conditie aanpassen**: toon "Maak je eigen organisatie aan" wanneer de gebruiker geen organisatie **bezit** (geen org met `owner_email` = eigen e-mail), in plaats van wanneer er geen actieve organisatie is. Lost gap B op.
3. **`hasAccess` uitbreiden**: events van organisaties waarvan je **eigenaar** bent altijd tonen (en desgewenst: waar je org-rol `can_view_guests`/eventrechten geeft). Lost gap C op.
4. Optioneel: een lichte **uitnodigingsflow** (e-mailnotificatie + acceptatie) om spookleden en typefouten te voorkomen.

---

*Alle genoemde logica is client-side; harde afdwinging zou server-side regels in Base44 vergen. Wijzigingen aan de app staan in de Base44-sandbox en gaan live via de Publish-knop in de Base44-editor.*
