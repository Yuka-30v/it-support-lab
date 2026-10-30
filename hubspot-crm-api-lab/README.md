# HubSpot CRM API Lab (Postman)

**Cases 10–14** | Continues from Cases 8–9 (Microsoft Defender lab)

## Goal

Use Postman to create, read and update contacts through the HubSpot CRM API, then troubleshoot real and deliberately triggered API errors the way a support engineer would: read the raw response, isolate the cause, and apply the smallest fix.

## Setup

| Item | Detail |
|---|---|
| CRM | HubSpot free CRM account, test data only (fake `example.com` contacts) |
| Authentication | HubSpot service key, sent as a Bearer token |
| Scopes | `crm.objects.contacts.read`, `crm.objects.contacts.write` only |
| Postman workspace | `hubspot-crm-lab` |
| Environment | `HubSpot Test` with `base_url`, `token`, `contact_id` |
| Collection auth | Bearer `{{token}}`, inherited by every request |

## Requests

| Request | Method | Endpoint | Result |
|---|---|---|---|
| List contacts | GET | `{{base_url}}/crm/v3/objects/contacts?limit=5` | 200 OK |
| Create Contact | POST | `{{base_url}}/crm/v3/objects/contacts` | 201 Created |
| Get contact | GET | `{{base_url}}/crm/v3/objects/contacts/{{contact_id}}` | 200 OK |
| Update contact | PATCH | `{{base_url}}/crm/v3/objects/contacts/{{contact_id}}` | 200 OK |

### Request chaining

Create Contact has a post-response script that tests for 201 and saves the new contact ID for the Get and Update requests:

```javascript
pm.test("Contact created (201)", () => {
    pm.response.to.have.status(201);
});

if (pm.response.code === 201) {
    pm.environment.set("contact_id", pm.response.json().id);
}
```

The `if` guard stops a failed request from overwriting `contact_id` with an empty value.

The request body uses `test{{$timestamp}}@example.com` so each run creates a unique email and avoids accidental duplicates.

---

## Case 10: 403 Forbidden on create (unplanned)

**Symptom:** POST to create a contact returned `403 Forbidden`.

**What I checked:** The same key returned `200 OK` on the List contacts GET request, so the key itself was valid and correctly sent. That ruled out authentication and pointed to permissions.

**Response:** The error stated that *one of* a list of write scopes was required.

**Root cause:** The service key had only the read scope.

**Fix:** Added `crm.objects.contacts.write` only, then resent: `201 Created`.

**Note on the AI suggestion:** Postman's built-in AI assistant described the token as OAuth 2.0, suggested re-authorizing the connection, and implied all listed scopes were needed (including sensitive-data scopes). I declined it. The key was not OAuth, and the raw error said only one scope was required. Applying only the needed scope follows the principle of least privilege.

## Case 11: 401 Unauthorized (deliberate)

**Action:** Added one extra character to the end of the token, then sent List contacts.

**Response:** `401 Unauthorized`, category `INVALID_AUTHENTICATION`, message "Authentication credentials not found".

**Observation:** The message refers to OAuth 2.0 even though a service key was in use. An error message is a clue, not a diagnosis.

**401 vs 403:**
- **401:** the API cannot identify you. Check, correct or reissue the key.
- **403:** the API knows who you are, but you lack permission. Add the missing scope.

**Restore:** Removed the extra character and confirmed List contacts returned `200 OK` again.

## Case 12: 409 Conflict (deliberate)

**Action:** Created a contact with an email that already existed.

**Response:** `409 Conflict`, category `CONFLICT`, message "Contact already exists. Existing ID: ...".

**Support takeaway:** The response includes the existing contact's ID, so the right guidance is to update that record (PATCH) instead of creating a new one.

## Case 13: 404 Not Found (deliberate)

**Action:** Requested a contact ID that does not exist (`123`).

**Response:** `404 Not Found`, returned as an **HTML page, not JSON**, with no `category` or `correlationId`.

**Observation:** Every other error in this lab came back as JSON. An integration that always parses errors as JSON could fail on this 404 with a parsing error that hides the real cause. This is worth checking when a customer reports that their integration crashes on lookups.

## Case 14: 400 Bad Request (deliberate)

**Action:** Misspelled a property name as `firstnam` in the create request body.

**Response:** `400 Bad Request`, error `PROPERTY_DOESNT_EXIST`, naming the exact property (`firstnam`).

**Note on the AI suggestion:** Postman's AI assistant suggested making the email unique. That was wrong: the email was already unique through `{{$timestamp}}`. The raw error identified the real cause directly.

---

## Key learnings

- **Prove what works first.** A successful GET narrowed the 403 down to permissions before any change was made.
- **Read the raw error before trusting a suggested fix.** The AI assistant misdiagnosed two of five errors.
- **`correlationId`** appears in JSON error responses. When escalating to a vendor, this ID lets them trace the exact request.
- **`hs_object_source: INTEGRATION`** is set on contacts created through the API, which helps answer the common CRM support question "Where did this record come from?"

## Security notes

- The token is not included in this repository. Only the collection is exported, never the environment.
- Next improvement: move the token into Postman Vault. Environment values sync to Postman's cloud, while Vault keeps secrets local.
