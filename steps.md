# Gym Platform — Team Onboarding & Local Development Guide

This is the single source of truth for getting the local development environment running and for how to build any service in this system. Read this before writing code in any `gym-*-service` repo.

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

Every service repo follows the same internal layout. If you're starting a new service, use **"Use this template"** on `gym-service-template`.

---

## 2. One-time machine setup

Install once, per developer machine:

```bash
-----------------------DOCKER-----------------------
# remove any old versions first
sudo dnf remove -y docker docker-client docker-client-latest docker-common \
  docker-latest docker-latest-logrotate docker-logrotate docker-engine

# add Docker's official repo
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# install Docker
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# start and enable on boot
sudo systemctl start docker
sudo systemctl enable docker

# allow your user to run docker without sudo
sudo usermod -aG docker $USER
newgrp docker

# verify
docker version
docker run hello-world

-----------------------KIND-----------------------
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/repodata/repomd.xml.key
EOF

sudo dnf install -y kubectl

# verify
kubectl version --client

# kind is a single Go binary, no package manager needed
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

kind version

-----------------------HELM-----------------------
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# verify
helm version

-----------------------NODEJS-----------------------
curl -fsSL https://rpm.nodesource.com/setup_20.x | sudo bash -
sudo dnf install -y nodejs

# verify
node -v
npm -v

-----------------------GIT-----------------------
sudo dnf install -y git

# verify
git --version

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
│   ├── index.js                    # entrypoint: bootstraps express, DB, Kafka, routes
│   ├── config/                     # env config, DB connection, Kafka connection
│   │   ├── database.js
│   │   └── kafka.js
│   ├── models/                     # data layer — one file per sub-domain entity
│   │   └── example.model.js
│   ├── views/                      # only if you're server-rendering anything (likely empty/unused for a pure API — most teams skip this folder entirely for APIs, see note below)
│   ├── controllers/                # request handling — calls services, shapes responses
│   │   └── example.controller.js
│   ├── services/                   # business logic — the actual "how", called by controllers
│   │   └── example.service.js
│   ├── routes/                     # maps URLs to controllers
│   │   ├── example.routes.js
│   │   └── index.js                # combines all routers, mounted in index.js
│   ├── events/
│   │   ├── producers/              # one file per event this service publishes
│   │   │   └── sessionBooked.producer.js
│   │   └── consumers/              # one file per event this service subscribes to
│   │       └── paymentSucceeded.consumer.js
│   ├── middleware/                 # auth check, error handler, request validation
│   │   ├── auth.middleware.js
│   │   └── errorHandler.middleware.js
│   └── db/
│       └── migrations/             # schema migrations — this service's OWN tables only
├── tests/
│   ├── unit/
│   └── integration/
├── k8s/
│   ├── deployment.yaml     # Pod configurations, replica limits, and container specs
│   ├── service.yaml        # Service definition mapping internal container ports
│   └── configmap.yaml      # Non-sensitive runtime environment variables for THIS service
├── .github/workflows/ci.yml
├── Dockerfile
├── .env.example
└── README.md
```

---

## 6. Debugging checklist when something doesn't work

- `kubectl get pods -A` — is everything actually running?
- `kubectl logs <pod-name>` — check your service's own logs first
- `kubectl -n kafka get kafkatopics` — does your topic actually exist?
- `kubectl describe pod <pod-name>` — usually reveals image pull errors, crash loops, failed probes
