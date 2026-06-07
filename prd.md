# PRD - card-app

## Version

| Field   | Value        |
| ------- | ------------ |
| Product | card-app     |
| Version | 0.2          |
| Status  | Draft        |
| Author  | Product Team |
| Date    | 2026-06-07   |

---

# 1. Executive Summary

**Problem Statement**
Collectible card hobbyists juggle several disconnected tools to catalog collections, identify cards, find trade partners, and track value. No single modern, user-friendly platform combines these.

**Proposed Solution**
A mobile app (iOS/Android) where users register collections, maintain wishlists, discover other collectors, and arrange peer-to-peer trades. MVP focuses on Pokémon; architecture is multi-TCG ready.

**Success Criteria (KPIs)**

1. 1,000 registered users within month 1; 10,000 within year 1.
2. D30 retention ≥ 30%.
3. DAU/MAU ratio ≥ 20% (sustained engagement).
4. ≥ 500 completed trades per month by month 6.
5. Median ≥ 10 cards registered per active user.

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

## 4.1 User Authentication

**Story**: As a user, I want to create an account so my collection is saved across devices.

**Features**: Login with Apple, Login with Google, Logout, User profile management.

**Acceptance Criteria**
- User can sign in with Apple and with Google via Supabase Auth (OAuth 2.0).
- Session persists across app restarts; logout clears session on device.
- First successful login auto-creates a `User` row.
- No email/password flow in MVP (see Non-Goals). No "password reset" — removed (was a contradiction; no password exists).

## 4.2 User Profile

**Story**: As a user, I want to introduce myself to other collectors.

**Features**: Profile picture, display name, biography, country, city, public/private toggle.

**Acceptance Criteria**
- User edits display name, bio (≤ 300 chars), country, city; changes persist.
- Avatar upload ≤ 5 MB, stored in Supabase Storage, served via CDN URL.
- Private profile hides collection, wishlist, and profile from non-owners in all list/search views.

## 4.3 Collection Management

**Story**: As a user, I want to register my collection to get an overview of my cards.

**Features**: Add/edit/remove card, set quantity, set condition, mark available-for-trade.

**Supported Conditions**: Mint, Near Mint, Excellent, Good, Played.

**Acceptance Criteria**
- Add card from card database to collection with quantity ≥ 1 and a condition.
- Edit quantity/condition/for_trade; remove card.
- Collection list renders 500 items with scroll at ≥ 55 FPS on a mid-tier device (e.g. iPhone 12 / Pixel 6).

## 4.4 Card Database

**Story**: As a user, I want to find card information before I buy or trade.

**Features**: Search by name, filter by set / rarity / Pokémon; show card info + image.

**Data source**: Pokémon TCG API (pokemontcg.io). Cards synced into local `Card` table; images served from source CDN.

**Acceptance Criteria**
- Text search returns results in ≤ 500 ms (p95) over the synced Pokémon catalog.
- Filters (set, rarity, Pokémon) combine and return correct subsets.
- Card detail shows name, set, number, rarity, image, and Cardmarket value when available.
- Catalog sync job runs on a schedule; new sets appear within 24h of source update.

## 4.5 Wishlist

**Story**: As a user, I want to show which cards I'm looking for so others can offer relevant trades.

**Features**: Add/remove card, prioritize, public wishlist.

**Acceptance Criteria**
- Add card to wishlist with priority (low/medium/high).
- Wishlist visible on public profile; hidden when profile is private.
- Duplicate wishlist entry for same card is prevented.

## 4.6 Trading (off-platform / trust model)

**Story**: As a user, I want to arrange trades with other collectors.

**Model**: App matches and facilitates. Users ship cards privately off-platform. No escrow, no payments, no shipping integration in MVP.

**Features**: Create trade offer, send request, accept, reject, cancel, view history.

**Trade Statuses**: Pending, Accepted, Rejected, Cancelled, Completed.

**Acceptance Criteria**
- Sender builds an offer of their for_trade cards ↔ requested cards, sends to a receiver → status `Pending`.
- Receiver can Accept (→ `Accepted`) or Reject (→ `Rejected`); sender can Cancel a Pending offer (→ `Cancelled`).
- After Accepted, **both** parties tap "Confirm received" to move to `Completed`. One-sided confirm stays Accepted.
- All status changes notify the counterparty (§4.8).
- History view lists past trades with status and timestamps.

## 4.7 Messaging

**Story**: As a user, I want to discuss a trade before completing it.

**Features**: Direct messages, messages linked to a trade, read receipts.

**Acceptance Criteria**
- Real-time 1:1 messages via Supabase Realtime; delivered ≤ 2 s when both online.
- A trade has an associated thread reachable from the trade detail.
- Read receipt shows when the counterparty has read the latest message.
- Users can report a message/user (routes to moderation queue — see Risks).

## 4.8 Notifications

**Story**: As a user, I want to be notified of important events.

**Features**: New trade request, accepted trade, new message, wishlist match.

**Acceptance Criteria**
- Push via Expo Notifications on: new Pending request, Accept/Reject, new message, and wishlist match (someone marks a wishlisted card for_trade).
- User can toggle each notification category in settings.
- Tapping a notification deep-links to the relevant trade/message.

## 4.9 Non-Goals (NOT building in MVP)

- No marketplace, buying/selling, or integrated payments.
- No escrow, shipping labels, or platform-mediated dispute resolution.
- No AI card recognition (photo → identify).
- No graded-card support (PSA/Beckett/CGC).
- No non-Pokémon TCGs (data model ready; not populated).
- No email/password login (social only).
- No web client (mobile only).

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
    - Card catalog sync  ← Pokémon TCG API (scheduled)
    - Market value sync  ← Cardmarket (EU) (scheduled)
    - Notification fan-out → Expo Notifications
```

Data flow: client reads cards/prices from local Postgres (synced copies, not live third-party calls on the hot path). Trades/messages/wishlist are first-party Postgres + Realtime.

## 5.2 Integration Points

- **Auth**: Supabase Auth — Apple Sign In, Google Sign In (OAuth 2.0).
- **Card catalog**: Pokémon TCG API (pokemontcg.io) — scheduled sync into `Card`.
- **Pricing**: Cardmarket (EU) — scheduled sync into `Card.market_value` (+ currency, fetched_at). EU pricing suits Swedish/European base.
- **Push**: Expo Notifications.
- **Storage**: Supabase Storage for avatars.

## 5.3 Security & Privacy

- OAuth 2.0; TLS 1.2+ in transit; encrypted storage at rest.
- GDPR: account + data export and deletion ("right to be forgotten"); EU data residency where Supabase supports it.
- Private-profile rule enforced server-side (RLS policies), not just client-side.
- Abuse: report flow + moderation queue for messages and profiles.

## 5.4 Non-Functional Requirements

- **Performance**: cold app start ≤ 3 s (p95, mid-tier device, 4G). API p95 ≤ 500 ms for read endpoints under 100 concurrent users.
- **Scalability**: design for 100,000 users and 10M card records (synced multi-TCG headroom).
- **Availability**: 99.9% monthly for core API.
- **Reliability**: daily automated backups; documented disaster-recovery runbook.

---

# 6. Data Model

```sql
User
----
id, username, email, avatar_url, bio, country, city,
is_public, created_at

Card                         -- synced from Pokémon TCG API
----
id, external_id, game, name, set, number, rarity, image_url,
market_value, currency, price_fetched_at

CollectionCard
----
id, user_id, card_id, quantity, condition, for_trade

WishlistItem
----
id, user_id, card_id, priority

Trade
----
id, sender_id, receiver_id, status,
sender_confirmed, receiver_confirmed, created_at, updated_at

TradeItem
----
id, trade_id, card_id, quantity, side   -- side = offered | requested
```

Notes vs v0.1: added `is_public` (User); `external_id`, `game`, `currency`, `price_fetched_at` (Card, for multi-TCG + pricing provenance); `sender_confirmed`/`receiver_confirmed` (Trade, for two-sided Completed); `side` (TradeItem). Dropped `for_sale` (no marketplace in MVP).

---

# 7. Success Metrics

| Category   | Metric                                   | Target           |
| ---------- | ---------------------------------------- | ---------------- |
| Adoption   | Registered users — month 1               | 1,000            |
| Adoption   | Registered users — year 1                | 10,000           |
| Engagement | DAU/MAU ratio                            | ≥ 20%            |
| Engagement | Median cards per active user             | ≥ 10             |
| Trading    | Completed trades / month (by month 6)    | ≥ 500            |
| Retention  | D30 retention                            | > 30%            |

---

# 8. Risks & Roadmap

## 8.1 Technical & Business Risks

| Risk | Impact | Mitigation |
| ---- | ------ | ---------- |
| Pokémon TCG API no SLA / rate limits / goes away | No catalog | Cache full catalog in Postgres; abstract source behind sync layer; evaluate fallback provider. |
| Cardmarket API access gated / commercial terms | No pricing | Confirm access early; if blocked, ship MVP without value-tracking (graceful degrade). |
| Trade fraud (off-platform shipping, no escrow) | User trust loss | Two-sided confirm; report flow; reputation/feedback as fast-follow; clear "trade at own risk" terms. |
| Abuse in messaging / profiles | Safety, legal | Report → moderation queue; rate limits; block-user. |
| GDPR non-compliance | Legal | Data export/delete from day 1; RLS; EU residency. |
| Notification fan-out cost/latency at scale | Engagement drop | Batch via Edge Functions; per-category opt-out. |

## 8.2 Phased Rollout

**MVP (v1.0)** — Auth, Profile, Collection, Card DB (Pokémon), Wishlist, Trading (trust model), Messaging, Notifications.

**v1.1** — Reputation/feedback on trades; market value tracking (price graph, history, alerts); richer search; abuse tooling hardening.

**v2.0** — Additional TCGs; AI card recognition (photo → identify; needs tool + eval strategy); marketplace exploration (buy/sell, payments, ratings); graded-card support (PSA/Beckett/CGC).

---

# 9. MVP Release Criteria

Launch when:

- Social login (Apple + Google) works end-to-end.
- Collection management works (add/edit/remove/quantity/condition/for_trade).
- Card database search + filters meet §4.4 AC.
- Wishlist works incl. public/private rules.
- Trading full lifecycle works incl. two-sided Completed.
- Real-time messaging + read receipts work.
- Notifications fire for all §4.8 events with per-category toggle.
- GDPR export/delete + RLS private-profile enforcement verified.
- Report/moderation path live.
- Security review completed.
- Beta test completed.
