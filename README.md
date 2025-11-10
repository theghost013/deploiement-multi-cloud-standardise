# Déploiement Multi-Cloud Standardisé (Crossplane + Helm)

Normaliser et standardiser le déploiement Cloud Native sur AWS EKS et GCP GKE à l’aide de Crossplane (abstraction/infra) et Helm (applications). Ce dépôt fournit:
- Un contrat unique (XRD) pour décrire des clusters managés multi-cloud.
- Des Compositions Crossplane pour EKS et GKE.
- Un flux de déploiement applicatif manuel via Helm, identique sur les deux clouds.

---

## Prérequis (Outils Locaux)
Installez ces outils sur votre machine:
- kubectl 1.26+
- helm 3.11+
- gcloud CLI (authentifié sur votre projet GCP)
- aws CLI (configuré avec des identifiants ayant les droits EKS)
- Un cluster de management Kubernetes (ex: minikube) où Crossplane sera installé

Optionnel mais recommandé:
- watch, jq

---

## I. Préparation du Cloud & Sécurité

### 3a. Création des Fichiers de Secrets Locaux
Créez ces 3 fichiers à la racine du dépôt (ou mettez à jour leur contenu):

1) `aws-role-source.conf` – source des ARN IAM (documentation et bootstrap du Secret)
```
CLUSTER_ROLE_ARN=arn:aws:iam::651665396672:role/eks-cluster-role-crossplane
NODEGROUP_ROLE_ARN=arn:aws:iam::651665396672:role/eks-nodegroup-role
```

2) `aws-creds.txt` – identifiants AWS au format ini du provider Upbound
```
[default]
aws_access_key_id = <YOUR_AWS_ACCESS_KEY_ID>
aws_secret_access_key = <YOUR_AWS_SECRET_ACCESS_KEY>
```

3) `gcp-sa-key.json` – clé JSON d’un Compte de Service GCP habilité (Container API, Compute, etc.)
```
{
  "type": "service_account",
  ...
}
```

Assurez-vous aussi d’avoir votre Project ID GCP (fichier facultatif: `gcp-project-id.txt`). Le ProviderConfig GCP de ce dépôt est déjà paramétré sur `white-ground-477519-h1`.

### 3b. Préparation IAM / Rôles requis
- AWS
  - EKS Control Plane Role (assumé par EKS) – ex: `arn:aws:iam::689665396672:role/eks-cluster-role-crossplane`
  - EKS NodeGroup Role – ex: `arn:aws:iam::689665396672:role/eks-nodegroup-role`
- GCP
  - Le Compte de Service doit avoir les rôles nécessaires, par ex: Kubernetes Engine Admin, Compute Admin, Service Account User (à adapter selon vos règles).

---

## II. Configuration du Control Plane (Crossplane)
Toutes les commandes ci-dessous ciblent le cluster de management (ex: minikube).

### 4a. Installation de Crossplane (v1.13.0)
```fish
# Contexte management
kubectl config use-context minikube

# Helm repo Crossplane
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

# Installer Crossplane v1.13.0
helm upgrade --install crossplane crossplane-stable/crossplane \
  --namespace crossplane-system --create-namespace \
  --version 1.13.0

# Attendre que les pods soient prêts
kubectl -n crossplane-system rollout status deploy/crossplane --timeout=120s
kubectl -n crossplane-system rollout status deploy/crossplane-rbac-manager --timeout=120s
```

### 4b. Création des Secrets Kubernetes (providers + ARNs IAM)
```fish
# AWS credentials (provider-aws)
kubectl -n crossplane-system delete secret aws-creds --ignore-not-found
kubectl -n crossplane-system create secret generic aws-creds \
  --from-file=key=aws-creds.txt

# GCP credentials (provider-gcp)
kubectl -n crossplane-system delete secret gcp-creds --ignore-not-found
kubectl -n crossplane-system create secret generic gcp-creds \
  --from-file=key=gcp-sa-key.json

# Secret central des ARNs IAM (aws-roles-secret) depuis aws-role-source.conf
set -x CLUSTER_ROLE_ARN (grep '^CLUSTER_ROLE_ARN=' aws-role-source.conf | cut -d= -f2-)
set -x NODEGROUP_ROLE_ARN (grep '^NODEGROUP_ROLE_ARN=' aws-role-source.conf | cut -d= -f2-)

kubectl -n crossplane-system delete secret aws-roles-secret --ignore-not-found
printf '%s' $CLUSTER_ROLE_ARN | kubectl -n crossplane-system create secret generic aws-roles-secret \
  --from-file=cluster-role-arn=/dev/stdin
kubectl -n crossplane-system patch secret aws-roles-secret --type=merge \
  -p '{"data":{"nodegroup-role-arn":"'"'(printf $NODEGROUP_ROLE_ARN | base64 -w0)'"'"}}'
```

### 4c. Application du Contrat et des Providers
```fish
# Providers (packages Upbound)
kubectl apply -f crossplane/provider-aws.yaml
kubectl apply -f crossplane/provider-gcp.yaml

# ProviderConfigs (référencent les secrets précédents)
kubectl apply -f crossplane/aws-provider-config.yaml
kubectl apply -f crossplane/gcp-provider-config.yaml

# XRD (contrat) et EnvironmentConfig pour rôles AWS
kubectl apply -f crossplane/xrd-managed-cluster.yaml
kubectl apply -f crossplane/environmentconfig-aws-roles.yaml

# Compositions (AWS & GCP)
kubectl apply -f crossplane/composition-aws.yaml
kubectl apply -f crossplane/composition-gcp.yaml
```

---

## III. Déploiement Multi-Cloud & Validation

### 5a. Lancement de l’Infrastructure
```fish
# Instancier AWS (EKS)
kubectl apply -f deployments/instance-1-aws.yaml

# Instancier GCP (GKE)
kubectl apply -f deployments/instance-2-gcp.yaml

# Suivre la progression
autowatch; and kubectl get xmanagedclusters.devops.com -w
```

### 5b. Validation AWS (EKS)
Quand le cluster et le NodeGroup sont `Ready`, obtenez un kubeconfig local et déployez l’app via Helm:
```fish
# Auth locale vers le cluster EKS (nom = spec.id de l’instance)
aws eks update-kubeconfig --region eu-west-3 --name cluster-instance-1-aws

# Déployer l’app (chart fourni dans ce dépôt)
helm upgrade --install app-base ./app/app-base --namespace default --create-namespace

# Récupérer l’adresse (Service type LoadBalancer)
kubectl get svc -l app.kubernetes.io/name=app-base -o wide
```

### 5c. Validation GCP (GKE)
Pour GKE, authentifiez d’abord votre kubectl avec gcloud, puis déployez via Helm:
```fish
# Auth locale vers le cluster GKE
gcloud container clusters get-credentials cluster-instance-2-gcp \
  --region europe-west1 --project white-ground-477519-h1

# Déployer l’app (chart fourni dans ce dépôt)
helm upgrade --install app-base ./app/app-base --namespace default --create-namespace

# Récupérer l’adresse (Service type LoadBalancer)
kubectl get svc -l app.kubernetes.io/name=app-base -o wide
```

---

## Nettoyage
Supprimez d’abord les applications Helm sur chaque cluster cible, puis supprimez l’infra via Crossplane.
```fish
# GKE (après s’être authentifié avec gcloud)
helm uninstall app-base --namespace default; or true

# EKS (après avoir mis à jour le kubeconfig)
helm uninstall app-base --namespace default; or true

# Supprimer les XRs (management cluster)
kubectl delete -f deployments/instance-2-gcp.yaml --ignore-not-found
kubectl delete -f deployments/instance-1-aws.yaml --ignore-not-found

# Optionnel: supprimer les providers et compositions si vous voulez nettoyer le management cluster
kubectl delete -f crossplane/composition-gcp.yaml --ignore-not-found
kubectl delete -f crossplane/composition-aws.yaml --ignore-not-found
kubectl delete -f crossplane/xrd-managed-cluster.yaml --ignore-not-found
kubectl delete -f crossplane/gcp-provider-config.yaml --ignore-not-found
kubectl delete -f crossplane/aws-provider-config.yaml --ignore-not-found
kubectl delete -f crossplane/provider-gcp.yaml --ignore-not-found
kubectl delete -f crossplane/provider-aws.yaml --ignore-not-found

# Crossplane (si vous souhaitez le désinstaller)
helm -n crossplane-system uninstall crossplane; or true
```

---

## Fichiers à ignorer (.gitignore)
Vérifiez que ces fichiers sensibles ne sont pas versionnés:
```
aws-creds.txt
gcp-sa-key.json
aws-role-source.conf
gcp-project-id.txt
*.kubeconfig
```

---

## Notes
- Cette configuration utilise Crossplane Core v1.13.0 et les providers Upbound: AWS EKS v1.3.0 et GCP v0.25.0.
- Les ARN IAM nécessaires pour EKS sont injectés dans la Composition via un EnvironmentConfig et un Secret central `aws-roles-secret`. Le fichier `aws-role-source.conf` sert uniquement à (re)générer ce Secret.
- Le déploiement applicatif Helm est manuel pour chaque cluster afin de garder un flux clair et contrôlé.
