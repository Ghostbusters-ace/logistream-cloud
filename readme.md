# LogiStream — Système de Tracking de Flotte en Temps Réel

LogiStream est une plateforme cloud-native de traitement de flux de données (Event-Driven Architecture) conçue pour suivre en temps réel la position géographique d'une flotte de camions de livraison, détecter les excès de vitesse et pousser les données vers un tableau de bord.

Cette architecture moderne résout les problèmes de scalabilité et de latence des anciens systèmes monolithiques en s'appuyant sur **Apache Kafka** comme cœur d'ingestion et **Kubernetes (GKE)** pour l'orchestration des microservices.

---

## 🏗️ Architecture du Système

Le projet est entièrement conteneurisé et déployé sur un cluster managé Google Kubernetes Engine (GKE) en mode Autopilot.


### Flux de données de bout en bout :
1. 🚚 **Camions (Simulateur GPS) :** Le microservice `gps-producer` génère en continu des coordonnées géographiques réalistes pour chaque camion.
2. 🔑 **Partitionnement par Clé :** Les messages sont envoyés à Kafka avec l'ID du camion comme clé de partitionnement. Cela garantit que toutes les positions d'un même camion arrivent dans la même partition et sont traitées dans l'ordre chronologique exact.
3. 🪵 **Apache Kafka (Strimzi) :** Un cluster Kafka résilient (basé sur KRaft) orchestré par l'opérateur Strimzi ingère les flux de données à haute disponibilité sur le topic `truck-positions`.
4. 🧠 **Traitement & Alertes (Tracker Consumer) :** Le microservice `tracker-consumer` (faisant partie du groupe `tracker-service-group`) consomme les messages en temps réel, calcule la vitesse entre deux positions et génère un événement sur le topic `delivery-alerts` si une limite est dépassée.
5. 📊 **Dashboard (Visualisation) :** Les données finalisées et les alertes de criticité sont centralisées et prêtes pour la consommation des interfaces utilisateurs.

---

## 🛠️ Technologies Utilisées

* **Orchestration & Cloud :** Google Kubernetes Engine (GKE Autopilot), Google Artifact Registry.
* **Streaming de données :** Apache Kafka (v3.7.0) via l'opérateur Kubernetes **Strimzi**.
* **Développement Microservices :** Node.js avec la librairie hautement résiliente **KafkaJS**.
* **CI/CD :** Pipeline automatisé avec **GitHub Actions** (Validation ➔ Build Docker ➔ Push GAR ➔ GitOps Deployment sur GKE).
* **Observabilité :** Cloud Monitoring (Suivi du Consumer Lag), Cloud Logging (Centralisation des logs structurés).

---

## 📦 Structure du Dépôt

```text
├── .github/
│   └── workflows/
│       └── logistream-deploy.yml  # Pipeline CI/CD complet (Build & Deploy)
├──
│── producer/              # Code source du simulateur de positions
│── consumer/          # Code source de l'analyseur de flux et alertes
├── k8s/
│   ├── apps-deployments.yml       # Configuration des Deployments & Services GKE
│   ├── kafka-cluster.yml          # Définition du cluster Kafka (Strimzi CRD)
│   ├── kafka-topics.yml           # Définition déclarative des topics Kafka
│   └── hpa.yml                    # Horizontal Pod Autoscaler pour la scalabilité
└── README.md

```

---

## 🚀 Déploiement Rapide

### 1. Prérequis

* Un compte Google Cloud Platform avec un projet actif.
* Le SDK `gcloud` et `kubectl` installés localement (ou via GitHub Codespaces).
* L'opérateur Strimzi déployé sur le cluster.

### 2. Initialisation de l'infrastructure

Configurez l'accès à votre cluster GKE :

```bash
gcloud container clusters get-credentials logistream-cluster --region=europe-west9

```

Déployez l'écosystème Kafka et applicatif :

```bash
# Déploiement des topics et du cluster Kafka
kubectl apply -f k8s/kafka-cluster.yml -n logistream
kubectl apply -f k8s/kafka-topics.yml -n logistream

# Déploiement des microservices applicatifs
kubectl apply -f k8s/apps-deployments.yml -n logistream
kubectl apply -f k8s/hpa.yml -n logistream

```

---

## 📊 Diagnostics et Observabilité

### Vérifier la consommation des ressources

```bash
kubectl top pods -n logistream -l strimzi.io/cluster=logistream-kafka

```

### Analyser le Consumer Lag (Retard de traitement)

Pour s'assurer que le système traite les messages en temps réel sans accumuler de retard, interrogez le groupe de consommateurs directement depuis un broker Kafka :

```bash
kubectl exec -n logistream logistream-kafka-pool-a-5 -c kafka -- bin/kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group tracker-service-group

```

*Un `LAG` à `0` confirme un traitement fluide et sans latence.*

---

## 🧹 Nettoyage des ressources

Pour éviter des coûts de facturation inutiles sur Google Cloud, détruisez l'ensemble de l'infrastructure à la fin de vos tests :

```bash
kubectl delete -f k8s/ -n logistream
kubectl delete kafka logistream-kafka -n logistream
kubectl delete namespace logistream kafka
gcloud container clusters delete logistream-cluster --region=europe-west9 --quiet

```