# DataTrue Personas & Persona Items Skill

Personas represent user identities used for **sensitive data detection** in DataTrue tests.
They hold a collection of persona items — individual data points (emails, passwords, IDs, etc.)
that DataTrue watches for in captured network requests.

## Object Hierarchy

```
Account
  └── Persona
        └── PersonaItem
```

A persona is linked to a Suite or Test to enable sensitive data detection for that suite/test.

---

## Personas

### List Personas

```
ListPersonas({
  accountId: "<account_id>",
  first: 50
})
```

Returns: `id`, `name`, `createdAt`, `personaItems.totalCount`

### Create a Persona

```
CreatePersona({
  newPersona: {
    accountId: "<account_id>",
    name: "Site User"
  }
})
```

Returns: `id`

### Update a Persona

```
UpdatePersona({
  id: "<persona_id>",
  set: {
    name: "Updated Name"
  }
})
```

### Delete a Persona

Uses a filter — always scope by `id` to avoid accidental bulk deletion:

```
DeletePersonas({
  where: { id: { eq: "<persona_id>" } }
})
```

Returns: integer count of deleted records. Note: a return value of `0` does not always
mean the deletion failed — verify by re-listing if in doubt.

### Link a Persona to a Suite

Personas are linked to suites via `UpdateSuite`:

```
UpdateSuite({
  id: "<suite_id>",
  set: { personaId: "<persona_id>" }
})
```

### Unlink a Persona from a Suite

Set `personaId` to `null`:

```
UpdateSuite({
  id: "<suite_id>",
  set: { personaId: null }
})
```

---

## Persona Items

Persona items are the individual data values that DataTrue detects in network traffic.

### Fields

| Field             | Description                                                        |
|-------------------|--------------------------------------------------------------------|
| `name`            | Human-readable label (e.g. "Email Address")                        |
| `category`        | One of: `"User Identification"`, `"Credentials"`, `"Device Identification"` |
| `value`           | The actual sensitive value to detect (e.g. "test@test.com")        |
| `itemType`        | Typically `"text"` for string values                               |
| `detectionEnabled`| Whether DataTrue actively scans for this value in requests         |
| `attributeName`   | Auto-generated snake_case version of `name` (read-only)            |

### List Persona Items

```
ListPersonaItems({
  personaId: "<persona_id>",
  first: 50
})
```

Returns full item details including `id`, `name`, `category`, `value`, `detectionEnabled`.

### Create a Persona Item

```
CreatePersonaItem({
  newPersonaItem: {
    personaId: "<persona_id>",
    name: "Email Address",
    category: "User Identification",
    value: "test@example.com"
  }
})
```

**Required fields**: `personaId`, `name`, `category`, `value`

**Category values** (case-sensitive, use exactly as shown):
- `"User Identification"` — emails, usernames, full names
- `"Credentials"` — passwords, API keys, tokens
- `"Device Identification"` — device IDs, mobile IDs, MAC addresses

### Update a Persona Item

```
UpdatePersonaItem({
  id: "<item_id>",
  set: {
    name: "New Name",
    value: "new_value",
    category: "Credentials",
    detectionEnabled: true
  }
})
```

### Delete a Persona Item

```
DeletePersonaItems({
  where: { id: { eq: "<item_id>" } }
})
```

To scope to a specific persona (safer):

```
DeletePersonaItems({
  where: {
    id: { eq: "<item_id>" },
    personaId: { eq: "<persona_id>" }
  }
})
```

Returns: integer count of deleted records. A return of `0` may be misleading —
verify with `ListPersonaItems` if uncertain.

---

## Common Workflows

### Create a persona with items

1. `CreatePersona` → get `id`
2. `CreatePersonaItem` for each data point (email, password, etc.)
3. Optionally link to a suite via `UpdateSuite({ set: { personaId: "..." } })`

### Remove a persona from a suite without deleting it

```
UpdateSuite({
  id: "<suite_id>",
  set: { personaId: null }
})
```

### Full cleanup — delete persona and all its items

Delete items first, then the persona:

```
DeletePersonaItems({ where: { personaId: { eq: "<persona_id>" } } })
DeletePersonas({ where: { id: { eq: "<persona_id>" } } })
```

---

## Gotchas

- **Category is case-sensitive**: Always use `"User Identification"`, `"Credentials"`,
  or `"Device Identification"` exactly. Wrong casing will cause an API error.
- **`attributeName` is read-only**: It is auto-derived from `name` (snake_case).
  Do not attempt to set it directly.
- **Delete returns 0 ≠ failure**: The `deletePersonaItems` and `deletePersonas`
  mutations return the count of deleted rows. A return of `0` can occur even when
  deletion succeeded in a prior call. Always verify state with a List call.
- **Persona deletion does not unlink suites**: If a persona is linked to a suite,
  deleting it may leave the suite with a broken reference. Unlink first with
  `UpdateSuite({ set: { personaId: null } })` before deleting the persona.