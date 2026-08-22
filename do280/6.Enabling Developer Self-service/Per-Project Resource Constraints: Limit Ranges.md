# Per-Project Resource Constraints: Limit Ranges

## Outline
- Objectives
- Managing Namespace Resources
- Limit Ranges
- Setting Maximum and Minimum Limit Ranges
- Setting Defaults
- Creating Limit Ranges

## Quick Summary
Covers LimitRange objects — namespaced resources that set default/min/max compute limits and requests for workloads — explaining why they're needed alongside ResourceQuotas, how they auto-apply defaults to containers, how min/max enforcement works, and key rules/gotchas when combining default/defaultRequest/min/max values.

## Key Concepts
- **Namespace resource quota gap** — quotas limit total namespace resource usage, but don't stop users from: (1) accidentally creating resource-heavy workloads that starve others, or (2) forgetting to set limits/requests (which, with a quota present, causes workload creation to **fail** outright).
- **LimitRange** — namespaced object defining resource limits for workloads within a namespace; addresses the above gaps.
- **Limit types** — Default limit (applied if not explicitly set), Default request (applied if not explicitly set), Maximum (upper bound for both requests and limits), Minimum (lower bound for both requests and limits), Limit-to-request ratio (max allowable ratio between limit and request — not covered in depth in this course).
- **Scope of LimitRange application** — applies to containers, pods, images, image streams, and persistent volume claims.
- **Maximum limit range** — prevents workloads from exceeding specified limits/requests, avoiding quota exhaustion; overly restrictive maximums can block legitimate workloads — allow flexibility.
- **Minimum limit range** — ensures workloads have sufficient resource requests/limits, preventing under-provisioning.
- **Defaults (default/defaultRequest)** — auto-apply limits/requests, streamlining workload creation in quota-enforced namespaces; especially useful for dynamic workloads (e.g. CI-generated pods) needing consistent resource allocation without manual config; harder to get right in namespaces with diverse workload types.
- **Admission controller enforcement** — the Kubernetes API server's admission controller enforces limit ranges; it affects **pod definitions directly**, not Deployments, StatefulSets, or other higher-level workload controllers (the pod created from those inherits the enforced values).
- **LimitRange does not affect existing pods** — only applies at pod creation time; to apply a limit range retroactively, the pod (e.g. via its deployment) must be deleted and recreated.
- **Key value ordering rules** — `max` ≥ `default` ≥ `defaultRequest` ≥ `min` (for CPU/memory keys).
- **Auto-derivation of default/defaultRequest** — if you specify `min`/`max` but omit `default`/`defaultRequest`, Kubernetes automatically adds `default`/`defaultRequest` values copied from `min`/`max` — for more predictable behavior, always specify `default`/`defaultRequest` explicitly when specifying `min`/`max`.
- **Conflicting limit ranges** — avoid creating conflicting LimitRanges in a namespace (e.g. two different default CPU values) — behavior becomes unclear/ambiguous.
- **Rollout interaction with quotas** — modifying a deployment creates a replacement ReplicaSet, but the old ReplicaSet's pods continue running until rollout completes; pods from **both** ReplicaSets count toward the namespace resource quota — if the new ReplicaSet alone satisfies the quota but the **combined** total exceeds it, the rollout **cannot complete**.

## Operators
(none named on this page)

## Commands
| Command | What it does | Key flags/options |
|---|---|---|
| `oc create deployment example --image=image` | Creates a deployment (may fail to create pods if quota + no limits/requests set) | `--image=` |
| `oc get event --sort-by .metadata.creationTimestamp` | Shows namespace events, useful to diagnose quota/limit-range failures | `--sort-by` |
| `oc describe pod NAME` | Shows a pod's actual applied limits/requests (e.g. from a LimitRange default) | — |
| `oc set resources deployment example --limits=cpu=new-cpu-limit` | Updates/replaces a deployment's CPU limit (or other resource specs) | `--limits=` |

## Config / YAML Snippets
Basic LimitRange (memory defaults only):
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
  namespace: default
spec:
  limits:
    - default:
        memory: 512Mi         # Applied if a container doesn't set a memory limit
      defaultRequest:
        memory: 256Mi           # Applied if a container doesn't set a memory request
      type: Container
```

Namespace ResourceQuota requiring limits/requests:
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: example
  namespace: example
spec:
  hard:
    limits.cpu: "8"
    limits.memory: 8Gi
    requests.cpu: "4"
    requests.memory: 4Gi
```

Full LimitRange with all limit types:
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: example
  namespace: example
spec:
  limits:
  - default:
      cpu: 500m               # Default CPU limit if not specified
      memory: 512Mi             # Default memory limit if not specified
    defaultRequest:
      cpu: 250m                 # Default CPU request if not specified
      memory: 256Mi               # Default memory request if not specified
    max:
      cpu: "1"                   # Max allowed CPU (limit or request)
      memory: 1Gi                  # Max allowed memory (limit or request)
    min:
      cpu: 125m                    # Min allowed CPU (limit or request)
      memory: 128Mi                  # Min allowed memory (limit or request)
    type: Container
```

Resulting pod values (after LimitRange applied to a deployment with no explicit limits):
```
Containers:
  hello-world-nginx:
    Limits:
      cpu:     500m
      memory:  512Mi
    Requests:
      cpu:        250m
      memory:     256Mi
```

## Comparisons
| Aspect | ResourceQuota | LimitRange |
|---|---|---|
| Scope | Total combined usage across a namespace | Per-workload (container/pod) defaults and bounds |
| Enforces | Namespace-wide caps | Individual container/pod limit/request values |
| Effect on missing limits/requests | Blocks workload creation if quota requires them and they're unset | Auto-fills defaults so quota requirements are satisfied |
| Applies to | Aggregated namespace resources | Containers, pods, images, image streams, PVCs |

## Exam Tips / Gotchas
- **Order rule for CPU/memory keys**: `max` ≥ `default` ≥ `defaultRequest` ≥ `min`.
- If `min`/`max` are set but `default`/`defaultRequest` are omitted, Kubernetes **auto-derives** them from `min`/`max` — for predictable behavior, **always explicitly set `default`/`defaultRequest`** when using `min`/`max`.
- **LimitRange enforcement happens at the admission controller level and affects pod definitions directly** — Deployments, StatefulSets, and similar controllers themselves are unaffected (their spawned pods are what get the enforced values).
- **LimitRanges do NOT retroactively affect existing pods** — you must delete and recreate the pod (e.g. via its deployment) for a new/changed LimitRange to take effect.
- Requesting CPU/memory **outside** the min/max range results in **pod creation failure** with a specific warning (e.g. "maximum cpu usage per Container is 1, but limit is 1200m").
- **Do not create conflicting LimitRanges** in the same namespace (e.g. two different default CPU values) — the resulting behavior is unclear.
- During a deployment rollout, pods from **both the old and new ReplicaSets** count toward the namespace quota simultaneously — a rollout can get stuck/fail to complete if the combined total exceeds the quota, even if the new ReplicaSet alone would satisfy it.

## Glossary
- **LimitRange**: namespaced object defining default, minimum, maximum, and limit-to-request ratio constraints for workloads.
- **Admission controller**: Kubernetes API server component that enforces LimitRange values on pod creation.
- **default**: LimitRange key specifying the default resource limit applied if unset.
- **defaultRequest**: LimitRange key specifying the default resource request applied if unset.
- **max / min**: LimitRange keys specifying upper/lower bounds for both requests and limits.
- **Limit-to-request ratio**: LimitRange key enforcing a maximum allowed ratio between a resource's limit and request (not covered in depth).