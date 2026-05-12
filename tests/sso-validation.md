# SSO & Access Validation Tests

**Project:** Zero Trust Corporate System
**Author:** Asier Barranco
**Date:** 12/05/2026
**Version:** 1.0

---

## 1. Objective

Verify that the SSO integration between Nextcloud and Keycloak is fully operational, that MFA is enforced, that user provisioning from Active Directory works correctly, and that access controls are applied as expected.

---

## 2. Test Environment

| Component | URL |
|---|---|
| Nextcloud (via Nginx) | `https://abb-ztcs.com` |
| Keycloak | `https://abb-ztcs.com/auth/` |
| Mattermost | `https://abb-ztcs.com/mattermost/` |
| Local admin bypass | `https://abb-ztcs.com/login?direct=1` |

---

## 3. Test Results

### TC-07 — Keycloak realm is accessible

**From:** external (ubuntu-desktop)
**Command:**
```bash
curl -s -o /dev/null -w "%{http_code}\n" https://abb-ztcs.com/auth/realms/zerotrust
```
**Result:** `200`
**Status:** ✓ PASS — Keycloak realm `zerotrust` is reachable via Nginx reverse proxy over HTTPS.

---

### TC-08 — Nextcloud redirects to Keycloak SSO

**From:** browser (private window)
**Steps:**
1. Navigate to `https://abb-ztcs.com`
2. Observe redirect behaviour

**Result:** browser is automatically redirected to `https://abb-ztcs.com/auth/realms/zerotrust/protocol/saml` — the Keycloak login page is presented.

**Status:** ✓ PASS — SAML SSO redirect is active. Users cannot access Nextcloud without authenticating through Keycloak.

---

### TC-09 — Local admin bypass is available

**From:** external (ubuntu-desktop)
**Command:**
```bash
curl -s -o /dev/null -w "%{http_code}\n" "https://abb-ztcs.com/login?direct=1"
```
**Result:** `200`
**Status:** ✓ PASS — The direct login URL bypasses SAML and presents the standard Nextcloud login form, preserving local admin access.

---

### TC-10 — AD user authenticates via SSO with MFA

**From:** browser (private window)
**Steps:**
1. Navigate to `https://abb-ztcs.com`
2. Enter credentials: `alice.smith` / `ASzerotrust2026!`
3. Keycloak prompts for TOTP — enter 6-digit code from authenticator app
4. Browser redirects to Nextcloud dashboard

**Result:** `https://abb-ztcs.com/apps/dashboard/` — authenticated session for `alice.smith`

**Status:** ✓ PASS — Full SSO + MFA flow validated end-to-end. AD credentials accepted by Keycloak via LDAP, TOTP enforced, SAML assertion delivered to Nextcloud.

![alice.smith authenticated — Nextcloud dashboard](../media/sso-config-01-dashboard.png)

---

### TC-11 — User provisioned in Nextcloud from AD

**From:** `ztcs-services`
**Command:**
```bash
docker exec -u 33 -it nextcloud php occ user:list
```
**Result:**
```
- alice.smith: alice.smith
- nc_admin: nc_admin
```
**Status:** ✓ PASS — `alice.smith` was automatically provisioned in Nextcloud upon first SSO login. The local admin account `nc_admin` coexists independently.

---

### TC-12 — Unauthenticated access to Nextcloud is denied

**From:** external (ubuntu-desktop)
**Command:**
```bash
curl -s -o /dev/null -w "%{http_code}\n" https://abb-ztcs.com/apps/dashboard/
```
**Result:** `302` — redirect to Keycloak login

**Status:** ✓ PASS — Protected resources are not accessible without authentication.

---

### TC-13 — MFA cannot be bypassed via direct API call

**From:** external (ubuntu-desktop)
**Command:**
```bash
curl -s -X POST "https://abb-ztcs.com/auth/realms/zerotrust/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=password&client_id=account&username=bob.jones&password=BJzerotrust2026!"
```
**Result:**
```json
{"error":"unauthorized_client","error_description":"Client not allowed for direct access grants"}
```
**Status:** ✓ PASS — Direct access grants are disabled. Password-only token requests are rejected before MFA is evaluated.

---

## 4. Summary

| Test | Description | Result |
|---|---|---|
| TC-07 | Keycloak realm accessible via HTTPS | ✓ PASS |
| TC-08 | Nextcloud redirects to Keycloak SSO | ✓ PASS |
| TC-09 | Local admin bypass available | ✓ PASS |
| TC-10 | AD user authenticates via SSO + MFA | ✓ PASS |
| TC-11 | User auto-provisioned in Nextcloud | ✓ PASS |
| TC-12 | Unauthenticated access denied | ✓ PASS |
| TC-13 | MFA API bypass blocked | ✓ PASS |

**All SSO and access validation tests passed. The identity federation, SSO flow and MFA enforcement are fully operational.**