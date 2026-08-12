# Attendium – Design- & UX-analyse (referentiedocument voor Glistix)

> Aangeleverd door de gebruiker op 12 augustus 2026, op basis van een read-only verkenning van de live Attendium-app (web + mobiele mock). Dient als referentie voor design- en UX-beslissingen in Glistix. Zie `rapport-attendium-gap-analyse.md` voor de vertaling naar Glistix.

---

## 1. Wat Attendium is

Een gastenlijst-/check-in-platform voor eventorganisatoren, beschikbaar als web-app, iOS en Android. De kernbelofte op de homepage: gasten uitnodigen, beheren en inchecken op elk apparaat. Positionering is duidelijk één zin, één werkwoordreeks — geen feature-soep.

---

## 2. Visuele taal (design tokens)

Uitgelezen uit de live CSS:

| Token | Waarde |
|---|---|
| Font | `Inter, system-ui, sans-serif` |
| Basis fontgrootte | 16px, line-height 1.6 |
| Kleinere schalen | 14px (labels/meta), 18px (koppen) |
| Achtergrond app | `#000000` (header/nav), `#161616`, `#181818` (panelen), `#2C2C2C` (velden/borders) |
| Primaire tekst | `#E6E6E6` |
| Secundaire tekst | `#737373` / `#4a4a4a` (disabled) |
| Accent (primary) | `#AEEA00` (lime) |
| Waarschuwing | `#FFA500` |
| Fout/gevaar | `#FF0033` |
| Border radius | 4px / 6px |
| Transitie | `100ms ease-in-out` |
| Single-column breakpoint | 599px |
| Max contentbreedte | 1100px |

**Belangrijkste principe:** het is een bijna volledig monochroom, pikzwart canvas waarin **kleur uitsluitend betekenis draagt**. Lime = actie/positief/actief. Rood/roze = gastenlijst-labels en destructief. Alles wat geen betekenis heeft is grijs. Daardoor springt bij weinig licht (een club, een deur, een backstage) alleen het relevante element eruit. Dit is de belangrijkste les voor Glistix: kleur is schaars en functioneel, niet decoratief.

Header is 55px hoog, sidebar 200px breed. Er is een light mode (`Instellingen → Lichte modus`), maar dark is de default — logisch voor nachtevenementen.

---

## 3. Navigatiestructuur (informatiearchitectuur)

Drie niveaus, consequent doorgevoerd:

**Niveau 1 – Account (rechtsboven, dropdown onder je naam):** Events, Uitnodigingen, Passen, Analytics, Verkoop, Berichten, Instellingen (met subitems Jouw login, Account, Locaties, Gebruikers, Lichte modus, Taal), Nieuws, Support, Uitloggen.

**Niveau 2 – Event (linker sidebar binnen een event):** Guests, Analytics, Log, Instellingen, met bovenaan een `‹ Events`-terugknop.

**Niveau 3 – Event-instellingen (dezelfde sidebar wordt vervangen):** Details, Gastenlijsten, Aangepaste velden, Automatische berichten, Ticketing & QR, Rechten, Kopiëren, Verwijderen — met `‹ Guests` als terugknop.

Het slimme hieraan: er is nooit een tweede navigatiebalk bij. De linkerkolom **verwisselt van inhoud** naarmate je dieper gaat, en de breadcrumb is één enkele terugknop bovenaan diezelfde kolom. Je hebt dus altijd maximaal één navigatiekolom in beeld, en toch drie niveaus diepte.

**Aanbeveling voor Glistix:** kopieer dit "sidebar-swap met terugknop"-patroon in plaats van geneste accordeon-menu's of breadcrumbs.

---

## 4. Het kernscherm: gastenlijst

Dit is 90% van het gebruik en verdient de meeste aandacht.

**Layout: drie kolommen.**
- Links (200px): eventnavigatie.
- Midden (elastisch, max ~1100px): zoekbalk + gastenlijst + persistente actiebalk onderaan.
- Rechts (~460px): contextpaneel dat afwisselend "Gasten toevoegen" of "Gastdetail" toont.

**Header (altijd zichtbaar):** eventnaam vet, dan datum en locatie in grijs op dezelfde regel — hiërarchie via gewicht en kleur, niet via grootte. In het midden-rechts staat de **live teller `28 / 165`** (ingecheckt / totaal). Dat ene getal is het belangrijkste cijfer voor een deurman en het staat permanent in beeld op elk subscherm, ook op Analytics en in Instellingen. Zeer sterk detail.

**De lijst zelf:**
- Eén rij per gast, geen avatar, geen checkbox, geen chevron. Alleen: naam (vet, wit) + `+1`/`+6` groepsgrootte in grijs + `(gratis)` indien van toepassing + gekleurd gastenlijst-label.
- Rijhoogte ruim genoeg om met een duim te raken, maar dichtbij genoeg om ~22 gasten per scherm te tonen.
- **Reeds ingecheckte gasten worden gedimd** (grijs i.p.v. wit). Geen vinkje, geen doorhaling, geen badge — alleen contrastverlaging. Dat is het meest onderschatte detail van de hele app: je oog scant automatisch naar de heldere namen, dus naar wie er nog moet komen.
- Sorteren standaard alfabetisch op voornaam.

**Zoeken:** één veld bovenaan, instant filtering vanaf de eerste letters (typ "mar" → 8 resultaten in <200ms), met een `✕`-knop om te wissen. Geen zoekknop, geen debounce die voelbaar is, geen "geen resultaten"-pagina die het scherm overneemt.

**"Meer"-popover (rechtsboven de lijst):** twee tabs, *Filters* en *Exporteren*. Filters bevat: Ingecheckt (Alle/…), Gastenlijsten (checkboxes), Toegevoegd door (checkboxes met snelkoppelingen "Alle / Geen"), Sorteren op, en een "Resetten"-link. Export is letterlijk één regel: "Downloaden ⬇". Alle secundaire functionaliteit zit dus achter één woord, waardoor de hoofdbalk leeg blijft.

**Persistente actiebalk onderaan de lijst:** een pill met `+ Gasten toevoegen` en `Selecteren`. Zodra je selecteert verandert dezelfde pill in `1 geselecteerd · Annuleren · Acties ▾`. Dezelfde ruimte, andere modus — geen tweede toolbar die inschuift en de lijst laat springen.

---

## 5. Gasten toevoegen: drie invoermodi (grootste UX-troef)

Het rechterpaneel heeft drie tabs, en dit is waar Attendium zich echt onderscheidt:

**Enkel** — Voor- & achternaam, gastenlijst (dropdown), totaal aantal tickets, gratis tickets, `+ Nieuw aangepast veld`, `Toevoegen`. Minimaal aantal velden; alleen de naam is verplicht.

**Importeren** — één grote dashed dropzone: "Sleep een bestand hierheen of klik om te selecteren. (Excel, CSV & txt)". Geen kolommapping-wizard vooraf.

**Tekst** — een vrij tekstvak waarin je een hele lijst plakt, met een mini-syntax die eronder gedocumenteerd staat:
- één naam per regel → `John Smith +2`
- gratis tickets tussen haakjes → `Katy Perry +1 (2)`, `Hugo Smith (gratis)`
- notitie na dubbele punt → `Jon Snow +2 (3) : King of the north`
- e-mail na de naam → `John Smith john@attendium.com`

Dit is briljant omdat gastenlijsten in de praktijk via WhatsApp en e-mail binnenkomen als losse tekst. De instructies staan *permanent onder het invoerveld*, niet in een help-artikel of tooltip.

**Voor Glistix:** dit als eerste overnemen én uitbreiden — bijvoorbeeld met live-preview van de geparste regels (aantal herkende gasten, gemarkeerde foutregels) vóór het importeren, en met tafelnummer/`@handle` in dezelfde syntax.

---

## 6. Gastdetail & check-in

Klik op een rij → het rechterpaneel wisselt van "toevoegen" naar "detail" (geen modal, geen paginanavigatie, lijst blijft staan met de rij gehighlight).

Paneelkop: `✕` links, dan drie iconen rechts: verwijderen (prullenbak), delen, en `Opslaan`.

Inhoud, van boven naar onder: naam, gastenlijst, totaal tickets, gratis tickets, dan de **grote lime-omlijnde knop `Inchecken 0 / 1`** — veruit het grootste en enige gekleurde element in het paneel. Daaronder `Check-ins bewerken ⌄` (voor gedeeltelijke groeps-check-in), `Toegevoegd door` (read-only, toont welke promotor de gast plaatste), `+ Nieuw aangepast veld`, `Opslaan`, en onderaan een uitklapbare `Log`-sectie met de historiek van die gast.

De check-in-knop toont een breuk (`0 / 1`) in plaats van een aan/uit-toggle, precies omdat groepen gedeeltelijk kunnen aankomen. Dat is het juiste mentale model voor deurbeheer.

---

## 7. Analytics

**Per event**: een platte tabel per promotor (Toegevoegd door / Totaal aantal tickets / Gratis tickets / Ingecheckte tickets / Ingecheckte gratis tickets + Total-rij). Geen grafieken, wel exact de vijf getallen waarop promotors afgerekend worden, plus `(+3 deleted)` als subtiele annotatie bij verwijderde gasten. Sorteerbaar via chevrons in de kolomkoppen, met eigen zoekveld en export.

**Account-breed** (`/manager/analytics`): dashboard met periodeselector ("Last 90 days", "per day") en kaarten:
- Check-in rate (donut, lime vs. grijs, met absolute aantallen én percentages)
- Names by estimated gender (donut, blauw/roze)
- Guest list additions over time (lijngrafiek + legenda-ranking per persoon)
- Invitation signups over time (met "Nothing to show. Try changing the search options." als lege staat)
- Guest check-ins over time (per venue)
- Guest list additions by day of week (staafdiagram)
- Guest check-ins by hour of day (curve)
- Voetnoot: "Refreshed at 11 aug, 11:02. Refreshes every 30 minutes."

Elke kaart heeft zijn **eigen inline dropdowns** in de titelregel (bv. "Tickets ⌄", "Added by ⌄", "Venue ⌄") in plaats van globale filters bovenaan. Dat maakt elke kaart zelfstandig bruikbaar.

**Voor Glistix:** neem de "check-ins per uur van de dag"-curve zeker over — dat is de grafiek waarmee een organisator personeelsbezetting aan de deur plant. En de expliciete refresh-timestamp voorkomt twijfel over of cijfers live zijn.

---

## 8. Event-instellingen (elk scherm is één kolom van ~290px)

- **Details**: checkbox "Een sjabloon gebruiken", Locatie, Naam, Begint (datum + tijd), Eindigt (datum + tijd), en een actierij `Verwijderen · Kopiëren · Annuleren · Opslaan`. Destructieve acties links als tekstlink, primaire actie rechts als knop.
- **Gastenlijsten**: kaartjes met naam + **kleurenswatch** + `⋯`-menu + "Van sjabloon ⌄", en `+ Gastenlijst toevoegen`. Die kleur is exact de kleur die je terugziet als label achter elke gastnaam in de lijst. Eén instelling, overal consistent zichtbaar.
- **Aangepaste velden**: leeg canvas met `+ Nieuw aangepast veld`; toevoegen gebeurt via een klein popover-dialoog met een combobox (bestaande veldnamen suggereren hergebruik) en `Annuleren / Toevoegen`.
- **Automatische berichten**: tabel Triggers / Onderwerp / Ontvangers.
- **Ticketing & QR**: Betaaldienstaanbieder, en per gastenlijst een blok (omkaderd in de kleur van die lijst!) met Ticketprijs + EUR, Belasting, en QR-modus: *Geen QR-codes / 1 QR-code per naam / 1 QR-code per ticket*.
- **Rechten**: doorzoekbare lijst van alle teamleden met `@handle`; wie rechten heeft staat vetgedrukt in wit, de rest grijs. Bovenaan een aggregatierij "Iedereen (in totaal)".
- **Kopiëren**: Naam, Locatie, Begint, Eindigt, checkbox "Rechten meenemen", plus de expliciete waarschuwing dat de kopie géén gastgegevens bevat.

---

## 9. Overige schermen

**Events-overzicht**: tabs *Lopend & toekomstig / Verleden / Sjablonen*, zoekveld, "Meer ⌄" met locatiefilters (checkboxes + Alle/Geen), tabel Naam / Datum / Locatie, en `+ Nieuw event` onderaan de lijst in plaats van rechtsboven. Lege staat: "Geen lopende & toekomstige events gevonden." + de knop. Datums worden getoond als "za 15 nov 2025", en het jaar valt weg bij events in het lopende jaar ("za 7 feb") — kleine, prettige detaillering.

**Nieuw event**: sjabloon-checkbox, Locatie, Naam (met placeholder = locatienaam), Begint, Eindigt, "Naam van de eerste gastenlijst" (default "Gasten") met de geruststellende hint "Je kunt later meer gastenlijsten toevoegen." Zes velden en je bent live.

**Uitnodigingen**: tabs *Open / Persoonlijk / Sjablonen*. Open toont Beschrijving (met de eventnaam als grijze subregel eronder), Gemaakt door, Tickets als `5 / 150`. Persoonlijk voegt Gast, Gedeeld en Antwoord toe.

**Verkoop/Bestellingen**: zoekveld met de zeer expliciete placeholder "Zoek bestellingen op ID, e-mail of ticket-ID".

**404**: gecentreerd, twee woorden: "Not found." Geen illustratie, geen knoppen.

---

## 10. Feature-inventaris (uit de abonnementspagina)

Handig als checklist om Glistix-scope tegen af te zetten:

*Gastenbeheer*: snel zoeken, snel toevoegen, importeren (Excel/CSV/plakken), exporteren, kleurcodering, aangepaste velden, duplicaatdetectie, analyse, toegevoegd-door bijhouden, +1's & groepsgrootte, bevestigingslinks, activiteitenlogboek, eventoverstijgende passen, Apple & Google Wallet, pasaanpassing.

*Eventregistratie*: eventwebsite, e-mailbezorging, open links, persoonlijke links, meerdere per event, multi-event, gemengd betaald & gratis, antwoordlimieten, inline afbeeldingen, bijlagen, aangepast Antwoord-aan, bevestigingsmails, bronregistratie.

*Event check-in*: check-in, gedeeltelijke groeps-check-in, **offline modus**, QR-codes, realtime synchronisatie, meerdere apparaten, camera scannen, hardwarescanners.

*Ticketing*: ticketverkoop (€0,29/ticket), aangepaste tickettypes, meerdere Stripe-accounts, transparante kosten.

*Eventbeheer*: dupliceren, sjablonen, meerdere locaties. *Teambeheer*: rechten, teamaccounts, gasttoewijzingen, rolgebaseerde toegang, externe medewerkers. *Platform*: mobiele app, web-app, cross-platform, donkere modus, AVG, EU-hosting. *Berichten*: geautomatiseerde berichten, pushmeldingen. *Integraties*: API. *AI*: MCP-server.

Prijzen: Free €0 (50 gasten/event, 2 gebruikers) · Lite €49 (300 gasten) · Starter €199 (onbeperkt, 5 gebruikers) · Professional €388 (alle functies, onbeperkt gebruikers).

---

## 11. Mobiele app (uit de productmock op de homepage)

Zelfde ontwerp, andere chrome: hamburger links, eventnaam + datum + venue centraal, teller `1008 / 3353` rechtsboven. Daaronder de identieke zoekbalk met "More ⌄", dezelfde gastenlijst met gekleurde labels, een **lime FAB `+`** rechtsonder, en een bottom-tabbar met vier items: *Guests · Stats · Log · Settings*.

De informatiehiërarchie is dus letterlijk identiek tussen web en mobiel — alleen de navigatiecontainer verandert (sidebar → bottom tabs, knop → FAB). Dat is waarom mensen die de web-app kennen de app meteen begrijpen.

---

## 12. Wat Attendium sterk maakt — 10 principes om over te nemen

1. **Eén getal domineert.** De `ingecheckt / totaal`-teller staat permanent in de header op élk scherm.
2. **Kleur = betekenis.** Alles grijs, behalve één accentkleur voor acties, en gebruikergedefinieerde kleuren voor gastenlijsten.
3. **Status via contrast, niet via iconen.** Ingecheckte gasten dimmen weg i.p.v. een vinkje te krijgen.
4. **Master-detail zonder modals.** Het rechterpaneel wisselt van inhoud; de lijst verliest nooit zijn scrollpositie.
5. **Eén "Meer"-popover** vangt alle filters, sorteringen en export op, zodat de hoofdbalk leeg blijft.
6. **De actiebalk verandert van modus** in plaats van dat er een tweede balk verschijnt.
7. **Instructies staan naast het invoerveld**, niet in een helpcentrum.
8. **Formulieren zijn kort en hebben nooit meer dan één kolom.**
9. **Destructieve acties zijn tekstlinks links; de primaire actie is een knop rechts.**
10. **Feature-gating is eerlijk en in context**: waar een functie ontbreekt staat een oranje-omlijnd blokje met exact wat ontbreekt en een "Upgraden"-link — niet een generieke paywall.

---

## 13. Waar Attendium zwak is — kans om "iets uitgebreider" te zijn

- **Trage cold loads.** Volledige pagina-navigaties duurden 5–10 seconden met een leeg wit/zwart scherm. Er zijn geen skeleton-states op paginaniveau (wel een spinner in het Log-paneel). Glistix zou hier meteen kunnen winnen met optimistic rendering en skeletons.
- **Rijen zijn niet toetsenbord-navigeerbaar.** Klikken op een gastrij werkt, maar er is geen zichtbare focus-ring of pijltjes-navigatie. Voor een deurmedewerker met toetsenbord/scanner is `↑↓ + Enter = inchecken` een enorme snelheidswinst.
- **Geen bulk-acties behalve verwijderen.** "Acties ▾" bevat uitsluitend "Verwijderen". Bulk verplaatsen tussen gastenlijsten, bulk inchecken of bulk taggen ontbreekt.
- **Geen undo.** Er is een Log (betaald) maar geen "Ongedaan maken"-toast na een verwijdering of check-in. Een undo-toast van 8 seconden is de goedkoopste veiligheidsfeature die je kunt bouwen.
- **Analytics per event is puur tabulair.** Geen sparkline, geen conversie per promotor over tijd.
- **Aangepaste velden zijn typeloos** (alleen een naam). Veldtypes — getal, keuzelijst, datum, ja/nee, telefoonnummer — zouden validatie en betere filters mogelijk maken.
- **Zoeken is enkel op naam.** Zoeken over aangepaste velden, e-mail, tafelnummer of `+1`-namen ontbreekt.
- **Geen duplicaat-waarschuwing zichtbaar bij handmatig toevoegen** (duplicaatdetectie staat wel in de featurelijst, maar is niet zichtbaar in de invoerflow).
- **Lege staten zijn kaal.** "Not found." en "Geen bestellingen gevonden." zonder vervolgactie.
- **Rechten is een lange platte namenlijst** zonder rolgroepen of zoekbare rolfilters — schaalt slecht bij grote teams.

---

## 14. Concrete opdracht-set

Bouw een driekolomslayout (nav 200px / lijst elastisch max 1100px / contextpaneel 460px) op een `#000`-canvas met `#E6E6E6` tekst, Inter, één accentkleur, 4–6px radius en 100ms transities. Zet een permanente `ingecheckt / totaal`-teller in de header die op elk subscherm meegaat. Maak de gastenlijst virtueel gescrolld met gedimde rijen voor ingecheckte gasten en gekleurde gastenlijst-labels. Implementeer instant client-side zoeken vanaf de eerste letter, uitgebreid naar e-mail, tafelnummer en aangepaste velden. Bouw het detailpaneel als in-place vervanging van het toevoegpaneel, met een prominente `Inchecken n / m`-knop en een uitklapbare per-persoon-historiek. Neem de drie invoermodi over (enkel, bestand, tekst-plakken met mini-syntax) en voeg een live parse-preview toe. Voeg toetsenbordnavigatie toe (↑↓ selecteren, Enter inchecken, `/` focus zoeken, Esc paneel sluiten). Voeg een undo-toast toe na elke destructieve of check-in-actie. Breid bulk-acties uit naar verplaatsen, taggen en inchecken. Geef aangepaste velden een type met validatie. Bouw de analytics-kaarten met per-kaart inline filters, inclusief check-ins per uur van de dag. En bouw skeleton-states voor elke route zodat er nooit een leeg scherm van meer dan 300ms zichtbaar is.
