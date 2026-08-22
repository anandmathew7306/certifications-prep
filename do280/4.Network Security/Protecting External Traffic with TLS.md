# Protecting External Traffic with TLS

## Outline
- Objectives
- Accessing Applications from External Networks
- Encrypting Routes
- Securing Applications with Edge Routes
- Securing Applications with Passthrough Routes
- Securing Applications with Re-encrypt Routes

## Quick Summary
Covers how OpenShift Routes expose applications to external traffic, and the three TLS termination types — Edge, Passthrough, Re-encryption — explaining where encryption starts/ends and how to create secured routes with certificates.

## Key Concepts
- **Route** — RHOCP resource exposing applications to external networks via a unique, publicly accessible hostname; relies on a router plug-in to redirect traffic from a public IP to application pods.
- **Router-to-pod traffic path** — routers send requests directly to pods based on service configuration (for performance), accessing pods through the services network.
- **Unencrypted routes** — simplest to configure; require no keys/certificates.
- **Encrypted routes** — require configuration of keys and certificates; specify a TLS termination type.
- **Edge termination** — TLS terminates at the router before reaching pods; router serves the TLS certificate (configured, or OpenShift's own default if omitted); connections from router to endpoints (internal network) are **unencrypted**.
- **Passthrough termination** — encrypted traffic is sent straight to the destination pod with no TLS termination at the router; the application itself serves certificates and decrypts traffic; supports mutual authentication between application and client.
- **Re-encryption termination** — a variation on edge: router terminates TLS with one certificate, then **re-encrypts** the connection to the endpoint (which may use a different certificate); full path is encrypted, including across the internal network; router uses health checks to determine host authenticity.
- **Ingress Operator default certificate** — if `--key`/`--cert` are omitted when creating an edge route, the Ingress Operator provides a certificate from the internal CA; the route references a secret (not the certificate directly) containing this internally provided cert.
- **Network policies** — can help protect internal traffic between applications/projects (relevant since edge/passthrough leave certain internal segments unencrypted or app-managed).
- **OpenShift TLS secret objects** — recommended way to provide a certificate to an application for passthrough routes; exposed to the app via a mount point in the container.
- **Re-encrypt route certificates** — external-facing certificate uses a client-trusted FQDN (e.g. `my-app.example.com`); internal certificate uses an OpenShift-internal FQDN (e.g. `my-app.namespace.svc.cluster.local`).
- **service-ca controller** — provides the PKI to generate/sign service certificates for internal traffic; creates a secret populated with a signed cert and key, which a deployment can mount as a volume (using this controller in depth is out of scope).

## Operators
- **Ingress Operator** — provides a default TLS certificate from the internal CA for edge routes when no custom cert/key is specified.
- **service-ca controller** — generates and signs service certificates for internal (re-encrypt) TLS traffic; creates secrets containing signed cert/key for deployments to mount.

## Commands
| Command | What it does | Key flags/options |
|---|---|---|
| `oc create route edge --service api-frontend --hostname api.apps.acme.com --key api.key --cert api.crt` | Creates an encrypted edge route with a custom TLS certificate | `--service`, `--hostname`, `--key` (private key), `--cert` (signed certificate) |
| `oc get secrets/router-ca -n openshift-ingress-operator -o yaml` | Views the internal CA-provided certificate | `-n`, `-o yaml` |

## Comparisons
| Termination Type | TLS terminates at | Certificate served by | Router↔endpoint traffic |
|---|---|---|---|
| Edge | Router | Router (configured cert, or OpenShift default if omitted) | Unencrypted |
| Passthrough | Not terminated at router | Application itself | Encrypted (full path, client-to-app) |
| Re-encryption | Router (then re-encrypted) | Router (external cert) + endpoint (internal cert) | Encrypted (re-encrypted with a different cert) |

## Exam Tips / Gotchas
- `oc create route` subcommands (`edge`, `passthrough`, `reencrypt`) do **NOT support `--dry-run=server`** — the command creates the route on the cluster regardless of that flag; use **`--dry-run=client`** instead to generate a manifest without creating the resource.
- With **edge** termination, traffic between client and router is encrypted, but traffic between router and application pods is **unencrypted** — network policies can help secure that internal segment.
- **Passthrough** routes provide a more secure alternative to edge because the application itself exposes its TLS certificate — encrypting the full path end-to-end.
- **Re-encrypt** routes require **two separate certificates**: one for the external FQDN (client-trusted) and one for the internal OpenShift FQDN (`*.svc.cluster.local`).
- If `--key`/`--cert` are omitted for an edge route, the route does **not** reference the certificate directly — it references a **secret** containing the Ingress Operator's internally provided certificate.

## Glossary
- **TLS termination**: the point in the connection path where encrypted (TLS) traffic is decrypted.
- **Edge termination**: TLS terminates at the router.
- **Passthrough termination**: TLS is not terminated at the router; encrypted traffic passes straight through to the pod.
- **Re-encryption termination**: TLS terminates at the router, then the router re-encrypts traffic to the endpoint using a different certificate.
- **service-ca controller**: OpenShift component providing PKI for signing internal service certificates.
- **FQDN**: Fully Qualified Domain Name — used to identify certificate trust scope (external vs. internal cluster hostname).