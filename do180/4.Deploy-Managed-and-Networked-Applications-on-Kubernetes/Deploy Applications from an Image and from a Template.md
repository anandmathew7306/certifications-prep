# Deploy Applications from an Image and from a Template

## Outline
- Objectives
- Deploying Applications
- Resources and Resource Definitions
- Managing Resources
- Common Resource Types and Their Uses
  - Templates
  - Pod
  - Deployments
  - Projects
  - Services
  - Persistent Volume Claims
  - Secrets
- Managing Resources from the Command Line
  - Imperative Resource Management
  - Declarative Resource Management
- Retrieving Resource Information

## Quick Summary
Covers how Kubernetes/RHOCP represent applications as a collection of resources (pods, deployments, projects, services, PVCs, secrets, templates), and the two ways to manage them from the CLI — imperative (`create`, `run`, `set`, `new-app` with an image) vs. declarative (`create -f`, `new-app` with a template/file/Git repo).

## Key Concepts
- **Resource** — a configuration piece for a cluster component; creating/deleting one is a *request* for eventual creation/deletion, not immediate.
- **Resource type** — represents a specific component type (e.g. pod); RHOCP includes default Kubernetes types plus its own; CRDs can add new types.
- **Template** — RHOCP-specific YAML manifest containing parameterized definitions of one or more resources; predefined templates live in the `openshift` namespace.
- **Pod** — smallest compute unit deployable/manageable in RHOCP; runs one or more containers sharing networking/storage.
- **Deployment** — describes intended state of an app component as a pod template; manages one or more replica sets.
- **Project** — a Kubernetes namespace with additional annotations; primary method for managing regular-user access to resources; must use RBAC; provides logical/organizational isolation (resources in one project can't access another by default).
- **Finalizer** — special metadata key telling Kubernetes to wait until a condition is met before fully deleting a resource.
- **Service** — configures internal pod-to-pod network communication; requests to the service name/port are rerouted to pods matched via label selectors.
- **Persistent Volume (PV)** — cluster-provisioned persistent storage; globally scoped object.
- **Persistent Volume Claim (PVC)** — namespaced request for PV resources without needing knowledge of underlying storage; once a PV is bound to a PVC, it cannot bind to additional PVCs (scopes the PV to a single namespace until the PVC is deleted).
- **Secret** — holds sensitive info (passwords, credentials, SSH keys, OAuth tokens); can be mounted via volume plug-in or used to declare env variables; supports multiple types (service account tokens, SSH keys, TLS certs).
- **Imperative command** — instructs the cluster what to do (e.g. `create`, `run`, `set`).
- **Declarative command** — defines the state the cluster should match (e.g. `create -f`, `new-app` with a template/manifest).
- **--dry-run=client** — generates an object definition without creating it in RHOCP (used with `-o yaml`/`-o json` to produce a manifest).
- **all argument** — shorthand for a predefined subset of common resource types; does **not** include every resource type.

## Operators
(none named on this page)

## Commands
| Command | What it does | Key flags/options |
|---|---|---|
| `oc delete deployment -l app=my-app` | Deletes deployments matching a label | `-l` = label selector |
| `oc get templates -n openshift` | Lists predefined templates in the openshift namespace | `-n` = namespace |
| `oc process -f mysql-template.yaml -o yaml` | Processes a template into resource definitions | `-o yaml` |
| `oc process -f mysql-template.yaml --parameters` | Lists a template's parameters | `--parameters` |
| `oc new-app --template mysql-persistent` | Creates an application and resources from a template | — |
| `oc create deployment my-app --image example.com/my-image:dev` | Imperatively creates a deployment from an image | `--image` |
| `oc set env deployment/my-app TEAM=red` | Sets an environment variable on a resource | — |
| `oc run example-pod --image=... --env GREETING='...' --port 8080` | Imperatively creates a pod | `--image`, `--env`, `--port` |
| `oc run example-pod --image=... --dry-run=client -o yaml` | Generates a YAML pod definition without creating it | `--dry-run=client`, `-o yaml`/`-o json` |
| `oc create -f my-app-deployment.yaml` | Declaratively creates resources from a manifest file | `-f` |
| `oc new-app --file=./example/my-app.yaml` | Declaratively creates resources from a YAML manifest | `--file=` |
| `oc new-app --template mysql-persistent --param MYSQL_USER=operator ...` | Creates resources from a template with overridden parameters | `--param` (repeatable) |
| `oc new-app --image example.com/my-app:dev` | Imperatively creates resources from a container image | `--image` |
| `oc new-app https://github.com/apache/httpd.git#2.4.56 --name httpd24` | Creates an application from a Git repository | `--name` |
| `oc api-resources --categories=all` | Shows which resources the `all` argument includes | `--categories=all` |
| `oc describe template mysql-ephemeral -n openshift` | Shows detailed info (parameters, objects) for a template | `-n` = namespace |

## Config / YAML Snippets
Pod definition:
```yaml
apiVersion: v1
kind: Pod                          # Resource kind is Pod
metadata:                          # App name, project, labels, annotations
  annotations: {}
  labels:
    deployment: docker-registry-1
  name: registry
  namespace: pod-registries
spec:                               # Application requirements: containers, env vars, volumes, network config
  containers:
  - env:
    - name: OPENSHIFT_CA_DATA
      value:
    image: openshift/origin-docker-registry:v0.6.2
    imagePullPolicy: IfNotPresent
    name: registry
    ports:
    - containerPort: 5000
      protocol: TCP
    resources: {}
    securityContext: {}
    volumeMounts:
    - mountPath: /registry
      name: registry-storage
  dnsPolicy: ClusterFirst
  imagePullSecrets:
  - name: default-dockercfg-at06w
  restartPolicy: Always
  serviceAccount: default
  volumes:
  - emptyDir: {}
    name: registry-storage
status:                             # Last condition: probe time, transition time, status true/false, etc.
  conditions: {}
```

Deployment definition:
```yaml
apiVersion: apps/v1
kind: Deployment                    # Resource kind is Deployment
metadata:
  name: hello-openshift               # Name of the deployment resource
spec:
  replicas: 1                          # Number of running instances
  selector:
    matchLabels:
      app: hello-openshift
  template:                            # Metadata, labels, container info for the deployment
    metadata:
      labels:
        app: hello-openshift
    spec:
      containers:
      - name: hello-openshift             # Container name
        image: openshift/hello-openshift:latest   # Image used to create the deployment
        ports:                             # Port config
        - containerPort: 80
```

Project definition:
```yaml
apiVersion: project.openshift.io/v1
kind: Project                       # Resource kind is Project
metadata:
  name: test                          # Name of the project
spec:
  finalizers:                          # Waits until a condition is met before full deletion
  - kubernetes
```

Service definition:
```yaml
apiVersion: v1
kind: Service                       # Resource kind is Service
metadata:
  name: docker-registry                # Name of the service
  namespace: test                       # Project where the service resides
spec:
  selector:
    app: MyApp                           # Selects pods with label app=MyApp as endpoints
  ports:
  - protocol: TCP                        # Protocol set to TCP
    port: 80                              # Port the service listens on
    targetPort: 9376                       # Port on backing pods that traffic is forwarded to
```

Persistent Volume Claim definition:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim         # Resource kind is PersistentVolumeClaim
metadata:
  name: mysql-pvc                      # Name of the claim
spec:
  accessModes:
    - ReadWriteOnce                      # Read/write and mount permission mode
  resources:
    requests:
      storage: 1Gi                        # Requested storage size
  storageClassName: nfs-storage           # Name of the required StorageClass
status: {}
```

Secret definition:
```yaml
apiVersion: v1
kind: Secret                        # Resource kind is Secret
metadata:
  name: example-secret                 # Name of the secret
  namespace: my-app                     # Project where the secret resides
type: Opaque                        # Type of secret
data:                                # Encoded string/data
  username: bXl1c2VyCg==
  password: bXlQQDU1Cg==
stringData:                          # Decoded (plaintext) string/data
  hostname: myapp.mydomain.com
  secret.properties: |
    property1=valueA
    property2=valueB
```

## Comparisons
| Aspect | Imperative Resource Management | Declarative Resource Management |
|---|---|---|
| Approach | Instructs the cluster what to do | Defines the intended state to match |
| Example commands | `oc create deployment ...`, `oc run ...`, `oc set env ...`, `oc new-app --image ...` | `oc create -f manifest.yaml`, `oc new-app --file=...`, `oc new-app --template ...` |
| Speed | Faster — no object definition file required | Requires authoring a manifest/template first |
| Versioning/incremental changes | Not well suited | Required for proper versioning |
| Typical use | Quick testing of a deployment | Production-grade, reproducible deployments |

## Exam Tips / Gotchas
- The `all` argument is **not** every resource type — it's shorthand for a predefined subset; verify coverage with `oc api-resources --categories=all`.
- `oc create -f manifest.yaml` is declarative even though it uses the `create` command — the declarative/imperative distinction depends on *how* you specify the resource (file vs. inline flags), not the command name alone.
- `oc new-app` can be used either imperatively (with `--image`) or declaratively (with `--file=`, a template, or a Git repo).
- Use `--dry-run=client` with `-o yaml`/`-o json` to generate a manifest from an imperative command without actually creating the resource — a common workflow: test imperatively, then generate the declarative definition.
- Once a PV is bound to a PVC, it **cannot** bind to additional PVCs — scopes it to a single namespace until that PVC is deleted.
- `oc describe` **cannot** generate structured (YAML/JSON) output — use `oc get` instead when output needs to be parsed/filtered (e.g. with JSONPath or Go templates).
- Resources in one project cannot access resources in another project by default.
- Commands run without specifying a namespace execute in the user's current namespace.

## Glossary
- **CRD**: Custom Resource Definition — used to add new resource types to Kubernetes/RHOCP.
- **Finalizer**: metadata key that delays resource deletion until a condition is met.
- **PV**: Persistent Volume — globally scoped cluster storage resource.
- **PVC**: Persistent Volume Claim — namespaced request for a PV.
- **Opaque (secret type)**: default/generic secret type for arbitrary key-value data.
- **Heuristics (new-app)**: automatic determination of which resource types to create based on given parameters/input source.