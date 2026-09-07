# Stayezy Recommendation System — Discovery Context

**Status:** Discovery / architecture only. **No ranking weights are frozen yet.**

## Product objective

Build a recommendation and ranking system for Stayezy that improves the probability of a **completed in-app booking**, not merely clicks or time spent browsing.

The current business problem is that a meaningful part of the customer-to-property matching and deal closure is handled by staff outside the app. That creates two problems:

1. the product does not own the complete conversion journey; and
2. the recommendation system does not receive reliable booking labels for all successful matches.

A transitional measurement plan may therefore need to record both `booking_completed_in_app` and an attributable `booking_closed_off_app` / staff-assisted conversion until the product moves more closure into the app.

## Whiteboard model currently being discussed

User split:

- **N — New user:** little or no behavioral history. Ranking must lean on query/context, listing quality, popularity, availability, price/value, location and exploration/diversity.
- **O — Old/returning user:** can additionally use historical behavior and preferences.

Inventory types currently called out:

- **F — Farmhouses**
- **A — Apartments / service apartments**
- **V — Villas**

Potential signals discussed so far:

- location / distance
- dates and calendar availability
- guest count
- property type/category
- price / value
- reviews and ratings
- amenities
- host/property response time
- calendar freshness / sync reliability
- property page views
- favourites / saves
- click-through rate
- booking starts and completed bookings
- cancellations (later, once trustworthy data is available)
- image/video quality and engagement

**Important:** Home-page CTR and search-result CTR must be tracked separately. They have different exposure, intent and position bias.

## Ranking principle

Do not directly start with a giant weighted formula such as `0.2 * CTR + 0.2 * favourites + ...` and call it an algorithm.

The intended architecture should separate:

1. **Eligibility / hard constraints** — location/search area, dates, availability, capacity, property rules, etc.
2. **Candidate generation** — retrieve a reasonable set of eligible properties.
3. **Ranking** — score candidates by estimated booking utility/relevance.
4. **Re-ranking / policy** — diversity, inventory balance, exploration, business rules and safety/quality constraints.
5. **Measurement** — log every exposure and downstream outcome.
6. **Experimentation** — A/B test model/weight changes before rollout.

The north-star ranking outcome should be close to **uncancelled completed in-app bookings**. CTR, property views and favourites are useful intermediate signals, but they must not become the final objective or the model can learn clickbait rather than bookable stays.

## Analytics / Mixpanel

Mixpanel appears to be the analytics platform being referred to in discussions. Treat it as **product analytics + experimentation/feature-flag infrastructure**, not as the recommendation engine itself.

Before implementation, confirm exactly what Stayezy currently sends to Mixpanel and whether event identity works across anonymous → logged-in users.

Minimum event families to verify/design:

- recommendation/search impression
- property impression with `surface`, `rank_position`, `algorithm_version`, `experiment_variant`
- property click
- property detail view
- image/video interactions
- favourite add/remove
- availability check
- chat/contact started
- booking started
- checkout/payment started
- booking completed in app
- staff-assisted/off-app closure attribution (temporary but important while this flow exists)
- cancellation/refund outcome when available

For every property impression, preserve enough context to reconstruct **what the user was shown**, where, at what rank, and under which algorithm version. Without impression logs, raw CTR is not trustworthy.

## Research notes — Airbnb

Airbnb's public recommendation/search material is useful as a reference, but Stayezy should not copy Airbnb's present-day ML stack before it has the data volume and clean labels to justify it.

Airbnb publicly lists factors such as guest search parameters, listing location/price/availability, image quality, reviews/ratings, listing type, guest engagement/popularity, host responsiveness/cancellation history, ease of booking, and guest history/preferences.

Airbnb Engineering describes production ranking around **booking probability**, with personalization from long-term booking/review/cancellation history plus short-term listing views, and controlled A/B testing against booking and guardrail metrics.

Their more advanced systems use learned representations / embeddings and multi-stage retrieval + ranking. That is a future direction for Stayezy, not a V1 requirement.

### About the referenced GitHub project

`DeveloperManisha/Airbnb-Recommendation-System` is a 2018 academic/student project using Airbnb NYC open data. Its recommendation module uses collaborative filtering based on inferred review ratings; other modules cover review sentiment and price prediction.

It is useful for understanding classic recommender concepts, but it is **not Airbnb production code or evidence of Airbnb's real production recommendation architecture**. Do not use it as the implementation blueprint.

## Questions that must be answered before freezing architecture or weights

### A. Business truth / conversion

1. What exactly counts as a successful conversion today: payment, booking confirmation, staff-confirmed deal, or check-in?
2. What percentage of successful bookings currently close fully inside the app vs WhatsApp/phone/staff/manual workflow?
3. Can every off-app closure be linked back to a Stayezy `user_id`, `property_id`, search/session and timestamp?
4. Why do customers leave the app to close: negotiation, trust, availability uncertainty, payment friction, staff intervention, host response, or something else?
5. Is the recommendation system allowed to optimize only conversion, or must it also enforce inventory/fairness/business priorities?

### B. Data / Mixpanel

6. Is Mixpanel actually installed in production mobile + web today? Which SDKs and environments?
7. What events and event properties already exist? Do we have property **impressions**, not just clicks/views?
8. Can anonymous history be merged correctly after login/signup?
9. Do we have a warehouse/database export of analytics events for training and audit, or only Mixpanel dashboards?
10. How much usable history exists: users, searches, impressions, clicks, favourites, booking starts and bookings?

### C. Search / recommendation surfaces

11. Which surfaces are we ranking separately: Home recommendations, search results, similar properties, favourites follow-up, map, notifications/email?
12. For search, what are hard filters vs soft preferences? Example: dates/availability/capacity should usually be hard constraints; amenities/property type may be hard or soft depending on user intent.
13. Are Home-page CTR and search-result CTR already distinguishable in telemetry?
14. Do we log `rank_position` and `algorithm_version` for every impression so we can correct for position/exposure bias?

### D. Listing / host truth

15. Is calendar availability trustworthy and real-time? What does "calendar sync" currently mean and how stale can it become?
16. What exactly is "response time": chat response, enquiry response, booking request response, or staff response?
17. Are reviews/ratings verified against completed stays?
18. Which amenity fields are structured and reliable vs free text?
19. What image/video metadata exists today: count, resolution, order, moderation, upload quality, engagement per asset?
20. Do we have host cancellations, guest cancellations, booking rejections and refund outcomes as structured data?

### E. Experimentation

21. Can users be deterministically assigned to A/B variants and remain in the same variant across sessions/devices?
22. What is the primary success metric for V1: in-app booking conversion, uncancelled bookings, booking value, or another metric?
23. What guardrails cannot regress: cancellation/refund rate, support contacts, latency, failed payments, host concentration, etc.?
24. What minimum sample size / experiment duration is realistic at Stayezy's current traffic? If traffic is low, weights cannot be changed every few days based on noisy CTR.

## Working recommendation for V1

Do **not** begin with deep learning or collaborative filtering.

Start with a deterministic, explainable ranker once the event instrumentation is trustworthy:

- hard eligibility filters first;
- normalized feature signals;
- explicit new-user vs returning-user handling;
- a versioned scoring configuration;
- impression/outcome logging;
- A/B-tested changes;
- periodic weight updates based on statistically credible booking outcomes, not automatic self-modification from raw clicks.

Only move to learned ranking / embeddings when Stayezy has enough clean impression-to-booking journey data to train and evaluate them.

## Reference material reviewed

- Airbnb Help: "Airbnb's recommendation systems" — https://www.airbnb.co.in/help/article/4083
- Airbnb Help/Resource Center: search-ranking factors and listing popularity/availability/host behavior.
- Airbnb Engineering: "Personalizing Airbnb search by learning from the guest journey".
- Airbnb Engineering: "Embedding-Based Retrieval for Airbnb Search".
- Airbnb Engineering: "Improving Search Ranking for Maps".
- Mixpanel product analytics / experimentation material.
- GitHub: https://github.com/DeveloperManisha/Airbnb-Recommendation-System

## Next step

Answer the discovery questions above before inspecting implementation code or freezing ranking weights. After that, produce:

1. event taxonomy + data contract;
2. metric tree;
3. V1 ranking formula and normalization rules;
4. high-level architecture;
5. A/B experiment design;
6. rollout and weight-revision policy.
