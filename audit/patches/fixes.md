# Glistix — Patchset bij audit (exacte oud→nieuw blokken)

Toepasbaar op de Base44-app `Glistix` (appId `69160faa53683c5a49a95c7d`).
Elke patch is een exacte string-vervanging (`OUD` moet uniek voorkomen in het bestand).

---

## P3 — `base44/functions/invitationRSVP/entry.ts` (K3 + H6: token-lookup + expiry)

**OUD**
```ts
  // Find invitation by token (service role - no user auth needed)
  const allInvitations = await base44.asServiceRole.entities.Invitation.list('-created_date', 500);
  const invitation = allInvitations.find(i => i.token === token);

  if (!invitation) return Response.json({ error: 'Invitation not found' }, { status: 404 });
```
**NIEUW**
```ts
  // Find invitation by token (service role - no user auth needed)
  const matches = await base44.asServiceRole.entities.Invitation.filter({ token });
  const invitation = matches[0];

  if (!invitation) return Response.json({ error: 'Invitation not found' }, { status: 404 });

  // Verlopen uitnodigingen zijn niet meer bruikbaar
  if (invitation.expires_at && new Date(invitation.expires_at) < new Date()) {
    if (invitation.status !== 'expired') {
      await base44.asServiceRole.entities.Invitation.update(invitation.id, { status: 'expired' });
    }
    return Response.json({ error: 'Invitation expired' }, { status: 410 });
  }
```

**OUD**
```ts
    await base44.asServiceRole.entities.Invitation.update(invitation.id, {
      status: newStatus,
      responded_at: new Date().toISOString(),
      rsvp_count: rsvp_count || 1
    });
```
**NIEUW**
```ts
    await base44.asServiceRole.entities.Invitation.update(invitation.id, {
      status: newStatus,
      responded_at: new Date().toISOString(),
      rsvp_count: accepted ? (rsvp_count || 1) : 0
    });
```

---

## P5a — `base44/functions/mollieWebhook/entry.ts` (K5: limieten conform pricing; K1: null-guard)

**OUD**
```ts
const PLAN_LIMITS = {
  lite:         { max_guests_per_event: 300, max_events_per_month: 5,  max_venues: 2 },
  starter:      { max_guests_per_event: null, max_events_per_month: 20, max_venues: 999 },
  professional: { max_guests_per_event: null, max_events_per_month: null, max_venues: 999 },
};
```
**NIEUW**
```ts
// Eén bron van waarheid met src/lib/planLimits.js + pricing-teksten:
// alle betaalde plannen hebben onbeperkt evenementen; lite = 300 gasten/event.
const PLAN_LIMITS = {
  lite:         { max_guests_per_event: 300,  max_events_per_month: null, max_venues: 2 },
  starter:      { max_guests_per_event: null, max_events_per_month: null, max_venues: 999 },
  professional: { max_guests_per_event: null, max_events_per_month: null, max_venues: 999 },
};
```

**OUD**
```ts
  } else if (meta.type === 'event_unlock') {
    const { eventPlan, eventId } = meta;
    const limits = PLAN_LIMITS[eventPlan];
```
**NIEUW**
```ts
  } else if (meta.type === 'event_unlock') {
    const { eventPlan, eventId } = meta;
    if (!eventId) return new Response('ok');
    const limits = PLAN_LIMITS[eventPlan];
```

---

## P13 — `base44/functions/syncVenueRoleToEvents/entry.ts` (H7: veld buiten schema)

**OUD**
```ts
        await base44.asServiceRole.entities.EventUserPermission.update(existing[0].id, {
          ...permData,
          event_id: event.id,
          _venue_synced: true,
        });
```
**NIEUW**
```ts
        await base44.asServiceRole.entities.EventUserPermission.update(existing[0].id, {
          ...permData,
          event_id: event.id,
        });
```

**OUD**
```ts
        await base44.asServiceRole.entities.EventUserPermission.create({
          ...permData,
          event_id: event.id,
          _venue_synced: true,
        });
```
**NIEUW**
```ts
        await base44.asServiceRole.entities.EventUserPermission.create({
          ...permData,
          event_id: event.id,
        });
```

---

## P4 — NIEUW BESTAND `base44/functions/redeemPromoCode/entry.ts` (K4: promocode server-side)

```ts
import { createClientFromRequest } from 'npm:@base44/sdk@0.8.25';

const PLAN_LIMITS = {
  lite:         { max_guests_per_event: 300,  max_events_per_month: null, max_venues: 2 },
  starter:      { max_guests_per_event: null, max_events_per_month: null, max_venues: 999 },
  professional: { max_guests_per_event: null, max_events_per_month: null, max_venues: 999 },
};

Deno.serve(async (req) => {
  const base44 = createClientFromRequest(req);
  const user = await base44.auth.me();
  if (!user) return Response.json({ error: 'Unauthorized' }, { status: 401 });

  const { code, organizationId } = await req.json();
  if (!code || !organizationId) return Response.json({ error: 'Missing code or organizationId' }, { status: 400 });

  const orgs = await base44.asServiceRole.entities.Organization.filter({ id: organizationId });
  const org = orgs[0];
  if (!org) return Response.json({ error: 'Organization not found' }, { status: 404 });
  if (org.owner_email !== user.email && user.role !== 'admin') {
    return Response.json({ error: 'Alleen de eigenaar kan een promocode inwisselen' }, { status: 403 });
  }

  const codes = await base44.asServiceRole.entities.PromoCode.filter({ code: String(code).trim().toUpperCase() });
  const promo = codes[0];
  const now = new Date();
  if (!promo || !promo.is_active) return Response.json({ error: 'Ongeldige of inactieve promocode' }, { status: 400 });
  if (promo.expires_at && new Date(promo.expires_at) < now) return Response.json({ error: 'Deze promocode is verlopen' }, { status: 400 });
  if ((promo.used_count || 0) >= (promo.max_uses || 1)) return Response.json({ error: 'Deze promocode is al volledig gebruikt' }, { status: 400 });

  const endsAt = new Date(now.getTime() + promo.duration_days * 24 * 60 * 60 * 1000);
  const limits = PLAN_LIMITS[promo.plan] || {};
  await base44.asServiceRole.entities.PromoCode.update(promo.id, { used_count: (promo.used_count || 0) + 1 });
  await base44.asServiceRole.entities.Organization.update(organizationId, {
    subscription_plan: promo.plan,
    subscription_status: 'active',
    subscription_started_at: now.toISOString(),
    subscription_ends_at: endsAt.toISOString(),
    next_renewal_date: endsAt.toISOString(),
    max_guests_per_event: limits.max_guests_per_event ?? 999999,
    max_events: limits.max_events_per_month ?? 999,
    max_venues: limits.max_venues ?? 3,
  });

  return Response.json({ success: true, plan: promo.plan, duration_days: promo.duration_days });
});
```

---

## P4b — `src/pages/Pricing.jsx` (promocode via backend; K4)

Vervang de volledige functie `handleApplyPromo` (van `const handleApplyPromo = async () => {` t/m de bijbehorende `};`) door:

```jsx
  const handleApplyPromo = async () => {
    if (!promoCode.trim() || !currentOrg) return;
    setPromoState('loading');
    setPromoResult(null);
    try {
      const res = await base44.functions.invoke('redeemPromoCode', {
        code: promoCode.trim().toUpperCase(),
        organizationId: currentOrg.id,
      });
      if (res.data?.success) {
        await refetchOrg();
        setPromoState('success');
        setPromoResult(`${res.data.plan.charAt(0).toUpperCase() + res.data.plan.slice(1)} plan geactiveerd voor ${res.data.duration_days} dagen!`);
        setPromoCode('');
      } else {
        setPromoState('error');
        setPromoResult(res.data?.error || 'Ongeldige promocode.');
      }
    } catch (e) {
      setPromoState('error');
      setPromoResult(e?.response?.data?.error || 'Ongeldige of verlopen promocode.');
    }
  };
```

> Let op: vereist dat backend-function `redeemPromoCode` (P4) eerst bestaat.

---

## P2 — `src/pages/EventDetails.jsx` (K2, M4, M5, H4, H1-gedeeltelijk)

### P2.1 Uitchecken repareren (K2)
**OUD**
```jsx
  const handleCheckIn = (guest) => {
    if (guest.checked_in && guest._checkin_paid === undefined && guest._checkin_free === undefined) {
      checkInMutation.mutate({ guestId: guest.id, data: { checked_in: false, check_in_time: null, plus_ones_checked_in: 0 } });
      return;
    }
    const totalPartySize = guest.party_size || 1;
    const checkinCount = (guest._checkin_paid !== undefined || guest._checkin_free !== undefined)
      ? (guest._checkin_paid || 0) + (guest._checkin_free || 0)
      : totalPartySize;
    const alreadyCheckedIn = guest.plus_ones_checked_in || 0;
```
**NIEUW**
```jsx
  const handleCheckIn = (guest) => {
    // Beslis altijd op basis van het actuele record; sommige componenten geven
    // een al-gewijzigd object door (bv. { ...guest, checked_in: false } bij uitchecken).
    const current = guests.find(g => g.id === guest.id) || guest;
    const wantsCheckout = guest.checked_in === false && current.checked_in;
    if ((current.checked_in || wantsCheckout) && guest._checkin_paid === undefined && guest._checkin_free === undefined) {
      checkInMutation.mutate({ guestId: current.id, data: { checked_in: false, check_in_time: null, plus_ones_checked_in: 0 } });
      return;
    }
    const totalPartySize = current.party_size || 1;
    const checkinCount = (guest._checkin_paid !== undefined || guest._checkin_free !== undefined)
      ? (guest._checkin_paid || 0) + (guest._checkin_free || 0)
      : totalPartySize;
    const alreadyCheckedIn = current.plus_ones_checked_in || 0;
```

**OUD**
```jsx
    if (hasCustomFields && guest._checkin_paid === undefined && guest._checkin_free === undefined && !guest.checked_in) {
      setSelectedGuestForCustomFields(guest);
    } else {
      checkInMutation.mutate({ guestId: guest.id, data: checkInData });
    }
```
**NIEUW**
```jsx
    if (hasCustomFields && guest._checkin_paid === undefined && guest._checkin_free === undefined && !current.checked_in) {
      setSelectedGuestForCustomFields(current);
    } else {
      checkInMutation.mutate({ guestId: current.id, data: checkInData });
    }
```

### P2.2 Gastenlijst-restricties ook in Check-in Station (M4)
**OUD**
```jsx
    return { total, checkedIn, remaining, rate, partial };
  }, [guests]);
```
**NIEUW**
```jsx
    return { total, checkedIn, remaining, rate, partial };
  }, [guests]);

  // Zelfde gastenlijst-restricties als de tabel, ook in het Check-in Station
  const checkInGuests = useMemo(() => {
    if (allowedGuestListIds === null) return guests;
    return guests.filter(g =>
      g.added_by_email === currentUser?.email ||
      (g.guest_list_id && allowedGuestListIds.has(g.guest_list_id))
    );
  }, [guests, allowedGuestListIds, currentUser]);
```

**OUD**
```jsx
                <NameSearchCheckIn
                  guests={filterCategory === 'all' && filterGuestLists.size === 0 ? guests : guests.filter(g => (filterCategory === 'all' || g.category === filterCategory) && (filterGuestLists.size === 0 || filterGuestLists.has(g.guest_list_id)))}
```
**NIEUW**
```jsx
                <NameSearchCheckIn
                  guests={filterCategory === 'all' && filterGuestLists.size === 0 ? checkInGuests : checkInGuests.filter(g => (filterCategory === 'all' || g.category === filterCategory) && (filterGuestLists.size === 0 || filterGuestLists.has(g.guest_list_id)))}
```

**OUD**
```jsx
                <QRScanCheckIn guests={guests} onCheckIn={handleCheckIn} />
```
**NIEUW**
```jsx
                <QRScanCheckIn guests={checkInGuests} onCheckIn={handleCheckIn} />
```

### P2.3 Bulk-inchecken synct personenteller (M5)
**OUD**
```jsx
    await Promise.all([...selectedGuestIds].map(id => base44.entities.Guest.update(id, { checked_in: checkIn, check_in_time: checkIn ? new Date().toISOString() : null })));
```
**NIEUW**
```jsx
    await Promise.all([...selectedGuestIds].map(id => {
      const g = guests.find(x => x.id === id);
      const ps = g?.party_size || 1;
      return base44.entities.Guest.update(id, { checked_in: checkIn, plus_ones_checked_in: checkIn ? ps : 0, check_in_time: checkIn ? new Date().toISOString() : null });
    }));
```

### P2.4 Offline queue verliest geen mislukte items meer (H4)
**OUD**
```jsx
    const syncQueue = async () => {
      setIsSyncing(true);
      const queue = getQueue();
      for (const item of queue) {
        try { await base44.entities.Guest.update(item.guestId, item.data); } catch (e) {}
      }
      clearQueue();
      queryClient.invalidateQueries({ queryKey: ['guests', eventId] });
      setIsSyncing(false);
      toast.success(`${queue.length} offline check-in${queue.length > 1 ? 's' : ''} gesynchroniseerd`);
    };
```
**NIEUW**
```jsx
    const syncQueue = async () => {
      setIsSyncing(true);
      const queue = getQueue();
      const failed = [];
      for (const item of queue) {
        try { await base44.entities.Guest.update(item.guestId, item.data); }
        catch (e) { failed.push(item); }
      }
      clearQueue();
      failed.forEach(item => enqueue(item.guestId, item.data));
      queryClient.invalidateQueries({ queryKey: ['guests', eventId] });
      setIsSyncing(false);
      const synced = queue.length - failed.length;
      if (synced > 0) toast.success(`${synced} offline check-in${synced > 1 ? 's' : ''} gesynchroniseerd`);
      if (failed.length > 0) toast.error(`${failed.length} check-in(s) niet gesynchroniseerd — ze blijven in de wachtrij`);
    };
```

### P2.5 Permissies: org-eigenaar + juiste gates (H1)
**OUD**
```jsx
    organizationId: currentUser?.current_organization_id,
```
**NIEUW**
```jsx
    organizationId: event?.organization_id || currentUser?.current_organization_id,
```

**OUD**
```jsx
const INHERITANCE_LABELS = {
  app_admin: { label: 'App Admin', color: 'bg-red-500' },
```
**NIEUW**
```jsx
const INHERITANCE_LABELS = {
  app_admin: { label: 'App Admin', color: 'bg-red-500' },
  org_owner: { label: 'Organisatie-eigenaar', color: 'bg-emerald-500' },
```

**OUD**
```jsx
  const navItems = [
    { icon: ArrowLeft, label: 'Dashboard', to: createPageUrl('Dashboard') },
    { icon: BarChart3, label: 'Statistieken', to: createPageUrl(`EventStatistics?id=${eventId}`) },
    ...(event.enable_table_plan && isAdmin ? [{ icon: Grid3x3, label: 'Tafelplan', to: createPageUrl(`TablePlan?id=${eventId}`) }] : []),
    ...(event.enable_table_plan && (isAdmin || userPermissions.can_use_table_manager) ? [{ icon: Users, label: 'Table Manager', to: createPageUrl(`TableManager?id=${eventId}`) }] : []),
    ...(isAdmin ? [{ icon: Shield, label: 'Machtigingen', to: createPageUrl(`EventPermissions?id=${eventId}`) }] : []),
    ...(isAdmin ? [{ icon: Settings, label: 'Aangepaste velden', to: createPageUrl(`CustomFieldsSettings?id=${eventId}`) }] : []),
    ...(isAdmin ? [{ icon: Eye, label: 'Zichtbare velden', to: createPageUrl(`CheckInSettings?id=${eventId}`) }] : []),
    ...(isAdmin ? [{ icon: Tag, label: 'Tags', to: createPageUrl(`TagManagement?id=${eventId}`) }] : []),
    ...(isAdmin ? [{ icon: List, label: 'Gastenlijsten', to: createPageUrl(`GuestLists?id=${eventId}`) }] : []),
  ];
```
**NIEUW**
```jsx
  const canManageEvent = isAdmin || userPermissions.can_manage_event;
  const canManageRoles = isAdmin || userPermissions.can_manage_roles;
  const navItems = [
    { icon: ArrowLeft, label: 'Dashboard', to: createPageUrl('Dashboard') },
    { icon: BarChart3, label: 'Statistieken', to: createPageUrl(`EventStatistics?id=${eventId}`) },
    ...(event.enable_table_plan && (isAdmin || userPermissions.can_manage_tables) ? [{ icon: Grid3x3, label: 'Tafelplan', to: createPageUrl(`TablePlan?id=${eventId}`) }] : []),
    ...(event.enable_table_plan && (isAdmin || userPermissions.can_use_table_manager || userPermissions.can_manage_tables) ? [{ icon: Users, label: 'Table Manager', to: createPageUrl(`TableManager?id=${eventId}`) }] : []),
    ...(canManageRoles ? [{ icon: Shield, label: 'Machtigingen', to: createPageUrl(`EventPermissions?id=${eventId}`) }] : []),
    ...(canManageEvent ? [{ icon: Settings, label: 'Aangepaste velden', to: createPageUrl(`CustomFieldsSettings?id=${eventId}`) }] : []),
    ...(canManageEvent ? [{ icon: Eye, label: 'Zichtbare velden', to: createPageUrl(`CheckInSettings?id=${eventId}`) }] : []),
    ...(canManageEvent ? [{ icon: Tag, label: 'Tags', to: createPageUrl(`TagManagement?id=${eventId}`) }] : []),
    ...(canManageEvent ? [{ icon: List, label: 'Gastenlijsten', to: createPageUrl(`GuestLists?id=${eventId}`) }] : []),
  ];
```

**OUD**
```jsx
            {isAdmin && (
              <Button variant="ghost" size="sm" className="text-white hover:bg-white/20" onClick={() => setShowEventDialog(true)}>
```
**NIEUW**
```jsx
            {canManageEvent && (
              <Button variant="ghost" size="sm" className="text-white hover:bg-white/20" onClick={() => setShowEventDialog(true)}>
```

**OUD**
```jsx
              {(isAdmin || userPermissions.can_admin) && (
```
**NIEUW**
```jsx
              {canManageEvent && (
```

**OUD**
```jsx
                {isAdmin && (
                  <>
                    <DropdownMenuItem onClick={async () => {
                      await base44.entities.Event.update(eventId, { is_template: !event.is_template });
```
**NIEUW**
```jsx
                {canManageEvent && (
                  <>
                    <DropdownMenuItem onClick={async () => {
                      await base44.entities.Event.update(eventId, { is_template: !event.is_template });
```

---

## P8 — `src/hooks/useEffectivePermissions.js` (H1: org-eigenaar = volledige rechten)

**OUD**
```js
  const orgRoleId = orgMembers[0]?.role_id;
  const { data: orgRoles = [] } = useQuery({
    queryKey: ['org-role', orgRoleId],
    queryFn: () => base44.entities.OrganizationRole.filter({ id: orgRoleId }),
    enabled: !!orgRoleId && !isAppAdmin,
  });
```
**NIEUW**
```js
  const orgRoleId = orgMembers[0]?.role_id;
  const { data: orgRoles = [] } = useQuery({
    queryKey: ['org-role', orgRoleId],
    queryFn: () => base44.entities.OrganizationRole.filter({ id: orgRoleId }),
    enabled: !!orgRoleId && !isAppAdmin,
  });

  // Organisatie-eigenaar → volledige rechten (owner heeft niet altijd een OrganizationRole)
  const { data: orgRecords = [] } = useQuery({
    queryKey: ['org-owner-check', organizationId],
    queryFn: () => base44.entities.Organization.filter({ id: organizationId }),
    enabled: !!organizationId && !!userEmail && !isAppAdmin,
  });
  const isOrgOwner = !!userEmail && orgRecords[0]?.owner_email === userEmail;
```

**OUD**
```js
  const effectivePermissions = useMemo(() => {
    if (isAppAdmin) return FULL_PERMISSIONS;
```
**NIEUW**
```js
  const effectivePermissions = useMemo(() => {
    if (isAppAdmin) return FULL_PERMISSIONS;
    if (isOrgOwner) return FULL_PERMISSIONS;
```

**OUD**
```js
  }, [isAppAdmin, eventPerms, userRoles, roles, venueRoles, orgRoles]);

  // Which guest lists the user may view explicitly
```
**NIEUW** *(let op: dit is de dependency-array van `effectivePermissions`)*
```js
  }, [isAppAdmin, isOrgOwner, eventPerms, userRoles, roles, venueRoles, orgRoles]);

  // Which guest lists the user may view explicitly
```

**OUD**
```js
  const allowedGuestListIds = useMemo(() => {
    if (isAppAdmin) return null;
```
**NIEUW**
```js
  const allowedGuestListIds = useMemo(() => {
    if (isAppAdmin || isOrgOwner) return null;
```

**OUD**
```js
  }, [isAppAdmin, eventPerms]);
```
**NIEUW**
```js
  }, [isAppAdmin, isOrgOwner, eventPerms]);
```

**OUD**
```js
  const inheritanceSource = useMemo(() => {
    if (isAppAdmin) return 'app_admin';
```
**NIEUW**
```js
  const inheritanceSource = useMemo(() => {
    if (isAppAdmin) return 'app_admin';
    if (isOrgOwner) return 'org_owner';
```

**OUD**
```js
    return 'default';
  }, [isAppAdmin, eventPerms, userRoles, roles, venueRoles, orgRoles]);
```
**NIEUW**
```js
    return 'default';
  }, [isAppAdmin, isOrgOwner, eventPerms, userRoles, roles, venueRoles, orgRoles]);
```

---

## P7 — `src/pages/Dashboard.jsx` (K7, M2, M3)

### P7.1 Events per organisatie i.p.v. hele app
**OUD**
```jsx
    queryFn: async () => {
      // If only 1 venue, filter server-side; otherwise fetch all and filter client-side
      if (venueIds.length === 1) {
        const events = await base44.entities.Event.filter({ venue_location_id: venueIds[0] });
        return events.map(e => ({ ...e, _orgId: currentOrg.id }));
      }
      const allEvents = await base44.entities.Event.list('-start_date');
      return allEvents
        .filter(e => !e.venue_location_id || venueIds.includes(e.venue_location_id))
        .map(e => ({ ...e, _orgId: currentOrg.id }));
    },
```
**NIEUW**
```jsx
    queryFn: async () => {
      // Server-side op organisatie filteren + venue-fallback voor oudere events zonder organization_id.
      // Nooit events van andere organisaties ophalen of tonen.
      const [orgEvents, ...venueEventLists] = await Promise.all([
        base44.entities.Event.filter({ organization_id: currentOrg.id }),
        ...venueIds.map(vid => base44.entities.Event.filter({ venue_location_id: vid })),
      ]);
      const seen = new Set();
      return [...orgEvents, ...venueEventLists.flat()]
        .filter(e => !e.organization_id || e.organization_id === currentOrg.id)
        .filter(e => { if (seen.has(e.id)) return false; seen.add(e.id); return true; })
        .map(e => ({ ...e, _orgId: currentOrg.id }));
    },
```

### P7.2 Maandlimiet: sjablonen niet meetellen (mutatie)
**OUD**
```jsx
        const eventsThisMonth = currentOrgEvents.filter(e => e.created_date >= startOfMonth && !e.is_unlocked_per_purchase).length;
```
**NIEUW**
```jsx
        const eventsThisMonth = currentOrgEvents.filter(e => e.created_date >= startOfMonth && !e.is_unlocked_per_purchase && !e.is_template).length;
```

### P7.3 Headerteller consistent met de echte check
**OUD**
```jsx
                  const eventsThisMonth = currentOrgEvents.filter(e => e.created_date >= startOfMonth).length;
```
**NIEUW**
```jsx
                  const eventsThisMonth = currentOrgEvents.filter(e => e.created_date >= startOfMonth && !e.is_unlocked_per_purchase && !e.is_template).length;
```

### P7.4 Niet-bestaand veld `rsvp_status` weg
**OUD**
```jsx
    return {
      total: sum(eventGuests),
      checkedIn: sum(eventGuests.filter(g => g.checked_in)),
      confirmed: sum(eventGuests.filter(g => g.rsvp_status === 'confirmed'))
    };
```
**NIEUW**
```jsx
    return {
      total: sum(eventGuests),
      checkedIn: sum(eventGuests.filter(g => g.checked_in)),
    };
```

---

## P1 — Event-unlock-flow repareren (K1)

### P1.1 `src/components/events/UnlockLimitDialog.jsx`
**OUD**
```jsx
export default function UnlockLimitDialog({ open, onClose, eventId, onUnlocked, limitType, organizationId }) {
  const [purchasing, setPurchasing] = useState(null);

  const handlePurchase = async (option) => {
    setPurchasing(option.plan);
    try {
      const response = await base44.functions.invoke('mollieCreatePayment', {
        type: 'event_unlock',
        eventPlan: option.plan,
        eventId: eventId || null,
        organizationId: organizationId || null,
      });
```
**NIEUW**
```jsx
export default function UnlockLimitDialog({ open, onClose, eventId, onUnlocked, onBeforePurchase, limitType, organizationId }) {
  const [purchasing, setPurchasing] = useState(null);

  const handlePurchase = async (option) => {
    setPurchasing(option.plan);
    try {
      // Bij aanmaak-limiet bestaat het event nog niet: eerst (draft) aanmaken,
      // anders heeft de Mollie-webhook geen eventId om te ontgrendelen.
      let targetEventId = eventId;
      if (!targetEventId && onBeforePurchase) {
        targetEventId = await onBeforePurchase(option);
        if (!targetEventId) {
          toast.error('Kon het evenement niet voorbereiden voor aankoop.');
          return;
        }
      }
      const response = await base44.functions.invoke('mollieCreatePayment', {
        type: 'event_unlock',
        eventPlan: option.plan,
        eventId: targetEventId || null,
        organizationId: organizationId || null,
      });
```

### P1.2 `src/pages/Dashboard.jsx` — dode `onUnlocked` vervangen
Vervang het volledige `<UnlockLimitDialog … />`-blok onderaan (van `<UnlockLimitDialog` t/m `/>` vóór het sluitende `</div>`) door:

```jsx
      <UnlockLimitDialog
        open={showUnlockDialog}
        onClose={() => { setShowUnlockDialog(false); setPendingEventData(null); }}
        eventId={null}
        limitType="event"
        organizationId={currentOrg?.id}
        onBeforePurchase={async () => {
          // Maak het evenement (incl. gastenlijst/tafels) aan vóór de betaling,
          // zodat de Mollie-webhook het via eventId kan ontgrendelen.
          if (!pendingEventData) return null;
          const { first_guest_list_name, _template_id, ...eventPayload } = pendingEventData;
          const newEvent = await base44.entities.Event.create({ ...eventPayload, status: 'draft', organization_id: currentOrg?.id });
          if (_template_id) {
            const [templateLists, templateTables] = await Promise.all([
              base44.entities.GuestList.filter({ event_id: _template_id }),
              newEvent.venue_location_id
                ? base44.entities.Table.filter({ venue_location_id: newEvent.venue_location_id, event_id: _template_id })
                : Promise.resolve([]),
            ]);
            await Promise.all([
              templateLists.length > 0
                ? templateLists.map(({ id, created_date, updated_date, created_by, ...listData }) =>
                    base44.entities.GuestList.create({ ...listData, event_id: newEvent.id })
                  )
                : [base44.entities.GuestList.create({ event_id: newEvent.id, name: first_guest_list_name || 'Gasten', is_default: true, color: '#3b82f6', sort_order: 0 })],
              ...templateTables
                .filter(t => t.table_number !== '__zone_placeholder__')
                .map(({ id, created_date, updated_date, created_by, ...tableData }) =>
                  base44.entities.Table.create({ ...tableData, event_id: newEvent.id })
                ),
            ].flat());
          } else {
            await base44.entities.GuestList.create({
              event_id: newEvent.id,
              name: first_guest_list_name || 'Gasten',
              is_default: true,
              color: '#3b82f6',
              sort_order: 0
            });
          }
          queryClient.invalidateQueries({ queryKey: ['events', currentOrg?.id] });
          setShowEventDialog(false);
          setPendingEventData(null);
          return newEvent.id;
        }}
      />
```

---

## P9 — `src/components/guests/AddGuestPanel.jsx` (H2, M9)

### P9.1 Fout tonen bij enkel-gast opslaan
**OUD**
```jsx
      toast.success(`${formData.full_name} toegevoegd`);
      setFormData(f => ({ full_name: '', email: '', phone: '', party_size: 1, plus_ones_allowed: 0, notes: '', guest_list_id: f.guest_list_id, table_id: '' }));
      setCustomFieldValues({});
    } finally {
      setSaving(false);
    }
```
**NIEUW**
```jsx
      toast.success(`${formData.full_name} toegevoegd`);
      setFormData(f => ({ full_name: '', email: '', phone: '', party_size: 1, plus_ones_allowed: 0, notes: '', guest_list_id: f.guest_list_id, table_id: '' }));
      setCustomFieldValues({});
    } catch (e) {
      toast.error('Gast opslaan mislukt: ' + (e?.message || 'onbekende fout'));
    } finally {
      setSaving(false);
    }
```

### P9.2 Bestandsimport: limiet + token/afzender
**OUD**
```jsx
  const handleFileImport = async () => {
    if (!fileRows?.length) return;
    setFileImporting(true);
    let ok = 0;
    for (const g of fileRows) {
      try {
        await base44.entities.Guest.create({
          event_id: eventId,
          full_name: g.full_name,
          email: g.email || '',
          phone: g.phone || '',
          party_size: Math.max(1, g.party_size || 1),
          plus_ones_allowed: Math.max(0, g.plus_ones_allowed || 0),
          notes: g.notes || '',
          guest_list_id: formData.guest_list_id || undefined,
        });
        ok++;
      } catch {}
    }
```
**NIEUW**
```jsx
  const handleFileImport = async () => {
    if (!fileRows?.length) return;
    const limitError = checkLimit();
    if (limitError) { setShowUnlockDialog(true); return; }
    const importMax = event?.max_guests_override ?? (orgData && !orgData.unlimited ? (PLAN_LIMITS[orgData.subscription_plan] || PLAN_LIMITS['free']).max_guests_per_event : null);
    if (importMax != null && currentGuestCount + fileRows.length > importMax) {
      toast.error(`Import van ${fileRows.length} gasten overschrijdt de limiet (${currentGuestCount}/${importMax}).`);
      setShowUnlockDialog(true);
      return;
    }
    setFileImporting(true);
    let me; try { me = await base44.auth.me(); } catch {}
    let ok = 0;
    const failed = [];
    for (const g of fileRows) {
      try {
        await base44.entities.Guest.create({
          event_id: eventId,
          full_name: g.full_name,
          email: g.email || '',
          phone: g.phone || '',
          party_size: Math.max(1, g.party_size || 1),
          plus_ones_allowed: Math.max(0, g.plus_ones_allowed || 0),
          notes: g.notes || '',
          guest_list_id: formData.guest_list_id || undefined,
          unique_access_token: `gec-${Math.random().toString(36).slice(2)}${Math.random().toString(36).slice(2)}`,
          added_by_email: me?.email,
        });
        ok++;
      } catch { failed.push(g.full_name); }
    }
    if (failed.length > 0) toast.error(`${failed.length} gast(en) niet geïmporteerd: ${failed.slice(0, 3).join(', ')}${failed.length > 3 ? '…' : ''}`);
```

### P9.3 Tekstimport: idem
**OUD**
```jsx
  const handleTextImport = async () => {
    const lines = textInput.trim().split('\n').map(parseLine).filter(Boolean);
    if (!lines.length) return;
    setTextImporting(true);
    let ok = 0;
    for (const g of lines) {
      try {
        await base44.entities.Guest.create({
          event_id: eventId,
          full_name: g.name,
          party_size: Math.max(1, g.tickets || 1),
          plus_ones_allowed: Math.max(0, g.free || 0),
          notes: g.notes || '',
          email: g.email || '',
          guest_list_id: textGuestListId || undefined,
        });
        ok++;
      } catch {}
    }
```
**NIEUW**
```jsx
  const handleTextImport = async () => {
    const lines = textInput.trim().split('\n').map(parseLine).filter(Boolean);
    if (!lines.length) return;
    const limitError = checkLimit();
    if (limitError) { setShowUnlockDialog(true); return; }
    const importMax = event?.max_guests_override ?? (orgData && !orgData.unlimited ? (PLAN_LIMITS[orgData.subscription_plan] || PLAN_LIMITS['free']).max_guests_per_event : null);
    if (importMax != null && currentGuestCount + lines.length > importMax) {
      toast.error(`Import van ${lines.length} gasten overschrijdt de limiet (${currentGuestCount}/${importMax}).`);
      setShowUnlockDialog(true);
      return;
    }
    setTextImporting(true);
    let me; try { me = await base44.auth.me(); } catch {}
    let ok = 0;
    const failed = [];
    for (const g of lines) {
      try {
        await base44.entities.Guest.create({
          event_id: eventId,
          full_name: g.name,
          party_size: Math.max(1, g.tickets || 1),
          plus_ones_allowed: Math.max(0, g.free || 0),
          notes: g.notes || '',
          email: g.email || '',
          guest_list_id: textGuestListId || undefined,
          unique_access_token: `gec-${Math.random().toString(36).slice(2)}${Math.random().toString(36).slice(2)}`,
          added_by_email: me?.email,
        });
        ok++;
      } catch { failed.push(g.name); }
    }
    if (failed.length > 0) toast.error(`${failed.length} gast(en) niet geïmporteerd: ${failed.slice(0, 3).join(', ')}${failed.length > 3 ? '…' : ''}`);
```

---

## P10 — `src/components/checkin/NameSearchCheckIn.jsx` (H3: tafelcapaciteit)

**OUD**
```jsx
                      const assignedGuests = guests.filter(g => g.table_id === table.id).length;
                      const isSelected = selectedGuest.table_id === table.id;
                      const isFull = assignedGuests > 0 && !isSelected;
```
**NIEUW**
```jsx
                      const assignedGuests = guests.filter(g => g.table_id === table.id).reduce((s, g) => s + (g.party_size || 1), 0);
                      const isSelected = selectedGuest.table_id === table.id;
                      const isFull = assignedGuests >= (table.capacity || 8) && !isSelected;
```

---

## P15 — `src/components/checkin/QRScanCheckIn.jsx` (M8: spookveld weg)

**OUD**
```jsx
                        <Badge variant="outline">
                          Party of {scannedGuest.party_size || 1}
                        </Badge>
                        <Badge className={cn(
                          scannedGuest.rsvp_status === 'confirmed' ? 'bg-green-500' : 'bg-orange-500'
                        )}>
                          {scannedGuest.rsvp_status}
                        </Badge>
```
**NIEUW**
```jsx
                        <Badge variant="outline">
                          Gezelschap van {scannedGuest.party_size || 1}
                        </Badge>
```

---

## P14 — `src/pages/InvitationManager.jsx` (M7)

**OUD**
```jsx
      setShowForm(false);
      toast.success('Uitnodiging aangemaakt');
    }
  });
```
**NIEUW**
```jsx
      setShowForm(false);
      toast.success('Uitnodiging aangemaakt');
    },
    onError: (e) => toast.error('Aanmaken mislukt: ' + (e?.message || 'onbekende fout'))
  });
```

**OUD**
```jsx
                        onClick={() => deleteMutation.mutate(inv.id)}
```
**NIEUW**
```jsx
                        onClick={() => { if (confirm(`Uitnodiging voor ${inv.guest_name} verwijderen?`)) deleteMutation.mutate(inv.id); }}
```

---

## P6 — `src/pages/GuestConfirmation.jsx` (K6: geen ID-enumeratie)

**OUD**
```jsx
        let guests = [];

        // Try unique_access_token first
        if (token.startsWith('gec-')) {
          guests = await base44.entities.Guest.filter({ unique_access_token: token });
        }
        // Fallback: try by ID (legacy links)
        if (!guests.length) {
          guests = await base44.entities.Guest.filter({ id: token });
        }
```
**NIEUW**
```jsx
        // Uitsluitend via het unieke toegangstoken — een ID-fallback maakt
        // het mogelijk om gasten op te vragen door ID's te raden.
        const guests = await base44.entities.Guest.filter({ unique_access_token: token });
```

---

## P12 — `src/pages/PaymentReturn.jsx` (H5: eerlijke copy)

**OUD**
```jsx
            {success ? 'Betaling geslaagd!' : 'Betaling geannuleerd'}
```
**NIEUW**
```jsx
            {success ? 'Bedankt voor je bestelling!' : 'Betaling geannuleerd'}
```

**OUD**
```jsx
              ? 'Je aankoop is verwerkt. Het kan even duren voordat je plan wordt geactiveerd.'
```
**NIEUW**
```jsx
              ? 'Zodra Mollie de betaling bevestigt, wordt je aankoop automatisch geactiveerd (meestal binnen een minuut). Is de betaling niet gelukt, dan verandert er niets aan je plan.'
```

---

## P5b — `src/pages/Pricing.jsx` (K5: promopad-limieten consistent)

**OUD**
```jsx
          max_guests_per_event: limits.max_guests_per_event ?? 500,
          max_events: limits.max_events_per_month ?? 10,
          max_venues: limits.max_venues ?? 3,
```
**NIEUW**
```jsx
          max_guests_per_event: limits.max_guests_per_event ?? 999999,
          max_events: limits.max_events_per_month ?? 999,
          max_venues: limits.max_venues ?? 3,
```
> Vervalt als P4b (backend-promocode) volledig wordt toegepast — dan bestaat dit codepad niet meer.

---

## Opruimen (⚪)

- Verwijder `src/components/utils/subscriptionLimits.jsx` (dode code met crashgevoelige tabel).
- Verwijder `src/pages/Passes.jsx` (leeg bestand, nergens geïmporteerd).
