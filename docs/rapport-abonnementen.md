# Rapport — Werken betaalde abonnementen correct?

*Stand: 30 juli 2026, gecontroleerd in de Base44-sandbox (appId `69160faa53683c5a49a95c7d`), na herverbinding van de connector.*

## Conclusie

**De kritieke betaalproblemen uit de audit van 5 juli zijn opgelost** — een betalende klant krijgt nu wat hij koopt. Er staan nog vier bekende, minder acute punten open (vooral: er is geen automatische verlenging of afloop).

## ✅ In orde bevonden

1. **Webhook zet de juiste limieten (was K5).** `mollieWebhook` gebruikt nu dezelfde limieten als `src/lib/planLimits.js` en de pricing-teksten: lite = 300 gasten/onbeperkt evenementen, starter/professional = onbeperkt. De tegenstrijdige derde tabel (`subscriptionLimits.jsx`) is verwijderd. Alle handhaving in de app (gasten-, event- en locatielimieten in EventDialog, AddGuestPanel, GuestDialog, checkGuest/Event/VenueLimit) leest uit die ene centrale tabel op basis van `subscription_plan` — betaalde klanten krijgen dus exact wat beloofd wordt.
2. **Event-unlock levert nu een event (was K1).** Vóór de betaling wordt het evenement als concept aangemaakt (`onBeforePurchase` in UnlockLimitDialog + Dashboard), zodat de webhook een geldig `eventId` heeft om te ontgrendelen (`is_unlocked_per_purchase` + `max_guests_override`). De webhook heeft bovendien een null-guard.
3. **Promocodes server-side (was K4).** `redeemPromoCode` draait als backend-functie met eigenaar-check, geldigheids-/verloop-/max_uses-controles en service-role updates. `Pricing.jsx` roept deze functie aan; de oude client-side activatie is weg.
4. **Betaalpagina claimt geen succes meer (was H5a).** `PaymentReturn` toont "Zodra Mollie de betaling bevestigt, wordt je aankoop automatisch geactiveerd" in plaats van een onterecht "Betaling geslaagd!"; de webhook activeert alleen bij status `paid`.

## ✅ Aangepast op 30 juli — verlengingsmodel + afloop-handhaving

Het gewenste model is geïmplementeerd: **automatische verlenging staat standaard aan** (een betaald plan loopt gewoon door) en met **één knop** zet de eigenaar de verlenging uit, waarna het plan aan het einde van de huidige periode terugvalt op **free**.

- **Schema**: `Organization.auto_renew` (boolean, standaard `true`); de Mollie-webhook zet hem bij elke betaling weer aan.
- **Effectief plan** (`getEffectivePlanKey`/`getEffectiveLimits` in `planLimits.js`): een plan met verstreken `subscription_ends_at` (verlenging uitgezet, of promo-periode voorbij) telt als **free** — en álle limiet-handhaving (gasten, evenementen, locaties, leden) gebruikt nu dit effectieve plan: EventDialog, AddGuestPanel, GuestDialog, Dashboard, OrganizationSettings, VenueManagement. 13 unit-tests op deze logica slagen.
- **Server-side beheer** (nieuwe backend-functie `manageSubscription`, met eigenaar-check — dicht meteen M10): `auto_renew_off` (einddatum = einde huidige periode), `auto_renew_on` (einddatum vervalt, weer actief), `downgrade_now` (direct naar free).
- **Pricing-UI**: knop "Automatische verlenging uitzetten" op het actieve plan; staat hij uit, dan toont de kaart "Loopt tot {datum}, daarna Free" met een knop om hem weer aan te zetten; is het plan verlopen, dan meldt de pagina dat de Free-limieten gelden. De directe free-downgrade loopt ook via de backend-functie, met bevestiging.
- **Promo's** verlopen nu echt: na de promo-einddatum gelden de free-limieten (bijv. Place2Party per 6 oktober, tenzij zij vóór die tijd betalen).

## ✅ Aangevuld op 30 juli — echte automatische incasso via Mollie Subscriptions

1. **Mandaat bij de eerste betaling.** `mollieCreatePayment` maakt (of hergebruikt) een Mollie-customer per organisatie (`mollie_customer_id`) en start de eerste abonnementsbetaling met `sequenceType: 'first'` — daarmee legt de klant een incassomandaat vast.
2. **Automatische incasso.** Na de eerste betaling maakt de webhook een Mollie-subscription aan (interval 1 maand of 12 maanden, startdatum = volgende verlengingsdatum, opgeslagen als `mollie_subscription_id`). Mollie incasseert vanaf dan zelf; elke incasso komt binnen als `subscription_renewal`-webhook die de toegang met één periode verlengt.
3. **Eén knop, echt effect.** "Automatische verlenging uitzetten" annuleert nu ook de Mollie-subscription (geen incasso's meer); "weer aanzetten" controleert het mandaat en zet een nieuwe subscription op vanaf de einddatum. Directe downgrade annuleert de incasso eveneens.
4. **Ownership-check toegevoegd (H5b dicht).** `mollieCreatePayment` controleert nu dat de betaler bij de organisatie hoort; abonnementen kan alleen de **eigenaar** (of platform-admin) afsluiten.
5. Bij een plan- of periodewissel wordt de oude Mollie-subscription automatisch geannuleerd en vervangen.

**Vereisten in het Mollie-dashboard** (buiten de code om):
- De **webhook-URL** van het website-profiel moet naar de `mollieWebhook`-functie wijzen (dat was al zo voor losse betalingen); optioneel kan de omgevingsvariabele `MOLLIE_WEBHOOK_URL` gezet worden zodat subscriptions hem expliciet meekrijgen.
- De **live API-key** moet recurring/incasso's toestaan (Mollie-account met Subscriptions geactiveerd).
- Testen kan met de test-API-key: eerste betaling → controleer in het Mollie-dashboard dat er een customer, mandaat én subscription zijn aangemaakt.

## ⚠️ Nog open (bekende beperkingen)

1. **Geen webhook-idempotentie.** Een opnieuw afgeleverde webhook herschrijft dezelfde waarden; bij een verlenging verschuift daarbij de verlengingsdatum naar "nu + periode". Klein risico; op te lossen door verwerkte `paymentId`'s op te slaan.
2. **Mislukte incasso's worden niet gedetecteerd.** Als Mollie een incasso definitief niet kan innen (subscription wordt dan door Mollie gestopt), merkt de app dat niet vanzelf; het plan loopt door tot er handmatig wordt ingegrepen. Oplosbaar met een periodieke controle van de subscription-status.
3. **Bestaande betaalde organisaties hebben nog geen mandaat.** Zij vallen pas onder automatische incasso nadat ze (bij de volgende betaling) opnieuw via de checkout betalen.

## 📋 Datacheck

Eén organisatie met een betaald plan: **Place2Party** (professional, actief, gestart 9 april, `subscription_ends_at` 6 oktober 2026 — afkomstig van een promocode van 180 dagen).

- De opgeslagen limietvelden op dat record (500 gasten / 10 events / 3 locaties) stammen nog uit het oude promo-pad en passen niet bij professional. **Cosmetisch**: de handhaving leest de centrale tabel, niet deze velden. Aan te raden ze gelijk te trekken (999999/999/999), maar niet urgent.
- Let op punt ⚠️1: op 6 oktober verloopt dit plan feitelijk niet vanzelf.

## Aanbevolen vervolg (niet uitgevoerd)

1. Backend-functie `cancelSubscription`/`downgradeToFree` (dicht M10 + deel van A1).
2. Verloop-handhaving: bij het bepalen van het effectieve plan rekening houden met `subscription_ends_at < nu` → terugvallen op free (client-side helper in `planLimits.js` als eerste stap; structureel een dagelijkse job of Mollie Subscriptions).
3. Ownership-check in `mollieCreatePayment` en `paymentId`-dedup in de webhook.
