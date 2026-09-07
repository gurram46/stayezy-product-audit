# Stayezy Recommendation System — Discovery Decisions — 2026-09-07

This note captures the business/product answers collected after the initial `CONTEXT.md`. It is intended to be merged into the long-lived context once implementation inspection begins.

## Booking and conversion truth

Stayezy has two booking modes:

1. **Instant booking** — availability/calendar is central; the user can proceed to payment/booking without waiting for host approval.
2. **Owner-approval booking** — the host must approve the request before the user proceeds to payment.

The recommendation north-star remains a **confirmed + paid Stayezy booking**, but event instrumentation must preserve the booking mode because the funnel and host-response dependency differ.

Suggested booking events/states to verify in code/DB:

- `booking_request_started`
- `booking_request_sent`
- `host_approved`
- `host_rejected`
- `host_approval_timeout`
- `payment_started`
- `payment_succeeded`
- `booking_confirmed`
- `stay_completed`

## Payment reality

Stayezy commonly collects an advance through Razorpay during the booking flow. The remaining amount may later be paid directly to the owner, or Stayezy may send another Razorpay payment link if the customer chooses to pay Stayezy.

Therefore do not model `payment_succeeded` as one undifferentiated event. Verify and track:

- amount displayed initially;
- advance amount;
- amount paid through Stayezy;
- remaining amount;
- final amount paid;
- whether balance went to owner or Stayezy;
- booking confirmation timestamp;
- stay completion timestamp.

## Staff-assisted / phone / WhatsApp outcomes

Staff can record structured outcomes after calls/WhatsApp. This should be implemented because a large share of current conversion activity is otherwise invisible to analytics.

Recommended outcome codes:

- `booked_in_app`
- `staff_assisted_booking`
- `owner_direct_off_platform`
- `lost_price`
- `lost_no_response`
- `lost_availability`
- `lost_trust`
- `lost_other`

The team is considering call recording + local speech-to-text models to extract service insights. Treat this as a separate analytics/operations project. It must be designed with appropriate user/staff notice, consent, retention and PII handling; recommendation V1 should not depend on it.

## Calendar semantics

Calendar dates may be:

- genuinely booked;
- blocked manually by the owner;
- blocked by Stayezy/admin.

These states must remain distinguishable. A blocked date is **not evidence of demand/popularity** and must not inflate a listing's trust or booking-history score.

Off-app / manually closed bookings are expected to be updated in the calendar either by the owner or Stayezy staff, but this process should be measured for freshness/compliance.

Potential state model:

- `available`
- `booked_stayezy`
- `booked_external`
- `blocked_owner`
- `blocked_admin`

## Search geography

When a user searches a locality such as Madhapur, explicit search intent overrides current GPS location.

Working geographic retrieval policy:

1. rank eligible Madhapur inventory first;
2. if inventory is insufficient, expand to nearby inventory;
3. use an approximately **2–5 km** expansion range;
4. nearby results should be distinguishable in the UI rather than silently mixed as if they were inside Madhapur.

Exact radius thresholds and expansion rules should be tested against inventory density.

## Pricing and negotiation

The displayed price may include negotiation room. Stayezy/owners have an internal floor/end price that should not be crossed. Customers frequently ask for discounts.

The data model should distinguish at minimum:

- `displayed_price`
- `quoted_price`
- `owner_floor_price` (internal/private)
- `final_booking_price`
- `discount_amount`
- `discount_source` / who approved it

The recommendation system must **never expose the owner floor price to customers**.

Price/value ranking should eventually use the real final-booking outcome rather than raw listed price alone.

## Reviews

Reviews are intended to come only from users who actually stayed. Treat reviews as verified-stay signals, subject to code/DB verification of the eligibility enforcement.

## Trust / negotiation questions users repeatedly ask

Current high-frequency friction/questions include:

- whether the calendar/date is genuinely available / synced;
- owner not responding quickly;
- price negotiation / "can you reduce the price?";
- whether the owner/property is trustworthy;
- security, including questions from women travellers;
- flexible check-in/check-out or extra time;
- increasing allowed guest/seating capacity beyond the listing's normal stated limit.

These are not merely ranking features. They identify **product-flow gaps** that should later be reduced through better property information, trust signals, structured request options, host SLAs and booking UX.

## Identical multi-unit inventory

The clarified duplicate example is: an owner may have multiple physically separate but effectively identical apartment units, sometimes using the same images/details.

Recommendation policy:

- group equivalent units into an `equivalence_group` / `inventory_cluster`;
- show only one representative unit from that cluster to a user in one recommendation/search exposure;
- do not allow five identical units to occupy five ranking positions;
- if the shown representative is booked/unavailable, select another eligible unit from the same cluster;
- retain the distinct unit/listing IDs behind the cluster for booking/inventory correctness.

This is closer to **multi-unit inventory pooling** than generic duplicate detection.

## Analytics implications

Mixpanel is expected/planned rather than trusted as existing production telemetry.

At minimum instrument separately:

- Home impressions and clicks;
- Search impressions and clicks;
- rank position;
- algorithm/config version;
- anonymous/session ID and login identity stitching;
- property views;
- favourite adds/removes;
- image/video engagement;
- chat start;
- host first response;
- staff takeover;
- structured staff outcome;
- booking mode (`instant` vs `approval_required`);
- booking request/approval states;
- payment states;
- booking confirmation;
- final booking price;
- stay completion;
- cancellations/refunds.

## Traffic / experimentation

Current traffic is roughly ~30 daily visitors with about ~2k downloads and paid Meta marketing expected to begin. Do not use online learning or continuously self-changing weights at this stage.

Use:

- deterministic/versioned ranking;
- stable control vs variant;
- longer evaluation windows;
- booking/paid-confirmation metrics as the north-star;
- CTR/favourites/chat as intermediate diagnostics;
- guardrails for support load, host concentration, payment failure and latency.

## External case studies

**Airbnb remains the primary analogue** because the inventory/booking/trust problem is structurally similar.

Swiggy and Zomato should be studied as **secondary ranking/personalization case studies**, particularly for:

- location-aware candidate retrieval;
- repeated-user personalization;
- popularity vs personalization;
- ranking diversity;
- sponsored/business-policy separation;
- experimentation;
- handling availability/operational reliability.

Do not copy food-delivery ranking objectives directly into Stayezy because booking value, decision latency, negotiation, host response and stay availability are materially different.

## Discovery status

Business/product discovery is now sufficient to move to **architecture + instrumentation design before code inspection**.

Remaining unknowns are mainly implementation facts to verify from the Stayezy codebase/database rather than additional founder-product questions:

1. exact booking/payment state machine and DB schema;
2. whether booking type is already represented cleanly;
3. current search geo representation (string vs canonical place/lat-long/radius);
4. current calendar state representation;
5. whether historical property views/impressions exist anywhere;
6. anonymous/session identity mechanism;
7. host-vs-staff identity in chat messages;
8. review eligibility enforcement;
9. existing fields for equivalent/multi-unit inventory;
10. current analytics/Mixpanel integration state.

No ranking weights are frozen yet.