# Spec: Public Health-Check API Endpoint (ping)

## Problem Statement

There is no unauthenticated endpoint a load balancer, uptime monitor, or CI pipeline can call to verify the bwh_hive app is alive and running. Adding a minimal `ping` endpoint satisfies this need without touching any DocType, database schema, or authentication flow.

## Intended Behavior

`GET /api/method/bwh_hive.bwh_hive.api.ping` (no auth required) returns HTTP 200 with a JSON body:

```json
{
  "message": {
    "app": "bwh_hive",
    "version": "0.0.1",
    "time": "2026-07-13T14:32:01.123456"
  }
}
```

- `app` — hard-coded string `"bwh_hive"`.
- `version` — the installed app version read from `bwh_hive.__version__` (currently `"0.0.1"` in `bwh_hive/__init__.py:1`).
- `time` — current server datetime in ISO 8601 format (`frappe.utils.now()` returns a datetime string; convert/format to ISO with `datetime.fromisoformat(...).isoformat()` or use `frappe.utils.now_datetime().isoformat()`).

## Concrete Changes Required

### 1. `bwh_hive/bwh_hive/api.py`

Append one function after the existing imports (no new imports needed beyond what is already present — `frappe` is already imported):

```python
@frappe.whitelist(allow_guest=True)
def ping():
    """Health-check endpoint. Returns app name, version, and current server time."""
    import bwh_hive
    return {
        "app": "bwh_hive",
        "version": bwh_hive.__version__,
        "time": frappe.utils.now_datetime().isoformat(),
    }
```

**File:** `bwh_hive/bwh_hive/api.py`  
**Change type:** append — no existing code modified.  
**New imports:** none at module level; `import bwh_hive` inside the function avoids any circular-import risk.

### 2. No other files change

| Area | Change |
|---|---|
| DocTypes / JSON fixtures | None |
| Database migrations | None |
| Frontend (React/TS) | None |
| hooks.py | None |
| tests | Optional — see Verification section |

## Edge Cases & Validation

| Scenario | Expected outcome |
|---|---|
| Unauthenticated request (no session cookie, no API key) | 200 OK — `allow_guest=True` bypasses Frappe's login gate |
| `bwh_hive.__version__` is absent or `None` | Would raise `AttributeError`; acceptable — `__init__.py` always defines it and a missing version signals a broken install that should surface as an error |
| `frappe.utils.now_datetime()` raises | Cannot happen in a live Frappe context; no defensive handling needed |
| POST / DELETE to the endpoint | Frappe returns 405 automatically — no `methods=` restriction needed (GET is the natural verb and callers should use GET) |
| Response envelope | Frappe wraps the return dict in `{"message": {...}}`; callers must unwrap via `.message` |

## Migration Concerns

None. This change adds a pure Python function to an existing module. No `bench migrate` is required.

## Verification Checklist

A reviewer can validate the implementation with the following steps:

1. **Syntax / import check**
   ```bash
   bench --site pms.localhost execute "import bwh_hive.bwh_hive.api"
   ```
   Should complete without errors.

2. **Unauthenticated curl (the primary requirement)**
   ```bash
   curl -s http://pms.localhost:8000/api/method/bwh_hive.bwh_hive.api.ping | python3 -m json.tool
   ```
   Expected: HTTP 200, body contains `message.app == "bwh_hive"`, `message.version` is a non-empty string, `message.time` is a parseable ISO datetime.

3. **Verify no auth is required**
   ```bash
   curl -s -o /dev/null -w "%{http_code}" \
     http://pms.localhost:8000/api/method/bwh_hive.bwh_hive.api.ping
   ```
   Must return `200`, not `403` or `401`.

4. **Version matches `__init__.py`**
   ```bash
   bench --site pms.localhost execute \
     "import bwh_hive; print(bwh_hive.__version__)"
   ```
   Output must equal the `version` field returned by the endpoint.

5. **Time field is valid ISO 8601**
   ```bash
   curl -s http://pms.localhost:8000/api/method/bwh_hive.bwh_hive.api.ping \
     | python3 -c "import sys, json, datetime; d=json.load(sys.stdin); datetime.datetime.fromisoformat(d['message']['time']); print('OK')"
   ```
   Must print `OK` without raising.

6. **No regression on existing authenticated endpoints**
   Spot-check one authenticated endpoint (e.g. `get_my_dashboard`) still requires a valid session — the `allow_guest=True` flag is scoped to `ping` only and should not affect adjacent functions.
