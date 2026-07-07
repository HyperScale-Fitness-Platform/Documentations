# Gym Platform — Team Onboarding & Local Development Guide

This is the single source of truth for getting the local development environment
running and for how to build any service in this system. Read this before writing
code in any `gym-*-service` repo.

---

## 1. Repository map

```
gym-platform-docs
gym-platform-infra
gym-shared-libs
gym-api-gateway
gym-auth-service
gym-operations-service     # occupancy + booking + membership
gym-people-service         # profiles + progress + plans
gym-social-service         # community + chat + notifications
gym-commerce-service       # catalog + orders + payments
gym-ai-service             # AI features + chatbot
```

Every service repo follows the same internal layout (see §5). If you're starting a
new service, use **"Use this template"** on `gym-service-template` rather than
copy-pasting `gym-auth-service` by hand.

---

## 2. One-time machine setup

Install once, per developer machine:

```bash
# Docker (required by everything below)
# https://docs.docker.com/get-docker/

# kind — local Kubernetes cluster running inside Docker
brew install kind          # macOS
# or: go install sigs.k8s.io/kind@latest

# kubectl
brew install kubectl

# Helm (used for some infra components)
brew install helm

# Node.js 20+ (or your service's runtime — see that repo's README)
```

Verify:
```bash
docker version
kind version
kubectl version --client
```

---

## 3. Standing up the local cluster (do this once per day / after a reset)

All of this lives in `gym-platform-infra`. Clone it first.

```bash
cd gym-platform-infra

# 1. Create the cluster
kind create cluster --name gym-dev --config kind-config.yaml

# 2. Install the Kafka operator (Strimzi)
kubectl create namespace kafka
kubectl create -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka
kubectl wait kafka/gym-kafka-dev --for=condition=Ready --timeout=300s -n kafka || true

# 3. Deploy the Kafka cluster itself
kubectl apply -f kafka/kafka-cluster.yaml -n kafka

# 4. Create all topics (one file per topic, defined as code)
kubectl apply -f kafka/topics/ -n kafka

# 5. Confirm brokers are up
kubectl get pods -n kafka
```

You should see `gym-kafka-dev-kafka-0` and `gym-kafka-dev-zookeeper-0` in `Running` state.

**Sanity check Kafka is actually working:**
```bash
# open a producer shell inside the broker pod
kubectl -n kafka exec -it gym-kafka-dev-kafka-0 -- bin/kafka-console-producer.sh \
  --broker-list localhost:9092 --topic session.booked

# in another terminal, open a consumer shell
kubectl -n kafka exec -it gym-kafka-dev-kafka-0 -- bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 --topic session.booked --from-beginning
```
Type a message in the producer terminal — if it appears in the consumer terminal,
Kafka is working end to end.

**Resetting the cluster** (if something's broken and you want a clean slate):
```bash
kind delete cluster --name gym-dev
```
Then repeat steps 1–4 above.

---

## 4. Running a service locally against the cluster

Two modes, use whichever fits what you're doing:

**Mode A — service runs on your laptop, talks to Kafka in the cluster (fastest iteration)**
```bash
kubectl -n kafka port-forward svc/gym-kafka-dev-kafka-bootstrap 9092:9092
# then in your service repo:
npm run dev
```
Use this while actively writing code — instant reload, no image rebuild.

**Mode B — service runs inside the cluster as a real pod (closer to production)**
```bash
cd gym-<your-service>
docker build -t gym-<your-service>:dev .
kind load docker-image gym-<your-service>:dev --name gym-dev
kubectl apply -f k8s/deployment.yaml
kubectl port-forward svc/<your-service> <local-port>:<container-port>
```
Use this before opening a PR, to confirm it actually works the way it will in staging/prod.

---

## 5. Standard service repo layout

Every service must look like this — consistency here is what lets anyone on the
team jump into any repo without relearning structure:

```
gym-<name>-service/
├── src/
│   ├── index.ts              # entrypoint, health check, wires up routes
│   ├── routes/                # HTTP handlers
│   ├── events/
│   │   ├── producers.ts       # publishes this service's events
│   │   └── consumers.ts       # subscribes to events from other services
│   ├── db/                    # this service's OWN schema/migrations — never shared
│   └── config/
├── tests/
├── k8s/
│   └── deployment.yaml         # Deployment + Service + Secret for this service
├── .github/workflows/ci.yml
├── Dockerfile
├── .env.example
└── README.md
```

**Non-negotiable rules for every service:**

1. **No service calls another service's database directly.** Ever. If you need data owned by another service, either call its API or consume its published events.
2. **Every event you consume must be idempotent to process twice.** Check `saga_id`/event ID before acting — brokers redeliver.
3. **Every event you publish must be defined in `gym-platform-docs` first**, as a versioned JSON schema, before you write the producer code. If the event doesn't exist there yet, add it in a PR to that repo and get it reviewed before building against it.
4. **`/health` endpoint is mandatory** — the k8s readiness/liveness probes depend on it.
5. **Secrets never get committed.** Use `.env.example` with placeholder values; real values go in k8s Secrets (local) or AWS Secrets Manager (later, on EKS).

---

## 6. What to do when implementing a new service — step by step

1. **Read the relevant contract in `gym-platform-docs`** — what events does this service publish, what does it consume, what's its API shape. If it's not documented, write it there first, get a quick review, then proceed.
2. **Generate the repo from `gym-service-template`.**
3. **Add `gym-shared-libs` as a dependency** — don't hand-roll a Kafka client or JWT verification; use the shared wrapper so behavior stays consistent across services.
4. **Build the `/health` endpoint first.** Get it deployed and passing readiness checks in the cluster before writing any business logic — this confirms your Dockerfile, k8s manifest, and CI are all correct early, rather than debugging plumbing and business logic at the same time.
5. **Implement your own local persistence** (your service's database only).
6. **Implement consumers for events you depend on**, using the schemas from `gym-platform-docs`. Write them idempotently from the start — don't bolt this on later.
7. **Implement producers for events you own.**
8. **If this service participates in a saga** (e.g., booking, membership, payment), write down: the forward action, the compensating action, and what "in progress" state looks like to the rest of the system — before coding it. Put this in the repo's README under a `## Saga participation` heading.
9. **Write tests**: unit tests for business logic, and at least one integration test that publishes/consumes a real event against a local Kafka (or a testcontainers instance) — not just mocked.
10. **Open a PR.** CI must pass: lint, build, test, Docker image build.
11. **Deploy to the local cluster (Mode B above) and manually verify** the event flow with the console producer/consumer trick from §3 before considering it done.

---

## 7. Debugging checklist when something doesn't work

- `kubectl get pods -A` — is everything actually running?
- `kubectl logs <pod-name>` — check your service's own logs first
- `kubectl -n kafka get kafkatopics` — does your topic actually exist?
- `kubectl describe pod <pod-name>` — usually reveals image pull errors, crash loops, failed probes
- If a saga seems stuck: check each participating service's logs for the specific `saga_id`, in step order, to find where the chain broke

---

## 8. Roadmap note (for planning, not day-one work)

This local setup (kind + Strimzi) is designed to map directly onto AWS EKS later:
kind → EKS, and Strimzi's Kafka either stays self-managed on EKS or gets swapped
for AWS MSK. Topic definitions and application code do not need to change either
way. Don't build anything today that only works locally — always go through
Kubernetes primitives (Deployments, Services, Secrets) even in dev, so the jump
to EKS is a config change, not a rewrite.
