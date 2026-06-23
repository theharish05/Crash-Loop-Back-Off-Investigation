##  Incident Summary & Context

A core component of the payment infrastructure, `payment-service`, was caught in a continuous crash loop, preventing financial transaction processing.

### The Diagnostic Log Blueprint

During triage, the following telemetry data was extracted from the failing workloads:

* **Pod Status:** `payment-service-xxxxxxxxx-xxxxx   0/1   CrashLoopBackOff`
* **System Event Logs:** `Back-off restarting failed container`
* **Internal Application Logs:** `panic: dial tcp 10.20.0.15:5432 connection refused`

---

## 🔬 Root Cause Analysis (The Evaluation Matrix)

To resolve the incident, three potential infrastructure failure variants were evaluated at different layers of the OSI model. Below is the technical breakdown of how each issue manifests, how to identify it, and how it was systematically verified.

### 1. The DNS Issue (Layer 7 Failure)

* **What it means:** The application is searching for a domain name or internal service discovery abstraction that does not exist in the cluster's internal registry.
* **The Log Signature:** ```text
panic: dial tcp postgres-service:5432: nc: bad address 'postgres-service'
```

```


* **Root Cause:** The application is calling a service wrapper (`postgres-service`) before the Kubernetes Service manifest itself has been applied or registered into **CoreDNS**.

### 2. The Database / Network Issue (Layer 4 Failure)

* **What it means:** Name resolution is working perfectly (CoreDNS maps the name to a valid IP address), but the network packet arrives at a destination where no backend application process is actively listening on the requested port (`5432`).
* **The Log Signature:** ```text
panic: dial tcp 192.168.15.70:5432: Connection refused (port closed)
```

```


* **Root Cause:** **This matches our incident blueprint.** The database tier is either completely offline, scaled to zero replicas, or the network service is pointing to incorrect backend pod label selectors.

### 3. The Secret / Credential Issue (Layer 7 Application Rejection)

* **What it means:** The network bridge is completely healthy, DNS resolves perfectly, and the database application answers the phone. However, the connection is dropped because the application passed missing, mismatched, or corrupted credentials.
* **The Log Signature:** ```text
panic: pq: password authentication failed for user "postgres"
```

```


* **Root Cause:** Misconfigured Kubernetes `Secret` resources or out-of-sync environment variables passed to the pod.

---

##  Infrastructure Replication Manifests

To reproduce and fix these states deterministically without anti-patterns like hardcoded IP routing, the following production-grade manifests were deployed.

### 1. Payment Application (`payment-service.yaml`)

This deployment utilizes a continuous loop wrapper around a real network probe utility (`nc`) to continuously validate the database socket state.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: payment-service
  template:
    metadata:
      labels:
        app: payment-service
    spec:
      containers:
      - name: payment-app
        image: alpine
        command: ["sh", "-c"]
        args:
          - |
            echo "Initializing Payment Engine..."
            sleep 2
            while true; do
              if OUTPUT=$(nc -z -w 3 $DB_HOST 5432 2>&1); then
                echo "$(date) - Success! Connected to database. Processing transactions..."
              else
                ERROR_MSG=${OUTPUT:-"Connection refused (port closed or destination unreachable)"}
                echo "panic: dial tcp $DB_HOST:5432: $ERROR_MSG"
                exit 1
              fi
              sleep 5
            done
        env:
        - name: DB_HOST
          value: "postgres-service"

```

### 2. Database & Service Layer (`postgres-db.yaml`)

This manifest creates both the underlying database engine instance and the logical network abstraction anchor required by the cluster's CoreDNS layer.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-db
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres-prod
  template:
    metadata:
      labels:
        app: postgres-prod
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          value: prodpassword123
        - name: POSTGRES_DB
          value: payments
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
spec:
  selector:
    app: postgres-prod
  ports:
    - protocol: TCP
      port: 5432
      targetPort: 5432

```


