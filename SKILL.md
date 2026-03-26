---
name: strong-app
description: >
  Interact with the Strong v6 workout tracker REST API — login, list exercises,
  fetch workout logs and templates, manage folders, tags, measurements, and
  widgets. Use when the user asks about their Strong app workouts, exercises, or
  training data.
version: 1.0.0
homepage: https://github.com/dmzoneill/strongapp-api
metadata:
  clawdbot:
    requires:
      env:
        - STRONG_USERNAME
        - STRONG_PASSWORD
      bins:
        - python3
    primaryEnv: STRONG_USERNAME
    emoji: "\U0001F4AA"
    files:
      - "scripts/*"
---

# Strong Workout Tracker (v6 API)

Unofficial REST API for the Strong workout tracking app (v6+). The backend is at
`https://back.strong.app` behind Azure Front Door.

> **Warning:** This API is reverse-engineered and unofficial. Use at your own
> risk.

## Environment Variables

| Variable | Description |
|---|---|
| `STRONG_USERNAME` | Strong account username or email |
| `STRONG_PASSWORD` | Strong account password |

## Common Headers

Every request requires these headers to pass Azure Front Door:

```bash
STRONG_BASE="https://back.strong.app"

COMMON_HEADERS=(
  -H "Content-Type: application/json"
  -H "User-Agent: okhttp/4.12.0"
  -H "X-Client-Platform: android"
  -H "X-Client-Build: 1118"
)
```

After login, add the Bearer token to every subsequent request:

```bash
AUTH_HEADER=(-H "Authorization: Bearer ${ACCESS_TOKEN}")
```

## Workflow

1. **Login** to obtain `ACCESS_TOKEN`, `REFRESH_TOKEN`, and `USER_ID`.
2. Use `Authorization: Bearer <ACCESS_TOKEN>` on all subsequent calls.
3. Tokens expire in 1200 seconds (20 min). Use the **refresh** endpoint to renew.
4. All collection responses use **HAL** format with `_links`, `_embedded`, and `total`.

---

## 1. Login

Authenticate and obtain a JWT access token.

```bash
LOGIN_RESPONSE=$(curl -s -X POST "${STRONG_BASE}/auth/login" \
  "${COMMON_HEADERS[@]}" \
  -d "{\"login\":\"${STRONG_USERNAME}\",\"password\":\"${STRONG_PASSWORD}\",\"usernameOrEmail\":\"${STRONG_USERNAME}\"}")

ACCESS_TOKEN=$(echo "${LOGIN_RESPONSE}" | jq -r '.accessToken')
REFRESH_TOKEN=$(echo "${LOGIN_RESPONSE}" | jq -r '.refreshToken')
USER_ID=$(echo "${LOGIN_RESPONSE}" | jq -r '.userId')
echo "User: ${USER_ID}  Token expires in: $(echo "${LOGIN_RESPONSE}" | jq '.expiresIn')s"
```

**Response:** `{ "accessToken": "eyJ...", "refreshToken": "kf3Z...", "expiresIn": 1200, "userId": "uuid" }`

---

## 2. Refresh Token

Renew an expired access token without re-entering credentials.

```bash
curl -s -X POST "${STRONG_BASE}/auth/login/refresh" \
  "${COMMON_HEADERS[@]}" "${AUTH_HEADER[@]}" \
  -d "{\"accessToken\":\"${ACCESS_TOKEN}\",\"refreshToken\":\"${REFRESH_TOKEN}\"}" \
  | jq '.'
```

**Response:** `{ "accessToken": "eyJ...", "refreshToken": "...", "expiresIn": 1200, "userId": "uuid" }`

---

## 3. Get User Profile

Fetch the authenticated user's profile, preferences, and purchases.

```bash
curl -s "${STRONG_BASE}/api/users/${USER_ID}" \
  "${COMMON_HEADERS[@]}" "${AUTH_HEADER[@]}" \
  | jq '{id, username, email, name, goal, firstWeekDay, preferences: .preferences | keys}'
```

**Response keys:** `id`, `created`, `lastChanged`, `username`, `email`, `emailVerified`, `name`, `goal`, `preferences`, `purchases`, `legacyPurchase`, `legacyGoals`, `startHistoryFromDate`, `firstWeekDay`, `availableLogins`, `migrated`.

---

## 4. List Exercises (Measurements)

Fetch all exercises in the user's library.

```bash
curl -s "${STRONG_BASE}/api/users/${USER_ID}/measurements" \
  "${COMMON_HEADERS[@]}" "${AUTH_HEADER[@]}" \
  | jq '{total, exercises: [._embedded.measurement[] | {id, name: .name.custom, type: .measurementType}]}'
```

**Response (HAL):**
```json
{
  "_links": { "self": {...} },
  "_embedded": {
    "measurement": [
      {
        "id": "uuid",
        "created": "ISO8601",
        "lastChanged": "ISO8601",
        "name": { "custom": "Bench Press (Barbell)" },
        "instructions": { "custom": "..." },
        "media": [],
        "cellTypeConfigs": [{ "cellType": "...", "mandatory": true }],
        "measurementType": "EXERCISE"
      }
    ]
  },
  "total": 360
}
```

---

## 5. Get Single Exercise

```bash
curl -s "${STRONG_BASE}/api/users/${USER_ID}/measurements/MEASUREMENT_ID" \
  "${COMMON_HEADERS[@]}" "${AUTH_HEADER[@]}" \
  | jq '.'
```

---

## 6. List Workout Templates

Fetch all workout templates (routines).

```bash
curl -s "${STRONG_BASE}/api/users/${USER_ID}/templates" \
  "${COMMON_HEADERS[@]}" "${AUTH_HEADER[@]}" \
  | jq '{total, templates: [._embedded.template[] | {id, name: .name.custom, logType, access}]}'
```

**Response (HAL):**
```json
{
  "_embedded": {
    "template": [
      {
        "id": "uuid",
        "created": "ISO8601",
        "lastChanged": "ISO8601",
        "name": { "custom": "Push Day" },
        "access": "PRIVATE",
        "logType": "TEMPLATE",
        "_embedded": {
          "cellSetGroup": [
            { "id": "uuid", "cellSets": [...] }
          ]
        }
      }
    ]
  },
  "total": 75
}
```

---

## 7. Get Single Template

```bash
curl -s "${STRONG_BASE}/api/users/${USER_ID}/templates/TEMPLATE_ID" \
  "${COMMON_HEADERS[@]}" "${AUTH_HEADER[@]}" \
  | jq '.'
```

**Response keys:** `id`, `created`, `lastChanged`, `name`, `access`, `logType`, `_embedded.cellSetGroup[]` (each with `id`, `cellSets[]`).

---

## 8. List Workout Logs

Fetch all completed workout sessions.

```bash
curl -s "${STRONG_BASE}/api/users/${USER_ID}/logs" \
  "${COMMON_HEADERS[@]}" "${AUTH_HEADER[@]}" \
  | jq '{total, logs: [._embedded.log[] | {id, name: .name.custom, startDate, endDate, logType}]}'
```

**Response (HAL):**
```json
{
  "_embedded": {
    "log": [
      {
        "id": "uuid",
        "created": "ISO8601",
        "lastChanged": "ISO8601",
        "name": { "custom": "Push Day" },
        "access": "PUBLIC",
        "startDate": "ISO8601",
        "endDate": "ISO8601",
        "logType": "WORKOUT",
        "_embedded": {
          "cellSetGroup": [
            { "id": "uuid", "cellSets": [...] }
          ]
        }
      }
    ]
  },
  "total": 552
}
```

---

## 9. Get Single Log

Include measurement (exercise) data with `?include=measurement`.

```bash
curl -s "${STRONG_BASE}/api/users/${USER_ID}/logs/LOG_ID?include=measurement" \
  "${COMMON_HEADERS[@]}" "${AUTH_HEADER[@]}" \
  | jq '.'
```

**Response keys:** `id`, `created`, `lastChanged`, `name`, `access`, `startDate`, `endDate`, `logType`, `_embedded.cellSetGroup[]` (each with `id`, `cellSets[]`, `_embedded.cellSet[]`).

---

## 10. List Folders

Fetch workout template folders.

```bash
curl -s "${STRONG_BASE}/api/users/${USER_ID}/folders" \
  "${COMMON_HEADERS[@]}" "${AUTH_HEADER[@]}" \
  | jq '{total, folders: [._embedded.folder[] | {id, name: .name, index}]}'
```

**Response (HAL):** `_embedded.folder[]` with keys: `id`, `created`, `lastChanged`, `name`, `index`.

---

## 11. List Tags

Fetch exercise tags/categories.

```bash
curl -s "${STRONG_BASE}/api/users/${USER_ID}/tags" \
  "${COMMON_HEADERS[@]}" "${AUTH_HEADER[@]}" \
  | jq '[._embedded.tag[] | {id, name: .name.en, color, isGlobal}]'
```

**Response (HAL):** `_embedded.tag[]` with keys: `id`, `created`, `name`, `color`, `isGlobal`.

---

## 12. List Widgets

Fetch dashboard widget configuration.

```bash
curl -s "${STRONG_BASE}/api/users/${USER_ID}/widgets" \
  "${COMMON_HEADERS[@]}" "${AUTH_HEADER[@]}" \
  | jq '[._embedded.widget[] | {id, index, widgetType, parameters}]'
```

**Response (HAL):** `_embedded.widget[]` with keys: `id`, `created`, `lastChanged`, `index`, `widgetType`, `parameters`.

---

## 13. Share a Template

Generate a shareable link for a workout template.

```bash
curl -s -X POST "${STRONG_BASE}/api/users/${USER_ID}/templates/TEMPLATE_ID/link" \
  "${COMMON_HEADERS[@]}" "${AUTH_HEADER[@]}" \
  | jq '.'
```

---

## 14. Share a Workout Log

Generate a shareable link for a workout log.

```bash
curl -s -X POST "${STRONG_BASE}/api/users/${USER_ID}/logs/LOG_ID/link" \
  "${COMMON_HEADERS[@]}" "${AUTH_HEADER[@]}" \
  | jq '.'
```

---

## 15. Get Shared Link

Retrieve a shared template or log by its link ID.

```bash
curl -s "${STRONG_BASE}/api/links/LINK_ID" \
  "${COMMON_HEADERS[@]}" "${AUTH_HEADER[@]}" \
  | jq '.'
```

---

## External endpoints

| Endpoint | Purpose |
|---|---|
| `https://back.strong.app/auth/login` | Authenticate and obtain JWT tokens |
| `https://back.strong.app/auth/login/refresh` | Refresh an expired access token |
| `https://back.strong.app/api/users/{userId}` | User profile |
| `https://back.strong.app/api/users/{userId}/measurements` | Exercises (measurements) |
| `https://back.strong.app/api/users/{userId}/templates` | Workout templates |
| `https://back.strong.app/api/users/{userId}/logs` | Workout logs |
| `https://back.strong.app/api/users/{userId}/folders` | Template folders |
| `https://back.strong.app/api/users/{userId}/tags` | Exercise tags |
| `https://back.strong.app/api/users/{userId}/widgets` | Dashboard widgets |
| `https://back.strong.app/api/links/{linkId}` | Shared links |

No other external endpoints are contacted.

---

## Security and privacy

- **Credentials** are read exclusively from `STRONG_USERNAME` and `STRONG_PASSWORD` environment variables and are never logged, cached, or written to disk.
- **Tokens** (JWT access and refresh) are held in memory only for the duration of a single invocation and are never persisted.
- All traffic uses **HTTPS only** to `back.strong.app`.
- The runner script (`scripts/strong_runner.py`) is pure Python using only the standard library — no shell expansion or interpolation of user input occurs.
- No local files are read or written.

---

## Model invocation note

The agent should invoke this skill whenever the user asks about their Strong app workouts,
exercises, templates, workout history, tags, folders, or training data. The agent should
call `login` implicitly before any data-fetching command and never expose raw tokens to the
user. Prefer `list_logs` and `list_templates` for broad queries, and `get_log --log_id ...`
or `get_template --template_id ...` for specific lookups.

---

## Trust statement

This skill is **unofficial** and reverse-engineered. It is not affiliated with, endorsed by,
or supported by Strong Fitness Ltd. The API surface may change without notice. Use at your
own risk. The author assumes no liability for data loss or account restrictions resulting
from use of this skill.
