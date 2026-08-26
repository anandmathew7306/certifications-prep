# EX380 Revision Sheet — ch01s07: OIDC Authentication & Group Claims
**Chapter 1: Authentication and Identity Management**

---

## 1. Why OIDC (vs LDAP/htpasswd)

| IdP type | Identity lives... | OpenShift's role |
|---|---|---|
| htpasswd | Secret inside cluster | IS the identity store |
| LDAP | External directory | Queries directly (bind) every login |
| OIDC | External IdP (Keycloak, Entra, Google) | Trusts a signed token — no password ever touches OpenShift |

**Why it matters at enterprise scale:** plugs into an org's existing SSO/MFA fabric instead of OpenShift maintaining its own bind config. Red Hat build of Keycloak replaces Red Hat SSO (SSO = maintenance mode).

> 💡 **Mental model:** LDAP = OpenShift asks the librarian every time. OIDC = OpenShift checks a signed membership card the user already has.

---

## 2. JWT & Claims — Core Concept

Token = `header.payload.signature`

- **Claim** = a fact about the user, already stamped onto the token by the IdP (not a mapping keyword — the actual data).
- OpenShift's job = decide **which claim to trust for which purpose** (`claims:` block in OAuth CR).

| Claim | Meaning |
|---|---|
| `sub` | Subject identifier — **default identity claim** |
| `preferred_username` | Typically → OpenShift username |
| `email` | Email address |
| `name` | Display name |
| `groups` | Non-standard but critical — maps to OpenShift `Group` objects |

**Fallback rule:** claim lists are tried top-to-bottom; **first non-empty value wins**.
```yaml
name:
  - nickname
  - given_name
  - name
```

⚠️ **Gotcha:** `sub` is the identity claim **by default** — must override explicitly if task requires `email`/`preferred_username`.
⚠️ Identity object name = `<idp_name>:<claim_value>` (e.g. `oidc_provider_name:abc123`).

---

## 3. Login Flow (Identity Broker)

1. User hits login → OpenShift OAuth server shows IdP choices
2. Redirect to external IdP → user authenticates there (OpenShift not involved)
3. IdP redirects back with signed JWT
4. OpenShift validates token, creates/updates `User`, `Identity`, `Group` objects
5. Issues OpenShift OAuth access token

**Prerequisite (on IdP side, before touching OpenShift):**
Register OpenShift as a client in the IdP → get **Client ID** + **Client Secret** → set redirect URI:
```
https://oauth-openshift.apps.<cluster_name>.<cluster_domain>/oauth2callback/<idp_provider_name>
```

⚠️ **Gotcha:** `<idp_provider_name>` in redirect URI must **exactly match** `identityProviders[].name` in the OAuth CR. Mismatch = broken redirect.

---

## 4. Group Claims → RBAC

```
IdP group membership → groups claim in JWT → OpenShift Group object → RoleBinding → permissions
```

Bind roles to Groups, not individual users — group membership auto-syncs on every login as long as `groups` claim is present.

⚠️ **Gotcha:** If IdP can't send a `groups` claim, **no built-in OIDC fallback exists**.
- Unsupported community tool: `redhat-cop/group-sync-operator` (know the name, not installable in exam scope)
- Do NOT confuse with EX280's `oc adm groups sync` (LDAPSyncConfig) — that's LDAP-only and fully supported; **no equivalent exists for OIDC**.

---

## 5. Configuring OIDC IdP — Procedure & OAuth CR

**5-step procedure:**
1. Get Client ID + Client Secret from IdP
2. Create Secret (client secret) in `openshift-config`
3. Create ConfigMap (CA bundle, `ca.crt` key) in `openshift-config` — only if CA isn't already cluster-wide trusted
4. Write OAuth CR YAML
5. Apply it

**Commands:**
```bash
# Secret with client secret
oc create secret generic <secret_name> \
  --from-literal=clientSecret=<the_secret_value> \
  -n openshift-config

# ConfigMap with CA bundle (if needed)
oc create configmap <config_map_name> \
  --from-file=ca.crt=<path_to_ca.crt> \
  -n openshift-config

# Edit OAuth CR — APPEND, never replace the array
oc edit oauth cluster
```

**Full annotated OAuth CR:**
```yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster                    # always "cluster" — singleton object
spec:
  identityProviders:
  - name: oidc_provider_name        # must match redirect URI segment
    mappingMethod: claim
    type: OpenID
    openID:
      clientID: oidc_clientid
      clientSecret:
        name: secret_name           # Secret in openshift-config
      ca:
        name: config_map_name       # ConfigMap in openshift-config (optional)
      claims:
        preferredUsername:
          - preferred_username
          - email
        name:
          - nickname
          - given_name
          - name
        email:
          - custom_email_claim
          - email
        groups:
          - groups
      issuer: https://external_idp_url.com   # HTTPS only
```

⚠️ **Gotchas (high exam-trap potential):**
- `metadata.name` is always `cluster` — never a new object, always edit/patch.
- `identityProviders` array **must never be emptied** — appending an OIDC entry must not wipe existing htpasswd/LDAP entries, or those IdPs become unusable.
- `issuer` must be HTTPS — no HTTP allowed.
- Secret + ConfigMap both live in `openshift-config` (same pattern as LDAP bind creds from EX280).

---

## 6. Logging In

| ROPC supported by IdP? | Behavior |
|---|---|
| Yes | `oc login -u <user> -p <pass>` works directly |
| No | Fails with `You must obtain an API token` |

**Workaround if ROPC not supported:**
1. Log in via web console through the OIDC tile
2. Help → Command line tools → Copy login command (reveals token)
3. ```bash
   oc login --token=<access_token> --server=<api_url>
   ```

⚠️ `You must obtain an API token` is **expected behavior**, not a misconfiguration — don't debug the OAuth CR for this.
(Requesting token via REST API = out of scope for this course.)

---

## 7. Mapping Methods

Shared across all IdP types (not OIDC-only):

| Value | Behavior |
|---|---|
| `claim` | Creates/reuses identity↔User mapping using claim value as username |
| `add` | Same, but tolerates same username already existing from a different IdP — adds identity to that user instead of erroring |

⚠️ If task = "avoid conflicts when same username exists across IdPs" → use `add`, not `claim`.

---

## 8. OAuth Access Tokens — Stale Token Gotcha

```bash
# List your own tokens (any IdP)
oc get useroauthaccesstokens
```

**Core gotcha:** OpenShift does **not** auto-sync when the IdP changes a user (removed, group changed, etc.). A logged-in user keeps old permissions until:
- token expires, or
- forced logout via token deletion

**Force logout / revoke access:**
```bash
oc delete oauthaccesstoken $(oc get oauthaccesstoken -o \
  jsonpath='{.items[?(@.userName=="username")].metadata.name}')
```

⚠️ **Gotchas:**
- `oauthaccesstoken` (cluster-scoped, all users) vs `useroauthaccesstokens` (your own) — easy to confuse.
- "Remove user access" tasks require **both** IdP-side removal **and** token deletion — IdP removal alone does not revoke an active session.
- Applies to **any** IdP, but especially relevant to OIDC since claims (esp. `groups`) can change frequently server-side without OpenShift knowing.

---

## Quick Command Reference

```bash
# Create secret + CM for OIDC integration
oc create secret generic <secret_name> --from-literal=clientSecret=<value> -n openshift-config
oc create configmap <config_map_name> --from-file=ca.crt=<path> -n openshift-config

# Edit OAuth CR
oc edit oauth cluster

# Login with token (if ROPC unsupported)
oc login --token=<access_token> --server=<api_url>

# List own tokens
oc get useroauthaccesstokens

# Force logout / revoke a specific user's access
oc delete oauthaccesstoken $(oc get oauthaccesstoken -o \
  jsonpath='{.items[?(@.userName=="username")].metadata.name}')
```