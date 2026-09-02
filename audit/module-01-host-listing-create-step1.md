# Module 01 — Host Listing Creation: Step 1 Contract Failure

## Status
Confirmed production host-listing blocker.

## Reproduction
Host opens `https://stayezy.co/create-listing`, completes Step 1, selects a main photo (preview renders successfully), fills title/type/city/location/timing/description, then presses **Next**.

## Network evidence
The web client sends:

- `POST https://devnode.stayezy.co/api/properties`
- `Content-Type: application/json`
- Response: HTTP `422`

Relevant request body fields observed:

```json
{
  "banner_image": {},
  "booking_type": "Approval Booking",
  "category_id": "7",
  "category_name": "Service Apartments",
  "check_in": "01:00 PM",
  "check_out": "11:00 AM",
  "city": "7",
  "city_name": "Hyderabad",
  "description": "[QA test listing description]",
  "lat": "17.451368",
  "long": "78.3910332",
  "name": "QA Test Property - Do Not Book",
  "property_type": "New",
  "step_1": "add_property_name",
  "user_firechat": "3664",
  "user_id": "3664"
}
```

Server response:

```json
{
  "status": false,
  "message": "banner_image must be a string",
  "data": null,
  "code": 422
}
```

## Finding
The UI accepts and previews a selected main photo, but the request serializes `banner_image` as an object (`{}`) while the API validates it as a string. This blocks progression beyond Step 1 of property creation.

### Most likely contract failure
One of the following is happening:

1. the frontend keeps the selected `File`/object in state and serializes it into JSON, producing `{}`;
2. a required upload-to-storage step is not executed before the create-property request;
3. the frontend and backend disagree on whether `banner_image` should be a file upload, storage key, or URL.

The screenshots are sufficient to confirm the contract mismatch, but source inspection is still required to assign the exact implementation root cause.

## Severity
**P1 product blocker** for the host onboarding/listing flow because a host can complete the visible form but cannot proceed.

## Security-adjacent observation
The create-property JSON also contains client-supplied `user_id` and `user_firechat` values. Their presence is not itself a vulnerability. Backend authorization must be verified to ensure ownership is derived from the authenticated principal rather than trusting mutable client IDs. This should be tested only with controlled test accounts.

## Recommended code review targets
- Step 1 image selection state
- upload handler / object-storage upload
- request serializer for `/api/properties`
- backend validation/schema for `banner_image`
- backend ownership binding for `user_id` / `user_firechat`

## Evidence handling
Screenshots captured from browser DevTools show request URL/status, JSON request payload, and 422 response. No authorization headers or cookies should be committed.
