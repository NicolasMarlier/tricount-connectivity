# Tricount API notes

Field notes on the **unofficial** API of [Tricount](https://tricount.com) (by bunq), as used by the Android app.
They come from public reverse engineering (starting point: [elrandar/tricount-api](https://github.com/elrandar/tricount-api))
and were then confirmed against the live API by clients that read and write expenses.

> ⚠️ This is not an official, documented or supported API. It can change without notice, and using it may go
> against Tricount/bunq's terms. Use it at your own risk, keep the traffic low, and never act on behalf of people
> who did not agree to it.

These notes are language-agnostic: no library, only the protocol, the pitfalls and a [changelog](#changelog) of
what changed or broke over time. Contributions are welcome. See [Contributing](#contributing).

---

## Current values

| What | Value | Last checked |
|---|---|---|
| Host | `https://api.tricount.bunq.com` (prefix `/v1`) | 2026-11-21 |
| User-Agent | `com.bunq.tricount.android:RELEASE:14.6.0:14600:ANDROID:13:C` | 2026-11-21 |
| `device_description` | `Android` | 2026-10-30 |

User-Agent format: `com.bunq.tricount.android:RELEASE:<version>:<build>:ANDROID:<android version>:C`.
To find a recent version, look at the "Tricount by bunq" listing on Google Play or APKMirror.
**Make the User-Agent configurable without a code change** (e.g. an environment variable): old versions get rejected (see [Pitfalls](#pitfalls)).

## Protocol

### Headers

```
User-Agent: <see above>
Content-Type: application/json
X-Bunq-Client-Request-Id: <uuid>          # a fixed value and a random one per request both work
X-Bunq-Client-Authentication: <token>     # on every call except opening the session
app-id: <app_installation_uuid>           # optional
X-Bunq-Language / X-Bunq-Region: en_US    # optional
```

The app registers an RSA key, but **requests do not need to be signed**.

### Response envelope

Every item is wrapped in its type name:

```json
{ "Response": [ { "TypeName": { ... } }, ... ], "Pagination": { ... } }
```

Errors:

```json
{ "Error": [ { "error_description": "...", "error_description_translated": "..." } ] }
```

Translated messages can contain non-breaking spaces (`\u00a0`, `\u202f`). Normalize them before you display the message.

### 1. Anonymous session

```
POST /v1/session-registry-installation        (no auth header)
{
  "app_installation_uuid": "<random uuid>",
  "client_public_key": "<RSA 2048 public key, PEM>",
  "device_description": "Android"
}
→ Response: [ { "Token": { "token": "..." } }, { "UserPerson": { "id": 123 } }, ... ]
```

- No account and no API key: you register as an anonymous device.
- **Persist the installation** (uuid + private key) and reuse it. The same installation gets back the same user,
  with the Tricounts it already joined. You can also persist the token and user id and renew them on 401.
- The user object may come as `UserPerson`, and possibly as `UserApiKey`, `UserCompany` or `UserLight`. Accept them all.
- `device_description` may be shown as a device name. Use a neutral value (`Android`), not the name of your tool.
- **409** while opening a session: another session of the same installation is being opened at the same time.
  Wait briefly (1 s, then 2 s…) and retry a few times.
- **401** on an authenticated call: the token expired. Open a new session and retry once.

### 2. Read a Tricount (no need to join it)

```
GET /v1/user/{userId}/registry?public_identifier_token=<share token>
→ Response: [ { "Registry": { "id": 456, "title": "...", "currency": "EUR", "memberships": [ ... ] } } ]
```

- Share token = the last path segment of the share link. Forms seen in the wild: `https://tricount.com/AbCdEfGhIjk`,
  `tricount.com/fr/tAbC1234`, or a share message that contains the link.
  **The token does not always start with `t`.** A reasonable regex:
  `tricount\.com/(?:[a-z]{2}(?:-[a-z]{2})?/)?([A-Za-z0-9]{6,})`.
- `memberships[]` items look like:
  `{ "RegistryMembershipNonUser": { "uuid": "...", "status": "ACTIVE", "alias": { "display_name": "...", "pointer": { "name": "..." } } } }`.
  Skip `status == "DELETED"`.
- **Identify members by `uuid`, never by display name**: names can change, sometimes because of your own writes (see step 5).
- 404 means an invalid share token.

### 3. Join (needed to list entries and to write)

```
POST /v1/user/{userId}/registry-synchronization
{
  "all_registry_active":   [ { "public_identifier_token": "<share token>" } ],
  "all_registry_archived": [],
  "all_registry_deleted":  []
}
```

- Idempotent. Normally invisible to the group: no member is added.
- Defensive check: compare the members before and after joining. If one shows up, stop writing.
- A write that fails with **404 "Registry … not found"** means the join was lost. Join again and retry once.

### 4. List entries (joined Tricount)

```
GET /v1/user/{userId}/registry/{registryId}/registry-entry
→ Response: [ { "RegistryEntry": { ... } }, ... ], "Pagination": { "older_url": "/v1/user/.../registry-entry?older_id=..." }
```

- **Paginated**: follow `Pagination.older_url` (a full path) until it is empty or null.
- Deleted entries are still returned, with `"status": "INACTIVE"`.
- `type_transaction`: `NORMAL` (expense), `INCOME`, `BALANCE` (reimbursement between members: the payer
  `membership_owned` gives money to the member of the allocation).
- **Expense amounts are negative**: `"amount": { "value": "-12.50", "currency": "EUR" }`.
- Payer: `"membership_owned": { "RegistryMembershipNonUser": { "uuid": "...", "alias": { "display_name": "..." } } }`.
- Allocation: `membership_uuid` or `"membership": { "RegistryMembershipNonUser": { "uuid": "..." } }`, plus `amount`.
- Large Tricounts are slow: allow 15–20 s per page.

### 5. Write an entry

Order matters, and the calls must run **sequentially**:

```
1. POST registry-synchronization                                   (join, once)
2. PUT  /v1/user/{userId}/registry/{registryId}
        { "membership_uuid_active": "<member uuid>" }
3. POST /v1/user/{userId}/registry/{registryId}/registry-entry      (create)
   PUT  /v1/user/{userId}/registry/{registryId}/registry-entry/{id} (update, numeric id from step 4)
   DELETE …/registry-entry/{id}                                     (delete, unconfirmed)
```

- Step 2 binds the session to a member, who becomes the **creator** (`membership_created`) of the next entries.
  The Tricount notification shows that member as the author. Without step 2, the notification may name the wrong person.
  The binding applies to the **whole session**: never run two writes in parallel on the same session.
- ⚠️ Step 2 (or joining) can **rename** the member to the name of the bunq account linked to it
  (e.g. "Alex" became "Alex Smith"). Re-read the registry afterwards, warn if a name changed, and keep tracking members by uuid.

Accepted body:

```json
{
  "uuid": "<uuid generated by the client, so a retry does not create a duplicate>",
  "status": "ACTIVE",
  "type_transaction": "NORMAL",
  "description": "Dinner",
  "date": "2026-10-28 12:00:00.000000",
  "amount": { "value": "-24.00", "currency": "EUR" },
  "membership_uuid_owner": "<payer uuid>",
  "category": "UNCATEGORIZED",
  "attachment": [],
  "allocations": [
    { "membership_uuid": "<uuid>", "type": "AMOUNT", "amount": { "value": "-12.00", "currency": "EUR" } },
    { "membership_uuid": "<uuid>", "type": "AMOUNT", "amount": { "value": "-12.00", "currency": "EUR" } }
  ]
}
```

Rejected with **HTTP 400 "Superfluous field"**:
- a top-level `currency`;
- the **read** format: `membership_owned`, or `"membership": { "RegistryMembershipNonUser": ... }` in allocations.
  For writes, use `membership_uuid_owner` and `membership_uuid`.

The response to the POST contains `{ "Id": { "id": ... } }` and/or the created `RegistryEntry`.

## Pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| **422 "Please update your app"** | User-Agent version too old | Bump the version/build in the User-Agent. Keep it configurable. |
| **409** when opening a session | Two sessions of the same installation opened at once (parallel jobs) | Retry with a pause. Sync Tricounts one after the other. |
| **400 "Superfluous field …"** on write | Read format or top-level `currency` in the body | Use the write body above. |
| **404 "Route not found"** | Wrong path (guessed or outdated endpoint) | Stick to the paths above. |
| **404 "Registry … not found"** on write | Tricount not joined (or join lost) | Join, then retry once. |
| Notification names the wrong author | No `membership_uuid_active` before the write | Do step 2 before each write. |
| Member renamed after a write | Member linked to a bunq account | Track by uuid, re-read and warn. |
| Mixed-up authors | Parallel writes on one session | Serialize writes. |

Other notes:
- Equal split remainder: giving all leftover cents to the first member and spreading them one by one both work.
- Entry date: `00:00:00` and `12:00:00` both work. Noon avoids an accidental day shift when a time zone is applied.

## Implementation checklist

- [ ] User-Agent up to date and configurable without a code change
- [ ] `device_description: "Android"`
- [ ] Installation persisted and reused
- [ ] Retry on 409 when opening a session, renew on 401
- [ ] Members tracked by uuid
- [ ] Join before listing or writing, re-join once on 404 "Registry … not found"
- [ ] Guard: no new member appears after joining
- [ ] `membership_uuid_active` before each write, writes serialized
- [ ] Client-generated entry `uuid` (idempotent retries)
- [ ] Pagination followed when listing entries
- [ ] Readable errors: `error_description_translated` + raw payload kept for debugging

## Changelog

Newest first. Format: date · symptom → cause → fix.

- **2026-11-21** · Every call fails with **422 "Please update your app"** with User-Agent
  `com.bunq.tricount.android:RELEASE:7.0.7:3174:ANDROID:13:C` → the API rejects old app versions →
  User-Agent bumped to `14.6.0:14600`.
- **2026-11-02** · Write rejected with 400 because of a top-level `currency` → field removed.
  A member was renamed after `PUT membership_uuid_active` / join to the name of the bunq account linked to it
  → members tracked by uuid, registry re-read after the write, warning if a name changed.
- **2026-10-30** · Write → 404 "Registry … not found" → the Tricount had not been joined → join
  (`registry-synchronization`) before writing, re-join once on 404. `device_description` set to `Android`
  (a custom label may show up as a device name). Guessed endpoints gave 404 "Route not found".
- **2026-10-25** · Write → 400 "Superfluous field membership…" → the read format (`membership_owned`,
  `RegistryMembershipNonUser` objects) is rejected in writes → `membership_uuid_owner` / `membership_uuid`, plus
  `category` and `attachment`. The notification named another member as the author → `PUT registry
  { membership_uuid_active }` before writing.
- **2026-10-24** · 409 when opening sessions in parallel → retries with a pause, Tricounts synced sequentially.
- **2026-10-23** · Some share links rejected → share tokens do not always start with `t`.
  First working read flow: anonymous session → registry by share token → join → paginated entries.

## Contributing

When you hit a change or a new quirk:
1. Add a line at the top of the [Changelog](#changelog) (date · symptom → cause → fix).
2. Update the relevant section (Current values, Protocol, Pitfalls).
3. **Never include personal data**: no real tokens, share links, uuids, ids, names or amounts from a real Tricount.
   Use placeholders (`<uuid>`, `Alex`, `123`).
