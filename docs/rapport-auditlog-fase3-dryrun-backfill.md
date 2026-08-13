# Rapport Fase 3 — DRY-RUN backfill added_by (geen writes uitgevoerd)

**Datum:** 13 augustus 2026
**App:** Glistix (Base44, app-ID `69160faa53683c5a49a95c7d`)
**Status: alleen gelezen.** Geen enkel record gewijzigd. Uitvoering wacht op expliciete "ok, voer uit".

## Totaalbeeld

589 gasten, verdeeld over 33 events en 4 organisaties (organisatie "Test Events BE" heeft geen events/gasten). **Alle 589 missen `added_by_user_id` en `added_via`** (beide velden bestaan pas sinds Fase 1). 92 gasten hebben al een `added_by_email`; 497 niet. Base44's ingebouwde `created_by` is bij 588 van de 589 een echte gebruikers-e-mail en bij 1 gast de service-role (aangemaakt via de publieke RSVP-link).

Alle voorkomende e-mails matchen exact één `User`-record, dus `added_by_user_id` is overal afleidbaar waar een e-mail bestaat:

| E-mail | User-ID |
|---|---|
| didierverstraete14@gmail.com | `69160faa53683c5a49a95c7e` |
| didier_verstraete@hotmail.com | `699f23f2f895213fbddf42d0` |
| simon@place2party.be | `69c69c6da45e906922a8961a` |
| ken.deroost@icloud.com | `69dd0416cf491fc12ca6171a` |
| didier@framebelgium.be | `69ded944165579e7a47a9dea` |

## Per organisatie

| Organisatie | Gasten zonder `added_by_user_id` | Bruikbare bron | Geen afleidbare bron |
|---|---|---|---|
| DiVers Management | 582 | 582 (90 via bestaand `added_by_email`, 492 via `created_by`) | 0 |
| Place2Party | 6 | 5 (2 via `added_by_email`, 3 via `created_by`) | **1** (service-role) |
| Didier TML | 1 | 1 (via `created_by`) | 0 |
| Back to the Classics | 0 | — | — |
| **Totaal** | **589** | **588** | **1** |

De enige gast zonder afleidbare actor is `69d7b0c45be785d5855e573c` — "Simon Deketelaere", event "P2P - This ain't Texas" (Place2Party), aangemaakt 9 april 2026 met `created_by = service+…@no-reply.base44.com`. Dat is aantoonbaar de `invitationRSVP`-function (het enige server-side create-pad), dus feitelijk een registratielink-gast. **Conform je opdracht ("geen waarden verzinnen") stel ik voor: `added_via = 'system'`, actorvelden leeg.** Alternatief, als je de herkomst-kennis wél wil vastleggen: `added_via = 'registration_link'` — zeg het bij je akkoord.

## Voorgestelde waarden — 10 voorbeeldrecords

| Guest-ID | Naam | Event | Nu | Voorstel |
|---|---|---|---|---|
| `6a3298a18e2086c8fdc32107` | Isabelle Vanleeuw | TML Saturday W2 | added_by leeg; created_by=didier gmail | added_by_email=didierverstraete14@gmail.com, added_by_user_id=`69160faa…c7e` |
| `6a3298095d7a2b15c07f004c` | Oud-Turnhoutse Koerier… | TML Sunday W1 | idem | idem |
| `6a3296cc142143b1766dd389` | A & C Systems | TML Saturday W1 | idem | idem |
| `6a3295bc3830d0003afb82d6` | Kumps Willy | TML Friday W1 | idem | idem |
| `6a32669b623482127ce54dff` | VPK Packaging NV | TML Friday W2 | idem | idem |
| `69d7b0e1ecd21ca11214c588` | Simon Deketelaere | P2P This ain't Texas | added_by leeg; created_by=simon@place2party.be | added_by_email=simon@place2party.be, added_by_user_id=`69c69c…961a` |
| `69c69f85ad8d40f7ce15563a` | TESTSTTST | FYI (Didier TML) | added_by leeg; created_by=hotmail | added_by_email=didier_verstraete@hotmail.com, added_by_user_id=`699f23…42d0` |
| `69f65308ac70d6c065a590cb` | (Versuz-gast) | Versuz | added_by_email=ken.deroost@icloud.com, user_id leeg | alleen added_by_user_id=`69dd04…171a` erbij |
| `69f3e2ecbf9c2414e2c96a60` | (Stressed Out-gast) | Stressed Out | added_by_email=didier@framebelgium.be, user_id leeg | alleen added_by_user_id=`69ded9…9dea` erbij |
| `69d7b0c45be785d5855e573c` | Simon Deketelaere | P2P This ain't Texas | created_by=service-role, added_by leeg | **alleen `added_via='system'`**, actorvelden leeg |

## Voorgesteld uitvoeringsplan (na jouw akkoord)

Acht `update_entities`-batches met `{"$set": …}`, telkens gefilterd zodat al-gevulde velden nooit overschreven worden (`added_by_user_id` moet leeg zijn; bij groep B ook `added_by_email` leeg):

**A. user_id aanvullen waar `added_by_email` al bestaat (92):** per e-mail één call — gmail (81), framebelgium (6), ken.deroost (3), simon (2).
**B. beide velden vullen vanuit `created_by` (496):** per e-mail één call — gmail (492), simon (3), hotmail (1).
**C. de service-role-gast (1):** alleen `added_via: 'system'`.

Alle batches blijven onder de 500-recordlimiet van `update_entities`, dus geen `has_more`-lussen nodig. `added_via` blijft bij groep A en B **bewust leeg**: of een gast destijds handmatig dan wel via import binnenkwam is niet afleidbaar, en het veld krijgt pas betekenis voor nieuwe gasten (Fase 2 vult het voortaan altijd).

**AuditLog tijdens de backfill — mijn voorstel: géén logrecords schrijven.** Zelfs niet één samenvattend record per event (33 stuks): de log hoort gebruikersacties te weerspiegelen, en deze migratie is volledig gedocumenteerd in dit rapport, de commit en het Base44-checkpoint dat ik vóór uitvoering maak. Wil je toch één system-record per event, zeg het bij je akkoord.

## Verificatie na uitvoering (dan pas)

- Hertelling: 0 gasten zonder `added_by_user_id` behalve de service-gast; die heeft `added_via='system'`.
- Steekproef van de 10 bovenstaande records.
- Geen enkel ander veld gewijzigd (spot-check op `updated_date`-drift accepteren we; inhoudelijk blijft alles gelijk).

**Wacht op: "ok, voer uit" (met eventueel je keuze: service-gast `system` of `registration_link`, en wel/geen samenvattende logrecords).**
