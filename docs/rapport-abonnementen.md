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

## ⚠️ Nog open (bekende beperkingen)

1. **Geen automatische incasso.** "Verlenging aan" betekent: het plan loopt door zonder onderbreking; er wordt niet automatisch een nieuwe betaling geïncasseerd (dat vergt Mollie Subscriptions met mandaten). Facturatie van verlengingen blijft dus handmatig/extern.
2. **`mollieCreatePayment` mist een ownership-check (H5b).** Ingelogd zijn is vereist, maar er wordt niet gecontroleerd of de betaler bij `organizationId`/`eventId` hoort. Beperkt risico (de "aanvaller" betaalt écht geld), maar hoort dicht.
3. **Geen webhook-idempotentie.** Een opnieuw afgeleverde webhook herschrijft dezelfde waarden; bij een abonnement verschuift daarbij de verlengingsdatum naar "nu + periode". Klein risico; op te lossen door verwerkte `paymentId`'s op te slaan.

## 📋 Datacheck

Eén organisatie met een betaald plan: **Place2Party** (professional, actief, gestart 9 april, `subscription_ends_at` 6 oktober 2026 — afkomstig van een promocode van 180 dagen).

- De opgeslagen limietvelden op dat record (500 gasten / 10 events / 3 locaties) stammen nog uit het oude promo-pad en passen niet bij professional. **Cosmetisch**: de handhaving leest de centrale tabel, niet deze velden. Aan te raden ze gelijk te trekken (999999/999/999), maar niet urgent.
- Let op punt ⚠️1: op 6 oktober verloopt dit plan feitelijk niet vanzelf.

## Aanbevolen vervolg (niet uitgevoerd)

1. Backend-functie `cancelSubscription`/`downgradeToFree` (dicht M10 + deel van A1).
2. Verloop-handhaving: bij het bepalen van het effectieve plan rekening houden met `subscription_ends_at < nu` → terugvallen op free (client-side helper in `planLimits.js` als eerste stap; structureel een dagelijkse job of Mollie Subscriptions).
3. Ownership-check in `mollieCreatePayment` en `paymentId`-dedup in de webhook.
