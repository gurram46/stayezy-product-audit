# Stayezy Recommendation System — Discovery Context

**Status:** Discovery / architecture only. **No ranking weights are frozen yet.**

## Product objective

Build a recommendation and ranking system for Stayezy that increases **confirmed, paid bookings completed inside Stayezy**, while reducing the current dependency on phone/WhatsApp/staff-assisted closure.

Clicks, property views, favourites, chats and enquiries are intermediate intent signals. They are not the north-star outcome.

The product problem is larger than ranking: today the app often acts as a property discovery/gallery layer, while trust-building, negotiation and deal closure frequently move to chat, phone or WhatsApp. The recommender therefore must be designed together with instrumentation and later booking-flow improvements.

## Current conversion reality — captured 2026-09-07

- A final conversion for the recommendation system means **booking confirmation + payment**.
- Rough working observation from the founder/team: of roughly 10 users engaging with listings, about **4 may directly book in-app without chat/enquiry**. This is a directional product observation, not yet an instrumented metric.
- Many users who like a listing first open chat to negotiate or ask about amenities/details.
- If the host does not respond, Stayezy staff may respond and provide a Stayezy contact number.
- If the host responds, the conversation may still move outside the app.
- Some hosts may attempt to take customers off-platform and collect the full payment directly.
- A large share of successful closures currently happens through staff calls / WhatsApp rather than the app.
- In-app chats are stored; calls/WhatsApp are not currently reliably attributable as product events.
- The team believes **trust** is a major reason users prefer human contact because Stayezy is still new in the market.

This means `off_app_closed` must **not automatically be treated as a positive recommendation label**. A ranker that rewards off-platform closures could accidentally learn to promote hosts/listings that leak customers away from Stayezy.

## Recommended outcome hierarchy

Primary product metric:

`paid_confirmed_in_app_booking_rate`

Secondary business/operational metrics:

- `staff_assisted_booking_rate`
- `off_platform_leakage_rate` when measurable
- `chat_to_in_app_booking_rate`
- `property_view_to_in_app_booking_rate`
- `search_to_in_app_booking_rate`
- support/staff interventions per booking
- net platform revenue per session / booking

Intermediate intent signals:

- property click / detail view
- favourite/save
- chat/enquiry started
- availability check
- booking started
- checkout/payment started

## User model

- **N — New/anonymous user:** little or no behavioral history. We may know device/current location and current search intent.
- **O — Returning/logged-in user:** can use historical searches, clicks, favourites, chats, bookings, locations, price bands, amenities and property-type affinity once tracking exists and policy allows it.

Current inventory categories called out:

- **F — Farmhouses**
- **A — Apartments / service apartments**
- **V — Villas**

The current direction is **not** to force a user into one category before discovery. Search should show the best eligible inventory in the requested locality, while filters allow the user to narrow to Farmhouse / Apartment / Villa or other attributes.

## Search / location principle

Current device location is context, not necessarily the user's travel intent.

If a user explicitly searches `Madhapur`, the search destination should override the user's current device location for candidate generation. Search should first retrieve eligible inventory in/around the requested area and then rank it rather than dumping every Hyderabad property into one list.

Exact locality/radius behavior is still unresolved and must be specified before implementation.

## Eligibility vs ranking

### Hard eligibility / candidate constraints

Likely hard constraints:

- searched locality / geographic area
- selected dates
- actual availability
- guest capacity
- property rules that make the stay impossible

### Ranking / soft signals under discussion

- price / value
- reviews / ratings
- amenities match
- property category/type affinity
- host first-response latency
- host response rate / ability to resolve enquiries
- calendar occupancy / booking history as a possible trust signal
- property views
- favourites
- surface-specific CTR
- booking starts / completions
- image/video presentation quality and engagement
- previous user behavior for returning users

**Important:** calendar availability and calendar occupancy are different concepts. Availability is an eligibility constraint. Historical occupancy/booked-date density might become a trust/popularity feature only if booked dates are verifiably real and cannot be trivially manipulated by owner blocks.

## Host response signal

The founder/team intends `response time` to mean how quickly and effectively the **host** responds to a user's in-app chat/enquiry and helps move the user toward closure.

Do not mix host response with Stayezy staff response. Instrument them separately.

Candidate fields later may include:

- `host_first_response_seconds`
- `host_response_rate`
- `host_chat_to_in_app_booking_rate`
- `staff_takeover_required`

## Amenities

Amenities are structured because Stayezy mandates amenity information during property onboarding. This makes amenity matching a viable V1 feature once the exact schema is inspected.

## Favourites

Favourites are currently relatively rare, and the founder considers a favourite a strong indicator that the user genuinely likes a property.

Treat favourites as **high-intent but sparse** positive signals. Do not over-weight raw favourite counts without normalizing for impressions, users, recency and listing age.

## Cancellations

Cancellations are reportedly very rare (directionally below ~1 per 100 bookings). Last-minute cancellations are generally not refundable, while some earlier cancellations may be handled differently.

Because cancellation volume is low and structured reasons are not yet verified, cancellation should be a guardrail / later feature rather than a major V1 ranking weight.

## Images / video

Properties contain exterior and interior media. The team believes media can help estimate perceived listing/property quality.

For V1, distinguish **listing presentation quality** from actual physical property quality. Prefer objective and explainable signals first:

- image count
- required room/area coverage
- resolution
- blur/exposure/technical quality
- media completeness
- video availability
- asset engagement (swipes, opens, watch time) once instrumented

Computer-vision embeddings or learned visual-quality models are a later option, not a V1 requirement.

## Duplicate / near-identical inventory suppression

A critical product requirement is to prevent one owner with many identical or near-identical listings from dominating a user's recommendation feed.

Working policy:

1. group truly identical / equivalent inventory into an **equivalence cluster**;
2. for one recommendation request/session, expose only the highest-ranked eligible representative from that cluster;
3. suppress the other equivalent listings from that same user exposure;
4. if the representative becomes booked/unavailable or is otherwise no longer eligible, promote the next eligible listing in the cluster;
5. preserve diversity so one owner cannot occupy many adjacent positions with effectively the same product.

This requires a reliable definition of `identical/equivalent`. Do **not** implement it merely as `same_owner_id` because an owner can legitimately have very different properties.

## Analytics / Mixpanel

Mixpanel is currently **not believed to be running in production yet**; the team is planning analytics work.

Treat Mixpanel as product analytics + experimentation/feature-flag infrastructure, not as the recommendation engine itself.

Minimum event families to design:

- anonymous/session identity established
- login/signup identity merge
- home recommendation impression
- search result impression
- property impression with `surface`, `rank_position`, `algorithm_version`, `experiment_variant`
- property click
- property detail view
- image swipe/open
- video play/watch duration
- favourite add/remove
- availability check
- chat started
- host first response
- staff takeover / staff response
- contact/call CTA used
- booking started
- checkout/payment started
- payment succeeded/failed
- booking confirmed
- booking cancelled/refunded when applicable
- attributable staff-assisted/off-app closure where operationally possible

**Home CTR and search CTR must never be combined into one raw number.** Their exposure, intent and position bias are different.

Without impression logging, CTR is not trustworthy because we do not know the denominator or rank exposure.

## Ranking architecture principle

Do not start with a giant formula such as `0.2 * CTR + 0.2 * favourites + ...` and call it the algorithm.

Use a staged system:

1. **Eligibility / hard constraints**
2. **Candidate generation**
3. **Duplicate/equivalence suppression**
4. **Ranking**
5. **Re-ranking / diversity / explicit business policy**
6. **Exposure logging**
7. **Outcome attribution**
8. **Experimentation**

At current traffic (roughly **~30 daily app visitors**, with ~2k downloads and paid marketing expected to begin later), Stayezy does **not** yet have enough traffic to justify an online-learning or deep-learning recommender. Start deterministic and explainable; instrument first.

Weight changes must be versioned and evaluated over meaningful windows. Do not automatically change weights every few days because a noisy CTR moved slightly.

## Business economics — needs explicit separation

The user reports two materially different economics:

- in-app bookings: roughly **10% + tax** platform take;
- some Stayezy-assisted closures may generate materially higher economics, described directionally as up to roughly **40%** in some cases.

This needs clarification before ranking optimization. The recommendation system should not secretly optimize whichever flow produces the highest short-term commission if the strategic product goal is to move users toward self-serve in-app booking and reduce staff dependency.

Keep **relevance/product ranking** separate from **business-policy re-ranking**.

## Research notes — Airbnb

Airbnb's public recommendation/search material is useful as a reference, but Stayezy should not copy Airbnb's current ML stack before it has the data volume and clean labels to justify it.

Airbnb publicly discusses guest search parameters, listing location/price/availability, image quality, reviews/ratings, listing type, guest engagement/popularity, host responsiveness/cancellation history, ease of booking, and guest history/preferences.

Airbnb Engineering describes production ranking around booking probability, personalization from both long-term and short-term guest behavior, and controlled A/B testing.

### Referenced GitHub project

`DeveloperManisha/Airbnb-Recommendation-System` is a 2018 academic/student project using Airbnb NYC open data. Its recommendation module uses collaborative filtering based on inferred review ratings; other modules cover review sentiment and price prediction.

It is useful for classic recommender concepts but is **not Airbnb production code** and must not be used as the implementation blueprint.

## Remaining discovery questions — answer before freezing V1 scoring

### Critical product / conversion

1. After payment, is a booking automatically confirmed, or does the host/staff still have to approve it? What exact state means `booking_confirmed` in the database?
2. When a host takes a customer directly off-platform, does Stayezy earn **zero**, and is that deal recorded anywhere? Separately, when Stayezy staff closes the booking by phone/WhatsApp, how is the ~40% figure calculated and where is that booking recorded?
3. Can staff add a simple structured outcome after a call/WhatsApp conversation: `booked_in_app`, `staff_assisted_booking`, `lost`, `host_leakage`, `no_response`, etc.? Without this, we cannot learn from most current conversions.
4. What are the top trust blockers we can change inside the product: verified badge, reviews, payment protection, refund policy, support guarantee, host verification, booking history/social proof, etc.?

### Search / geography

5. If the user searches `Madhapur`, should results be **strictly inside Madhapur**, or can the ranker expand to nearby areas (for example 2–5 km) when inventory is weak? Who decides that radius?
6. Does search currently store a canonical place/lat-long/radius, or only a location string?
7. Is price shown before booking the actual final payable price including taxes/fees, or does negotiation regularly change it?

### Availability / trust

8. Can owners block calendar dates manually for reasons other than a real Stayezy booking? If yes, calendar occupancy cannot safely be used as a trust score without distinguishing `booked` from `blocked`.
9. Are off-platform/WhatsApp bookings written back into the Stayezy calendar? If not, how often can the app show stale availability or permit double-booking?

### Duplicate inventory

10. What exactly are the owner's "5 identical properties"? Are they:
   - the same physical property duplicated under multiple listing IDs;
   - multiple identical units in the same building/project;
   - separate properties with almost the same photos/amenities/layout;
   - or an owner intentionally creating duplicate listings?
11. Does the database already have a project/building/group/parent-property identifier we can use, or would we need to create an `inventory_cluster_id` / `equivalence_group_id`?

### Identity / events

12. Is there currently any stable anonymous/session identifier before login? If session recording exists, which product provides it and can it be joined to `user_id` after login?
13. Which current DB tables record property views, chats, favourites, booking starts, payments and bookings? We need to inspect this before deciding whether historical data is usable.

### Host quality

14. Can chat messages identify whether the responder was the host or Stayezy staff? We need this to calculate true `host_first_response_time`.
15. Are reviews verified against completed bookings, and can owners/reviewers manipulate them?

### Experimentation / traffic

16. Once Meta ads start, what traffic target is expected? We need an approximate sessions/day or search sessions/day to decide realistic A/B-test duration.
17. Are we willing to keep one stable control ranking for multiple weeks while a variant runs, instead of changing weights continuously?

## Working V1 recommendation

Do **not** begin with collaborative filtering, deep learning or automatic online weight updates.

Start with:

- instrumentation and identity stitching;
- hard eligibility filters;
- duplicate/equivalence suppression;
- normalized explainable feature signals;
- explicit new-user vs returning-user handling;
- versioned scoring configuration;
- separate Home/Search surfaces;
- outcome attribution including staff-assisted paths;
- A/B-tested changes;
- periodic weight review based on credible in-app booking outcomes and guardrails.

Only move toward learned ranking / embeddings after Stayezy accumulates enough clean impression → interaction → paid booking journey data.

## Reference material reviewed

- Airbnb Help: `Airbnb's recommendation systems` — https://www.airbnb.co.in/help/article/4083
- Airbnb Engineering: `Personalizing Airbnb search by learning from the guest journey`
- Airbnb Engineering: `Embedding-Based Retrieval for Airbnb Search`
- Airbnb Engineering: `Improving Search Ranking for Maps`
- Mixpanel product analytics / experimentation material
- GitHub: https://github.com/DeveloperManisha/Airbnb-Recommendation-System

## Next step

Resolve the remaining critical questions, then inspect the current Stayezy data model / event availability before freezing:

1. event taxonomy + data contract;
2. metric tree;
3. V1 scoring features and normalization;
4. high-level recommendation architecture;
5. duplicate-inventory suppression design;
6. A/B experiment design;
7. rollout and weight-revision policy.
