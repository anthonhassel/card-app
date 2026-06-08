# PRD - card-app UI Pilot

## Version

| Field   | Value        |
| ------- | ------------ |
| Product | card-app     |
| Version | pilot-0.1    |
| Status  | Draft        |
| Author  | Product Team |
| Date    | 2026-06-08   |

---

# 1. Goal

Build a fully navigable UI prototype of card-app with zero backend dependencies. A tester should be able to walk through every major user journey end-to-end using hardcoded mock data.

**Done when**: one person can complete the journey "login → find a collector → propose a trade → exchange messages → confirm trade" without hitting a dead end, broken navigation, or placeholder screen.

**Not a goal**: data persistence, real API calls, real auth, performance, or production-ready code quality.

---

# 2. Tech Constraints

- React Native + Expo (same stack as full app)
- No Supabase, no external API calls
- All state in-memory (React state / context) — resets on reload
- Mock data imported from a local `/mock` folder (JSON + TypeScript)
- Navigation: React Navigation (stack + bottom tabs)

---

# 3. Navigation Structure

```
Root Stack
├── AuthScreen                  (initial screen)
└── MainTabs (Bottom Tab Bar)
    ├── Tab: Collection
    │   ├── CollectionScreen
    │   ├── AddCardScreen       (modal)
    │   └── CardDetailScreen
    ├── Tab: Discover
    │   ├── DiscoverScreen
    │   └── CollectorProfileScreen
    ├── Tab: Trades
    │   ├── TradesScreen
    │   ├── TradeDetailScreen
    │   └── TradeBuilderScreen  (modal)
    ├── Tab: Messages
    │   ├── InboxScreen
    │   └── ConversationScreen
    └── Tab: Profile
        ├── ProfileScreen
        ├── EditProfileScreen   (modal)
        ├── WishlistScreen
        └── SettingsScreen
```

Header (all tabs): bell icon → NotificationsScreen (stack modal)

---

# 4. Screen Inventory

Each screen lists: what it shows, what interactions work, and where each action navigates.

## 4.1 AuthScreen

**Shows**: app logo, "Sign in with Apple" button, "Sign in with Google" button, "Continue as Demo User" link (small, below buttons).

**Interactions**:
- All three options → navigate to MainTabs (no real auth, all use the same mock logged-in user)

---

## 4.2 CollectionScreen

**Shows**: logged-in user's collection. Cards grouped by name; each group shows condition rows (e.g. "Charizard — NM ×2 · Played ×1"). Sticky header shows total card count. Cards marked `for_trade` show a small "FT" badge. Locked cards show a lock icon.

**Interactions**:
- Tap card row → CardDetailScreen
- FAB "+" → AddCardScreen (modal)
- Pull to refresh → no-op (mock)

---

## 4.3 AddCardScreen (modal)

**Shows**: search field, results list of Pokémon cards from mock data. After selecting a card: condition picker (Mint / Near Mint / Excellent / Good / Played), quantity stepper, "Available for trade" toggle.

**Interactions**:
- Type in search → filters mock card list in real time
- Tap a card result → shows condition/quantity form
- "Add to Collection" → appends to in-memory collection, dismisses modal
- "Cancel" → dismisses without change

---

## 4.4 CardDetailScreen

**Shows**: card image, name, set, number, rarity, Cardmarket price (mock value). If card is in logged-in user's collection: condition(s) and quantity. "Add to Wishlist" button (if not already wishlisted).

**Interactions**:
- "Add to Wishlist" → adds to in-memory wishlist, button changes to "On Wishlist"
- Back → previous screen

---

## 4.5 DiscoverScreen

**Shows**: list of collectors ranked by matchmaking score ("3 cards match your wishlist"). Each row: avatar, display name, city/country, matchmaking score badge. Filter bar at top: country dropdown, city text field.

**Interactions**:
- Tap collector row → CollectorProfileScreen
- Change country/city filter → list filters in real time (mock)
- Tap matchmaking score badge → tooltip "3 of your wishlist cards are available for trade"

---

## 4.6 CollectorProfileScreen

**Shows**: avatar, display name, bio, country/city, reputation ("47 of 50 trades positive"). Two tabs: "For Trade" (their collection with `for_trade=true`) and "Wishlist". "Propose Trade" button (sticky footer). "Message" button (top right header).

**Interactions**:
- "Propose Trade" → TradeBuilderScreen (modal), pre-scoped to this collector
- "Message" → ConversationScreen with this collector
- Tap a card in "For Trade" tab → CardDetailScreen
- Back → DiscoverScreen

---

## 4.7 TradesScreen

**Shows**: segmented control: "Active" / "History". Active list shows trades with status badge (Pending, Countered, Accepted, Disputed). History shows Completed, Rejected, Cancelled, Expired. Each row: counterparty avatar + name, status, time ago, number of cards each side.

**Interactions**:
- Tap trade row → TradeDetailScreen
- FAB "+" → TradeBuilderScreen (modal, no pre-scoped collector)

---

## 4.8 TradeDetailScreen

**Shows**: status banner (colour-coded). Two columns: "You offer" / "They offer" — card images + names + conditions + quantities. Timeline of status changes with timestamps. Action buttons depending on status:

| Status    | Your buttons (if it's your turn)         |
| --------- | ---------------------------------------- |
| Pending   | Accept · Reject · Counter (receiver)     |
| Pending   | Cancel (sender)                          |
| Countered | Accept · Reject · Counter (other party)  |
| Accepted  | Confirm Received                         |
| Completed | Rate 👍 / 👎 (if not yet rated)          |
| Disputed  | Contact Support (mailto link)            |

Reputation rating shown for counterparty. Link to trade's message thread.

**Interactions**:
- Accept → status badge updates to Accepted in-memory, success toast
- Reject → status → Rejected, toast
- Cancel → status → Cancelled, toast
- Counter → TradeBuilderScreen (modal, pre-filled with current items, editable)
- Confirm Received → if both confirmed in mock state → Completed; else toast "Waiting for counterparty"
- Rate 👍/👎 → rating saved in-memory, buttons replaced with submitted rating
- "View Messages" → ConversationScreen for this trade thread

---

## 4.9 TradeBuilderScreen (modal)

**Shows**: two-panel layout.

- **Left panel — "Their cards"**: list of counterparty's `for_trade` cards (filtered from mock). Tap to toggle selection; selected cards appear in the offer summary.
- **Right panel — "Your cards"**: logged-in user's `for_trade` cards. Tap to toggle selection.
- **Offer summary strip** (bottom): "You offer X cards ↔ They offer Y cards". "Send Offer" button (disabled until ≥ 1 card on each side).

**Interactions**:
- Tap card on either side → toggles selection, updates summary strip
- "Send Offer" → creates in-memory trade with status `Pending`, navigates back to TradesScreen, success toast
- "Cancel" → dismisses modal

---

## 4.10 InboxScreen

**Shows**: list of DM conversations. Each row: avatar, display name, last message preview, time, unread badge.

**Interactions**:
- Tap conversation row → ConversationScreen
- (No new-conversation FAB in pilot — conversations are started from CollectorProfileScreen)

---

## 4.11 ConversationScreen

**Shows**: message bubbles (sent right, received left). Timestamp separators. Input field + send button at bottom. If conversation is linked to a trade: sticky banner at top "Re: trade with [Name]" → taps to TradeDetailScreen.

**Interactions**:
- Type message + send → appends bubble to in-memory thread, clears input
- Tap trade banner → TradeDetailScreen
- Back → InboxScreen

---

## 4.12 ProfileScreen (own profile)

**Shows**: avatar, display name, bio, country/city, reputation score. Row links: "My Wishlist", "Settings". "Edit Profile" button.

**Interactions**:
- "Edit Profile" → EditProfileScreen (modal)
- "My Wishlist" → WishlistScreen
- "Settings" → SettingsScreen

---

## 4.13 EditProfileScreen (modal)

**Shows**: editable fields — display name, bio (char counter ≤ 300), country, city. Avatar upload button (opens image picker, no upload in pilot — just updates local state with picked image). "Public profile" toggle.

**Interactions**:
- Edit any field → updates in-memory profile
- "Save" → dismisses, profile screen reflects changes
- "Cancel" → dismisses without change

---

## 4.14 WishlistScreen

**Shows**: wishlist cards grouped by priority (High / Medium / Low). Each row: card image thumbnail, name, set, priority badge. "Add to Wishlist" FAB.

**Interactions**:
- Tap card → CardDetailScreen
- FAB → AddCardScreen in wishlist mode (condition/quantity hidden; priority picker shown instead)
- Swipe left on row → "Remove" action (removes from in-memory wishlist)

---

## 4.15 SettingsScreen

**Shows**: notification toggles (one per category: New trade request, Counter-offer, Trade accepted, New message, Wishlist match, Confirmation reminder). "Delete Account" row (destructive, shows confirmation alert then resets to AuthScreen). "Privacy Policy" / "Terms" links (no-op in pilot).

**Interactions**:
- Toggles → update in-memory state (no real push effect)
- "Delete Account" → confirmation alert → reset app to AuthScreen

---

## 4.16 NotificationsScreen (modal, from bell icon)

**Shows**: list of mock notifications with icon, title, body, time ago. Unread notifications have a highlighted background.

**Interactions**:
- Tap notification → deep-link to relevant screen (trade, message, or collector profile depending on mock type)
- All notifications marked read on open

---

# 5. Mock Data Specification

Location: `/mock/` folder at project root.

## 5.1 Users

Three users — one logged-in, two others.

```
currentUser:  Alex Svensson, Stockholm, Sweden. 12 collection cards, 5 wishlist cards.
              Reputation: 8/10 positive trades.

traderSam:    Sam Lindqvist, Gothenburg, Sweden. Has 3 cards matching Alex's wishlist.
              Reputation: 22/24 positive trades.

traderMia:    Mia Karlsson, Malmö, Sweden. Has 1 card matching Alex's wishlist.
              Reputation: 5/5 positive trades.
```

## 5.2 Cards

20 Pokémon cards covering a variety of sets and rarities. Include at least:
- 5 cards in Alex's collection (mix of for_trade and not)
- 3 cards locked in an active trade
- Cards that appear in wishlists and in other collectors' for_trade lists (to demonstrate matchmaking)

## 5.3 Trades

Four pre-existing trades covering all interesting states:

| ID | Parties | Status | Notes |
| -- | ------- | ------ | ----- |
| T1 | Alex ↔ Sam | Pending | Alex is receiver — shows Accept/Reject/Counter buttons |
| T2 | Alex ↔ Mia | Countered | Alex is sender, Mia countered — shows Alex's turn |
| T3 | Alex ↔ Sam | Accepted | Both need to confirm; Sam already confirmed |
| T4 | Alex ↔ Mia | Completed | Both confirmed; rating not yet submitted by Alex |

## 5.4 Messages

Two pre-existing threads:
- Alex ↔ Sam: 4 messages, linked to T1
- Alex ↔ Mia: 2 messages, no trade link

## 5.5 Notifications

Five mock notifications:
- Sam sent you a trade offer (→ T1)
- Mia countered your offer (→ T2)
- New message from Sam (→ conversation)
- Wishlist match: Sam listed Charizard for trade (→ Sam's profile)
- Reminder: confirm you received cards from Mia (→ T3)

---

# 6. Out of Scope

- Real authentication (any login tap → demo user)
- Supabase or any network call
- Data persistence between app reloads
- Real Pokémon TCG card images (use placeholder images or a free public CDN like pokemontcg.io with no API key for basic card images)
- Push notifications
- Real-time message delivery
- GDPR export/delete
- Cardmarket pricing (show a hardcoded mock price)
- Avatar upload to storage (local image picker only)

---

# 7. Done Criteria

The pilot is complete when a tester can walk through all of the following without hitting a dead end:

- [ ] Launch app → tap "Continue as Demo User" → land on Collection tab
- [ ] View collection, see condition grouping and FT/lock badges
- [ ] Add a card via search → appears in collection
- [ ] Open Discover tab → see collectors ranked by matchmaking score → filter by city
- [ ] Open Sam's profile → browse his for-trade cards → tap "Propose Trade" → build an offer in Trade Builder → send offer → see it in Trades tab
- [ ] Open T1 (incoming offer from Sam) → Accept → see status change to Accepted
- [ ] Open T2 (countered offer) → Counter → modify items in Trade Builder → send counter
- [ ] Open T3 (accepted trade) → tap "Confirm Received" → see "Waiting for counterparty" toast
- [ ] Open T4 (completed trade) → rate Sam with 👍
- [ ] Open Messages tab → open conversation with Sam → send a message
- [ ] Open Profile tab → edit display name and bio → save → see changes on profile
- [ ] Open Wishlist → add a card → remove a card
- [ ] Open Settings → toggle a notification category
- [ ] Tap bell icon → open notifications → tap one → land on correct screen
