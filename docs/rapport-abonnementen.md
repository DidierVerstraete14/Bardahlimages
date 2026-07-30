# Rapport — Werken betaalde abonnementen correct?

*Stand: 30 juli 2026, gecontroleerd in de Base44-sandbox (appId `69160faa53683c5a49a95c7d`), na herverbinding van de connector.*

## Conclusie

**De kritieke betaalproblemen uit de audit van 5 juli zijn opgelost** — een betalende klant krijgt nu wat hij koopt. Er staan nog vier bekende, minder acute punten open (vooral: er is geen automatische verlenging of afloop).

## ✅ In orde bevonden

1. **Webhook zet de juiste limieten (was K5).** `mollieWebhook` gebruikt nu dezelfde limieten als `src/lib/planLimits.js` en de pricing-teksten: lite = 300 gasten/onbeperkt evenementen, starter/professional = onbeperkt. De tegenstrijdige derde tabel (`subscriptionLimits.jsx`) is verwijderd. Alle handhaving in de app (gasten-, event- en locatielimieten in EventDialog, AddGuestPanel, GuestDialog, checkGuest/Event/VenueLimit) leest uit die ene centrale tabel op basis van `subscription_plan` — betaalde klanten krijgen dus exact wat beloofd wordt.
2. **Event-unlock levert nu een event (was K1).** Vóór de betaling wordt het evenement als concept aangemaakt (`onBeforePurchase` in UnlockLimitDialog + Dashboard), zodat de webhook een geldig `eventId` heeft om te ontgrendelen (`is_unlocked_per_purchase` + `max_guests_override`). De webhook heeft bovendien een null-guard.
3. **Promocodes server-side (was K4).** `redeemPromoCode` draait als backend-functie met eigenaar-check, geldigheids-/verloop-/max_uses-controles en service-role updates. `Pricing.jsx` roept deze functie aan; de oude client-side activatie is weg.
4. **Betaalpagina claimt geen succes meer (was H5a).** `PaymentReturn` toont "Zodra Mollie de betaling bevestigt, wordt je aankoop automatisch geactiveerd" in plaats van een onterecht "Betaling geslaagd!"; de webhook activeert alleen bij status `paid`.

## ⚠️ Nog open (bekende beperkingen)

1. **Geen echte verlenging of afloop (A1).** Niets incasseert een volgende termijn (geen Mollie Subscriptions) en niets beëindigt een plan wanneer `subscription_ends_at`/`next_renewal_date` verstrijkt. Gevolg: een opgezegd abonnement blijft ná de einddatum gewoon doorlopen, promo-plannen verlopen nooit, en maandabonnees betalen één keer en houden het plan. Dit is het belangrijkste resterende punt.
2. **`mollieCreatePayment` mist een ownership-check (H5b).** Ingelogd zijn is vereist, maar er wordt niet gecontroleerd of de betaler bij `organizationId`/`eventId` hoort. Beperkt risico (de "aanvaller" betaalt écht geld), maar hoort dicht.
3. **Geen webhook-idempotentie.** Een opnieuw afgeleverde webhook herschrijft dezelfde waarden; bij een abonnement verschuift daarbij de verlengingsdatum naar "nu + periode". Klein risico; op te lossen door verwerkte `paymentId`'s op te slaan.
4. **Opzeggen en downgraden naar free zijn nog client-side (M10).** De browser doet rechtstreeks `Organization.update`. Zelfde patroon als de (inmiddels gerepareerde) promocode; hoort in een backend-functie met eigenaar-check.

## 📋 Datacheck

Eén organisatie met een betaald plan: **Place2Party** (professional, actief, gestart 9 april, `subscription_ends_at` 6 oktober 2026 — afkomstig van een promocode van 180 dagen).

- De opgeslagen limietvelden op dat record (500 gasten / 10 events / 3 locaties) stammen nog uit het oude promo-pad en passen niet bij professional. **Cosmetisch**: de handhaving leest de centrale tabel, niet deze velden. Aan te raden ze gelijk te trekken (999999/999/999), maar niet urgent.
- Let op punt ⚠️1: op 6 oktober verloopt dit plan feitelijk niet vanzelf.

## Aanbevolen vervolg (niet uitgevoerd)

1. Backend-functie `cancelSubscription`/`downgradeToFree` (dicht M10 + deel van A1).
2. Verloop-handhaving: bij het bepalen van het effectieve plan rekening houden met `subscription_ends_at < nu` → terugvallen op free (client-side helper in `planLimits.js` als eerste stap; structureel een dagelijkse job of Mollie Subscriptions).
3. Ownership-check in `mollieCreatePayment` en `paymentId`-dedup in de webhook.
