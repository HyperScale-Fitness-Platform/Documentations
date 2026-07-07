# Local Environment & Daily Workflow Guide

Purpose of this document: get every teammate from "empty laptop" to "actively
building a service that talks to the others through Kafka" — nothing more.
No sagas, no CQRS, no caching. Just environment, structure, and communication.

---

## PART 1 — One-time environment setup

Do this once per machine. (CentOS commands shown; swap package manager if on
another OS — the tools themselves are identical everywhere.)

### 1.1 Install Docker

```bash
sudo dnf remove -y docker docker-client docker-common docker-latest
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER
newgrp docker
docker run hello-world
```

### 1.2 Install kubectl

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/repodata/repomd.xml.key
EOF
sudo dnf install -y kubectl
kubectl version --client
```

### 1.3 Install kind (this is your local Kubernetes cluster)

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
kind version
```

### 1.4 Install Node.js (or your chosen runtime)

```bash
curl -fsSL https://rpm.nodesource.com/setup_20.x | sudo bash -
sudo dnf install -y nodejs
node -v
```

### 1.5 Install git, if missing

```bash
sudo dnf install -y git
```

At this point every teammate's machine can run containers, talk to a Kubernetes
cluster, and spin one up locally. That's all we need — no minikube, no
extra tools.

---

## PART 2 — Standing up the shared local cluster

This happens **once per person, on their own machine** — everyone runs their
own kind cluster locally. You are not sharing one cluster across the team;
each developer has an identical, disposable copy on their own laptop.

### 2.1 Create the cluster

```bash
kind create cluster --name gym-dev
kubectl cluster-info --context kind-gym-dev
```

If this succeeds, you have a working single-node Kubernetes cluster running
inside Docker on your machine.

### 2.2 Install Kafka using the Strimzi operator

Strimzi is what makes Kafka manageable on Kubernetes — you describe "I want a
Kafka cluster" and "I want a topic" as YAML files, and Strimzi's operator
actually creates and manages the underlying brokers for you.

```bash
kubectl create namespace kafka
kubectl create -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka
kubectl get pods -n kafka -w
```

Wait until you see a pod like `strimzi-cluster-operator-xxxxx` in `Running` state.
Press Ctrl+C to stop watching once it's ready.

### 2.3 Deploy the actual Kafka cluster

Save this as `kafka-cluster.yaml` (this lives in the `gym-platform-infra` repo,
everyone pulls the same file so all local clusters are identical):

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: gym-kafka-dev
  namespace: kafka
spec:
  kafka:
    version: 3.7.0
    replicas: 1
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
    config:
      offsets.topic.replication.factor: 1
      transaction.state.log.replication.factor: 1
      transaction.state.log.min.isr: 1
    storage:
      type: ephemeral
  zookeeper:
    replicas: 1
    storage:
      type: ephemeral
  entityOperator:
    topicOperator: {}
    userOperator: {}
```

```bash
kubectl apply -f kafka-cluster.yaml -n kafka
kubectl get pods -n kafka -w
```

Wait until you see `gym-kafka-dev-kafka-0` and `gym-kafka-dev-zookeeper-0`
running. This is your actual Kafka broker, running locally.

### 2.4 Create the topics every service will use

Topics are how services "talk" to each other — think of a topic as a named
channel that one service publishes to and any number of other services can
listen to, without knowing about each other directly.

Save each of these in `gym-platform-infra/kafka/topics/` (one file per topic):

```yaml
# session-booked-topic.yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: session.booked
  namespace: kafka
  labels:
    strimzi.io/cluster: gym-kafka-dev
spec:
  partitions: 3
  replicas: 1
```

Repeat this pattern for every topic you'll need starting out, for example:
- `session.booked`
- `payment.succeeded`
- `payment.failed`
- `order.placed`
- `session.confirmed`

```bash
kubectl apply -f kafka/topics/ -n kafka
kubectl get kafkatopics -n kafka
```

### 2.5 Confirm Kafka actually works, manually, before writing any code

```bash
# terminal 1 — act as a producer
kubectl -n kafka exec -it gym-kafka-dev-kafka-0 -- bin/kafka-console-producer.sh \
  --broker-list localhost:9092 --topic session.booked

# terminal 2 — act as a consumer
kubectl -n kafka exec -it gym-kafka-dev-kafka-0 -- bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 --topic session.booked --from-beginning
```

Type a line of text in terminal 1, hit enter — it should appear in terminal 2
within a second. If it doesn't, stop here and fix Kafka before doing anything
else. Everything downstream depends on this working.

### 2.6 Make Kafka reachable from services running outside the cluster

While you're actively coding, your service usually runs directly on your
laptop (not inside a container) so you get instant reload. To let it reach
Kafka inside the kind cluster:

```bash
kubectl -n kafka port-forward svc/gym-kafka-dev-kafka-bootstrap 9092:9092
```

Leave this running in a terminal tab. Your local service now connects to
`localhost:9092` exactly as if Kafka were running on your machine directly.

---

## PART 3 — Repo structure recap (so this guide is self-contained)

```
gym-platform-infra     # kind config, kafka-cluster.yaml, topics/ — everyone pulls this
gym-platform-docs      # event schema definitions (the actual JSON shape of each event)
gym-shared-libs        # shared Kafka producer/consumer wrapper, shared types
gym-api-gateway
gym-auth-service
gym-operations-service # occupancy + booking + membership
gym-people-service     # profiles + progress + plans
gym-social-service     # community + chat + notifications
gym-commerce-service   # catalog + orders + payments
gym-ai-service         # AI features + chatbot
```

Each service repo, internally, uses MVC:

```
src/
├── index.ts
├── config/
├── models/
├── controllers/
├── services/
├── routes/
├── events/
│   ├── producers/
│   └── consumers/
├── middleware/
└── db/migrations/
```

---

## PART 4 — How services actually talk to each other across repos

This is the part that's easy to get lost on, so let's be very explicit.

**There is no direct connection between two services' code.** `gym-operations-service`
never imports anything from `gym-commerce-service`, never calls its API to tell
it something happened. Instead:

1. `gym-operations-service` finishes booking a session in its own database.
2. It **publishes** a message to the `session.booked` Kafka topic — just a JSON
   payload like `{ "sessionId": "...", "customerId": "...", "amount": 500 }`.
3. `gym-commerce-service` has a **consumer** already running, subscribed to
   that exact topic. Kafka delivers the message to it automatically.
4. `gym-commerce-service` reacts — charges the card, then publishes its own
   message to `payment.succeeded` (or `payment.failed`).
5. `gym-operations-service` has a consumer subscribed to `payment.succeeded`
   and reacts by marking the session confirmed.

Neither service ever calls the other's URL. Neither knows the other exists.
They only know about **topic names and the shape of the JSON on them** — and
that shape is exactly what's documented in `gym-platform-docs`, so both teams
build against the same agreed contract.

**Concretely, in code, this is what it looks like on each side:**

```typescript
// gym-operations-service/src/events/producers/sessionBooked.producer.ts
import { kafkaProducer } from '../../config/kafka';

export const publishSessionBooked = async (session) => {
  await kafkaProducer.send({
    topic: 'session.booked',
    messages: [{ value: JSON.stringify({
      sessionId: session.id,
      customerId: session.customerId,
      amount: session.price,
    }) }],
  });
};
```

```typescript
// gym-commerce-service/src/events/consumers/sessionBooked.consumer.ts
import { kafkaConsumer } from '../../config/kafka';
import { paymentService } from '../../services/payment.service';

export const startSessionBookedConsumer = async () => {
  await kafkaConsumer.subscribe({ topic: 'session.booked', fromBeginning: false });
  await kafkaConsumer.run({
    eachMessage: async ({ message }) => {
      const event = JSON.parse(message.value.toString());
      await paymentService.chargeForSession(event);
    },
  });
};
```

That's the entire mechanism. Every cross-service interaction in your system
is this same pattern: **publish a JSON event to a named topic, and whoever
cares subscribes to that topic name.**

---

## PART 5 — What each person does, every single day

### First thing when starting work for the day

```bash
# 1. make sure your cluster is up (skip if you never tore it down)
kubectl cluster-info --context kind-gym-dev
# if this fails, your cluster is gone — recreate it (Part 2, steps 2.1-2.4)

# 2. pull latest infra changes in case topics were added
cd gym-platform-infra && git pull

# 3. re-apply in case new topics were added since yesterday
kubectl apply -f kafka/topics/ -n kafka

# 4. start the Kafka port-forward, leave it running in its own terminal tab
kubectl -n kafka port-forward svc/gym-kafka-dev-kafka-bootstrap 9092:9092

# 5. pull latest changes in the service repo(s) you're working on
cd ../gym-operations-service && git pull

# 6. start your service
npm run dev
```

### While implementing a feature inside your service

1. Check `gym-platform-docs` for the event(s) involved — does the event you
   need to publish or consume already have an agreed JSON shape? If not,
   propose it there first (a short PR, quick review), don't invent it
   silently in your own service.
2. Write the model (DB access) → service (business logic) → controller
   (HTTP handling) → route (wiring), in that order.
3. If your feature needs to tell another service something happened, write
   a producer in `events/producers/` and call it from your service layer,
   not from the controller.
4. If your feature needs to react to something another service did, write a
   consumer in `events/consumers/`, and register it on startup in `index.ts`.
5. **Test it against the real local Kafka**, not a mock — use the manual
   producer/consumer trick from Part 2.5, or write an integration test that
   actually publishes/consumes, so you know it works before opening a PR.
6. Open a PR. Get it reviewed. Merge.

### Before ending the day (optional but recommended)

You can just leave the kind cluster running overnight — it's local to your
machine and costs nothing but a bit of RAM. If you want to reclaim resources:

```bash
kind delete cluster --name gym-dev
```

Just remember you'll redo Part 2 (steps 2.1–2.4) tomorrow if you do this —
some teams instead just leave it running for weeks at a time and only
recreate it when something breaks.

---

## PART 6 — Quick sanity checklist when something feels broken

Run through this top to bottom, in order:

```bash
# is the cluster alive?
kubectl cluster-info --context kind-gym-dev

# is Kafka running?
kubectl get pods -n kafka

# do the topics exist?
kubectl get kafkatopics -n kafka

# is my port-forward to Kafka still open? (check the terminal tab)

# is my service actually connecting? check its own logs for connection errors
npm run dev   # watch the console output on startup

# did I actually publish/consume anything? verify manually with the console
# producer/consumer trick from Part 2.5 on the exact topic name involved
```

Almost every "service A isn't reacting to service B" bug traces back to one
of: port-forward not running, topic name typo/mismatch between publisher and
consumer, or JSON shape mismatch from not checking `gym-platform-docs` first.

---

That's the entire loop: **stand up cluster once → keep Kafka port-forwarded
while coding → publish/consume named topics with agreed JSON shapes → verify
manually → PR.** Nothing else is required to get moving.
