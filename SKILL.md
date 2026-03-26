---
name: strong-app
description: >
  Interact with the Strong v6 workout tracker REST API — login, list exercises,
  fetch workout logs and templates, manage folders, tags, measurements, and
  widgets. Use when the user asks about their Strong app workouts, exercises, or
  training data.
version: 1.0.1
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

Unofficial REST API for the Strong workout tracking app (v6+). All API
interaction goes through the CLI dispatcher at `scripts/strong_runner.py`.

> **Warning:** This API is reverse-engineered and unofficial. Use at your own
> risk.

## Environment Variables

| Variable | Description |
|---|---|
| `STRONG_USERNAME` | Strong account username or email |
| `STRONG_PASSWORD` | Strong account password |

Both must be set before running any command. The CLI reads them automatically.

## Runner

All commands are invoked via:

```bash
python3 scripts/strong_runner.py <command> [--param value ...]
```

Every command authenticates automatically (login is handled internally), prints
JSON to stdout, and exits. No manual token management is needed.

## Workflow

1. Pick the appropriate subcommand from the list below.
2. Run it — authentication and token handling happen inside the runner.
3. All collection responses use **HAL** format with `_links`, `_embedded`, and `total`.

---

## 1. Login

Authenticate and return tokens. Rarely needed directly — all other commands log in automatically.

```bash
python3 scripts/strong_runner.py login
```

**Output:** `{ "accessToken": "eyJ...", "refreshToken": "kf3Z...", "userId": "uuid" }`

---

## 2. Refresh Token

Renew an expired access token without re-entering credentials.

```bash
python3 scripts/strong_runner.py refresh_token
```

**Output:** `{ "accessToken": "eyJ...", "refreshToken": "...", "expiresIn": 1200, "userId": "uuid" }`

---

## 3. Get User Profile

Fetch the authenticated user's profile, preferences, and purchases.

```bash
python3 scripts/strong_runner.py get_profile
```

**Response keys:** `id`, `created`, `lastChanged`, `username`, `email`, `emailVerified`, `name`, `goal`, `preferences`, `purchases`, `legacyPurchase`, `legacyGoals`, `startHistoryFromDate`, `firstWeekDay`, `availableLogins`, `migrated`.

---

## 4. List Exercises (Measurements)

Fetch all exercises in the user's library.

```bash
python3 scripts/strong_runner.py list_exercises
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
python3 scripts/strong_runner.py get_exercise --measurement_id <uuid>
```

| Parameter | Required | Description |
|---|---|---|
| `--measurement_id` | Yes | Exercise/measurement UUID |

---

## 6. List Workout Templates

Fetch all workout templates (routines).

```bash
python3 scripts/strong_runner.py list_templates
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
python3 scripts/strong_runner.py get_template --template_id <uuid>
```

| Parameter | Required | Description |
|---|---|---|
| `--template_id` | Yes | Template UUID |

**Response keys:** `id`, `created`, `lastChanged`, `name`, `access`, `logType`, `_embedded.cellSetGroup[]` (each with `id`, `cellSets[]`).

---

## 8. List Workout Logs

Fetch all completed workout sessions.

```bash
python3 scripts/strong_runner.py list_logs
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

Fetch a single workout log. Pass `--include_measurement` to embed exercise data.

```bash
python3 scripts/strong_runner.py get_log --log_id <uuid>
python3 scripts/strong_runner.py get_log --log_id <uuid> --include_measurement
```

| Parameter | Required | Description |
|---|---|---|
| `--log_id` | Yes | Log UUID |
| `--include_measurement` | No | Include exercise/measurement data in the response |

**Response keys:** `id`, `created`, `lastChanged`, `name`, `access`, `startDate`, `endDate`, `logType`, `_embedded.cellSetGroup[]` (each with `id`, `cellSets[]`, `_embedded.cellSet[]`).

---

## 10. List Folders

Fetch workout template folders.

```bash
python3 scripts/strong_runner.py list_folders
```

**Response (HAL):** `_embedded.folder[]` with keys: `id`, `created`, `lastChanged`, `name`, `index`.

---

## 11. List Tags

Fetch exercise tags/categories.

```bash
python3 scripts/strong_runner.py list_tags
```

**Response (HAL):** `_embedded.tag[]` with keys: `id`, `created`, `name`, `color`, `isGlobal`.

---

## 12. List Widgets

Fetch dashboard widget configuration.

```bash
python3 scripts/strong_runner.py list_widgets
```

**Response (HAL):** `_embedded.widget[]` with keys: `id`, `created`, `lastChanged`, `index`, `widgetType`, `parameters`.

---

## 13. Share a Template

Generate a shareable link for a workout template.

```bash
python3 scripts/strong_runner.py share_template --template_id <uuid>
```

| Parameter | Required | Description |
|---|---|---|
| `--template_id` | Yes | Template UUID |

---

## 14. Share a Workout Log

Generate a shareable link for a workout log.

```bash
python3 scripts/strong_runner.py share_log --log_id <uuid>
```

| Parameter | Required | Description |
|---|---|---|
| `--log_id` | Yes | Log UUID |

---

## 15. Get Shared Link

Retrieve a shared template or log by its link ID.

```bash
python3 scripts/strong_runner.py get_shared_link --link_id <id>
```

| Parameter | Required | Description |
|---|---|---|
| `--link_id` | Yes | Shared link ID |

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
