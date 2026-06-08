# PRD - card-app

## Version

| Field   | Value        |
| ------- | ------------ |
| Product | card-app     |
| Version | 0.3          |
| Status  | Draft        |
| Author  | Product Team |
| Date    | 2026-06-08   |

---

# 1. Executive Summary

**Problem Statement**
Collectible card hobbyists juggle several disconnected tools to catalog collections, identify cards, find trade partners, and track value. No single modern, user-friendly platform combines these.

**Proposed Solution**
A mobile app (iOS/Android) where users register collections, maintain wishlists, discover other collectors, and arrange peer-to-peer trades. MVP focuses on Pokémon; architecture is multi-TCG ready.

**Success Criteria**

Success in MVP is qualitative: members of the existing community use the app to find and complete real trades, and no fraud reports occur in the first 3 months.

---

# 2. Vision

card-app lets collectors register collections, find other collectors, make trades, and track card value in one place.

MVP targets Pokémon cards. The data model and card-source abstraction are designed to add more TCGs later (Magic: The Gathering, Disney Lorcana, One Piece Card Game, Yu-Gi-Oh!) without schema rewrite.

---

# 3. Target Audience

## User Personas

**Primary — "The Active Collector" (Alex, 24)**
Owns 300+ Pokémon cards. Wants overview of the collection and to trade duplicates. Mobile-first, checks app several times a week.

**Primary — "The Hobby Trader" (Sam, 31)**
Trades regularly, values reputation and reliable counterparties. Needs fast browse + messaging.

**Secondary**
Parents of young collectors; professional dealers; collectible-card investors. Not optimized for in MVP.

---

# 4. MVP Scope — User Experience & Functionality

> Acceptance Criteria (AC) define "Done" per story. All AC are testable.
> App language: English only.

## 4.1 User Authentication

**Story**: As a user, I want to create an account so my collection is saved across devices.

**Features**: Login with Apple, Login with Google, Logout, User profile management.

**Acceptance Criteria**
- User can sign in with Apple and with Google via Supabase Auth (OAuth 2.0).
- Session persists across app restarts; logout clears session on device.
- First successful login auto-creates a `User` row.
- No email/password flow in MVP (see Non-Goals).

## 4.2 User Profile

**Story**: As a user, I want to introduce myself to other collectors.

**Features**: Profile picture, display name, biography, country, city, public/private toggle.

**Acceptance Criteria**
- User edits display name, bio (≤ 300 chars), country, city; changes persist.
- Avatar upload ≤ 5 MB, stored in Supabase Storage, served via CDN URL.
- Private profile hides collection, wishlist, and profile from non-owners in all list/search views.
- Email address is never exposed to other users (RLS: readable by account owner only).

## 4.3 Collection Management

**Story**: As a user, I want to register my collection to get an overview of my cards.

**Features**: Add/edit/remove card, set quantity, set condition, mark available-for-trade.

**Supported Conditions**: Mint, Near Mint, Excellent, Good, Played.

**Acceptance Criteria**
- Add card from card database to collection with quantity ≥ 1 and a condition.
- The same card can exist as multiple rows with different conditions (e.g. 2× Charizard NM + 1× Charizard Played). Duplicate entries for the same `(user_id, card_id, condition)` are prevented.
- Edit quantity/condition/for_trade; remove card.
- Cards involved in an `Accepted` trade display a "currently in a trade" indicator and cannot be added to new trade offers until the lock is released.
- Collection list renders 500 items with scroll at ≥ 55 FPS on a mid-tier device (e.g. iPhone 12 / Pixel 6).

## 4.4 Card Database

**Story**: As a user, I want to find card information before I buy or trade.

**Features**: Search by name, filter by set / rarity / Pokémon; show card info + image.

**Data source**: Pokémon TCG API (pokemontcg.io). Cards synced into local `Card` table; images served from source CDN.

**Acceptance Criteria**
- Text search returns results in ≤ 500 ms (p95) over the synced Pokémon catalog.
- Filters (set, rarity, Pokémon) combine and return correct subsets.
- Card detail shows name, set, number, rarity, image, and Cardmarket price when available.
- Catalog sync job runs on a schedule; new sets appear within 24 h of source update.

## 4.5 Wishlist

**Story**: As a user, I want to show which cards I'm looking for so others can offer relevant trades.

**Features**: Add/remove card, prioritize, public wishlist.

**Acceptance Criteria**
- Add card to wishlist with priority (low/medium/high).
- Wishlist visible on public profile; hidden when profile is private.
- Duplicate wishlist entry for same card is prevented.

## 4.6 Collector Discovery

**Story**: As a user, I want to find collectors who have cards I need.

**Features**: Explorer screen, matchmaking score, location filter, user search.

**Acceptance Criteria**
- Explorer screen lists collectors ranked by matchmaking score: number of their `for_trade` cards that overlap with the viewer's wishlist.
- Score and number of matching cards are shown per collector card in the list.
- Filter by country and city (using existing `User.country` / `User.city` fields — no GPS required).
- Search by username.
- Private profiles are excluded from all discovery views.

## 4.7 Trading

**Story**: As a user, I want to arrange trades with other collectors.

**Model**: App matches and facilitates. Users ship cards privately off-platform. No escrow, no payments, no shipping integration in MVP. A "trade at own risk" disclaimer is shown before a user submits their first trade offer.

**Trade Builder**: Central screen for composing offers, reachable from the Explorer screen, a collector's profile, or a wishlist-match notification. Left side = their `for_trade` cards you select; right side = your `for_trade` cards you offer. Wishlist-match notifications pre-fill the matching card.

**Features**: Create trade offer, counter-offer, accept, reject, cancel, confirm received, view history, rate counterparty.

**Trade Statuses**: Pending, Countered, Accepted, Rejected, Cancelled, Completed, Expired, Disputed.

**Trade Lifecycle**
- Sender builds offer via Trade Builder → status `Pending`.
- Offer expires after **7 days** if no action (`expires_at`); status → `Expired`.
- Receiver can Accept (→ `Accepted`), Reject (→ `Rejected`), or Counter (→ `Countered`).
- When `Countered`, roles flip: the other party receives a notification and may Accept, Reject, or Counter again. Both parties can counter; rounds are unlimited.
- Sender can Cancel a `Pending` or `Countered` offer (→ `Cancelled`).
- After `Accepted`, both parties tap "Confirm received" to move to `Completed`. One-sided confirm stays `Accepted`.
- A `confirmation_deadline` is set 14 days from `Accepted`. An automated reminder fires at day 7. If deadline passes without both confirmations → status `Disputed`; both parties are notified to contact support.
- At `Completed`, both parties can leave a 👍 or 👎 rating for the counterparty.

**Card Locking**
- When a trade reaches `Accepted`, all involved `CollectionCard` rows are locked (`locked_by_trade_id`). Locked cards cannot be included in new offers.
- Lock is released when status moves to `Completed`, `Rejected`, `Cancelled`, `Expired`, or `Disputed`.

**Acceptance Criteria**
- All status transitions above work correctly and notify the counterparty (§4.9).
- Expired offers are automatically moved to `Expired` by a scheduled Edge Function.
- Disputed trades are automatically flagged after `confirmation_deadline` and an email alert is sent to the app owner.
- History view lists past trades with status, timestamps, and ratings.
- "Trade at own risk" disclaimer shown before first offer submission.

## 4.8 Reputation

**Story**: As a user, I want to know if a trade partner is trustworthy.

**Features**: 👍/👎 rating per completed trade, reputation score on profile.

**Acceptance Criteria**
- After a trade reaches `Completed`, both parties can rate the experience (👍 or 👎).
- Profile displays reputation as "X of Y trades positive" (e.g. "47 of 50 trades positive").
- Rating is optional; a trade can complete without a rating from either party.
- Rating can only be submitted once per trade per user; cannot be changed after submission.

## 4.9 Messaging

**Story**: As a user, I want to discuss a trade before completing it.

**Features**: Full 1:1 direct messages, message threads linked to a trade, read receipts.

**Acceptance Criteria**
- Real-time 1:1 messages via Supabase Realtime; delivered ≤ 2 s when both online.
- A trade has an associated thread reachable from the trade detail.
- Users can open a DM with any public-profile collector.
- Read receipt shows when the counterparty has read the latest message.
- Users can report a message/user — triggers auto-block (reporter can no longer be contacted by the reported user) and sends an email alert to the app owner.

## 4.10 Notifications

**Story**: As a user, I want to be notified of important events.

**Features**: New trade request, counter-offer received, accepted trade, new message, wishlist match, trade confirmation reminder.

**Acceptance Criteria**
- Push via Expo Notifications on: new `Pending` request, `Countered`, `Accepted`, `Rejected`, new message, wishlist match (someone marks a wishlisted card `for_trade`), and day-7 confirmation reminder.
- User can toggle each notification category in settings.
- Tapping a notification deep-links to the relevant trade/message.

## 4.11 Non-Goals (NOT building in MVP)

- No marketplace, buying/selling, or integrated payments.
- No monetization / paywalls (deferred to v1.1).
- No escrow, shipping labels, or platform-mediated dispute resolution.
- No AI card recognition (photo → identify).
- No graded-card support (PSA/Beckett/CGC).
- No non-Pokémon TCGs (data model ready; not populated).
- No email/password login (social only).
- No web client (mobile only).
- No GPS-based location (city/country only).
- No in-app GDPR export/delete (manual via email — see §5.3).

---

# 5. Technical Specifications

## 5.1 Architecture Overview

```text
Mobile (React Native + Expo + TypeScript)
    │  OAuth (Apple/Google) → Supabase Auth
    │  REST/RPC + Realtime  → Supabase (PostgreSQL, Edge Functions)
    │  Media               → Supabase Storage (CDN)
    │
Edge Functions:
    - Card catalog sync      ← Pokémon TCG API (scheduled)
    - Price guide sync       ← Cardmarket Price Guide CSV (daily)
    - Trade expiry sweep     ← marks Expired / Disputed trades (scheduled)
    - Notification fan-out   → Expo Notifications
```

Data flow: client reads cards/prices from local Postgres (synced copies, not live third-party calls on the hot path). Trades/messages/wishlist are first-party Postgres + Realtime.

## 5.2 Integration Points

- **Auth**: Supabase Auth — Apple Sign In, Google Sign In (OAuth 2.0).
- **Card catalog**: Pokémon TCG API (pokemontcg.io) — scheduled sync into `Card`.
- **Pricing**: Cardmarket Price Guide CSV — daily scheduled sync into `Card.market_value`. No direct Cardmarket API dependency (official API is closed to new applicants).
- **Push**: Expo Notifications.
- **Storage**: Supabase Storage for avatars.

## 5.3 Security & Privacy

- OAuth 2.0; TLS 1.2+ in transit; encrypted storage at rest.
- GDPR: account deletion and data export handled manually — user emails support address, owner runs a Supabase script within 30 days (compliant for small operators). In-app automation deferred to v1.1.
- `User.email` protected by RLS: readable only by `auth.uid() = id`.
- Private-profile rule enforced server-side (RLS policies), not just client-side.
- Abuse: report triggers auto-block + email alert to app owner. No manual moderation queue in MVP.

## 5.4 Non-Functional Requirements

- **Performance**: cold app start ≤ 3 s (p95, mid-tier device, 4G). API p95 ≤ 500 ms for read endpoints under 100 concurrent users.
- **Scalability**: design for 100,000 users and 10 M card records (synced multi-TCG headroom).
- **Availability**: 99.9% monthly for core API.
- **Reliability**: daily automated backups; documented disaster-recovery runbook.

---

# 6. Data Model

```sql
User
----
id, username, email, avatar_url, bio, country, city,
is_public, created_at
-- email: RLS read restricted to owner only

Card                         -- synced from Pokémon TCG API
----
id, external_id, game, name, set, number, rarity, image_url,
market_value, currency, price_fetched_at

CollectionCard
----
id, user_id, card_id, quantity, condition, for_trade,
locked_by_trade_id           -- NULL when not in an active Accepted trade
-- UNIQUE (user_id, card_id, condition)

WishlistItem
----
id, user_id, card_id, priority

Trade
----
id, sender_id, receiver_id, status,
last_actor_id,               -- tracks whose turn it is
expires_at,                  -- 7 days from creation; enforced by scheduled Edge Function
confirmation_deadline,       -- 14 days from Accepted; enforced by scheduled Edge Function
sender_confirmed, receiver_confirmed,
created_at, updated_at

TradeItem
----
id, trade_id, collection_card_id, quantity, side   -- side = offered | requested
-- collection_card_id → CollectionCard (carries card_id, condition, for_trade)

TradeRating
----
id, trade_id, rater_id, rated_user_id, positive    -- positive = bool (👍/👎)
-- UNIQUE (trade_id, rater_id)

Message
----
id, sender_id, receiver_id, trade_id (nullable),
body, read_at, created_at
```

---

# 7. Risks & Roadmap

## 7.1 Technical & Business Risks

| Risk | Impact | Mitigation |
| ---- | ------ | ---------- |
| Pokémon TCG API no SLA / rate limits / goes away | No catalog | Cache full catalog in Postgres; abstract source behind sync layer; evaluate fallback provider. |
| Cardmarket CSV format changes / goes away | No pricing | Daily sync with schema validation; graceful degrade (hide price field) if CSV unavailable. |
| Trade fraud (off-platform shipping, no escrow) | User trust loss | Two-sided confirm; 👍/👎 reputation from day 1; `Disputed` escalation; "trade at own risk" disclaimer; report + auto-block. |
| Abuse in messaging / profiles | Safety, legal | Report → auto-block + owner email alert; rate limits. |
| GDPR non-compliance | Legal | Manual export/delete within 30 days; RLS; EU residency. |
| Apple App Store rejection | No iOS launch | Review App Store guidelines pre-submission; include "trade at own risk" disclaimer; age rating 13+; TestFlight validation before public release. |
| Notification fan-out cost/latency at scale | Engagement drop | Batch via Edge Functions; per-category opt-out. |

## 7.2 Phased Rollout

**MVP (v1.0)** — Auth, Profile, Collection, Card DB (Pokémon), Wishlist, Collector Discovery (matchmaking + location), Trading (counter-offers, locking, Disputed flow), Reputation (👍/👎), Messaging (full DM), Notifications.

**v1.1** — Monetization (card-limit freemium + Collector Pro tier); in-app GDPR export/delete; counter-offer UX improvements; market value tracking (price graph, history, alerts); richer search; GPS-based discovery; abuse tooling hardening.

**v2.0** — Additional TCGs; AI card recognition; marketplace exploration (buy/sell, payments, ratings); graded-card support (PSA/Beckett/CGC).

---

# 8. MVP Release Criteria

Launch when:

- Social login (Apple + Google) works end-to-end.
- Collection management works (add/edit/remove/quantity/condition/for_trade/locking).
- Multiple conditions per card supported with duplicate prevention.
- Card database search + filters meet §4.4 AC.
- Wishlist works incl. public/private rules.
- Collector discovery with matchmaking score + city/country filter works.
- Trading full lifecycle works: Pending → Countered → Accepted → Completed / Disputed / Expired.
- Trade Builder reachable from all three entry points.
- Card locking on Accepted, auto-release on terminal states.
- Reputation (👍/👎) submittable after Completed.
- Real-time messaging + read receipts work.
- Notifications fire for all §4.10 events with per-category toggle.
- "Trade at own risk" disclaimer shown before first offer.
- GDPR manual export/delete process documented and tested.
- RLS private-profile and email enforcement verified.
- Report + auto-block flow live; owner email alert confirmed working.
- App Store Guidelines reviewed; age rating 13+ set; TestFlight build validated.
- Security review completed.
