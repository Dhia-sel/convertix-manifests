# convertix-manifests

Manifests Kubernetes de [convertix](https://github.com/Dhia-sel/convertix), une API de conversion d'unités écrite en Go. Ce dépôt est la **source de vérité GitOps** : ArgoCD lit ce dépôt et applique son contenu sur le cluster. Le code et les workflows CI sont dans le dépôt `convertix`.

## Structure

```text
.
├── argocd/
│   └── application-dev.yaml     # Application ArgoCD de l'environnement dev
├── base/
│   ├── deployment.yaml          # Deployment commun à tous les environnements
│   ├── service.yaml             # Service ClusterIP
│   └── kustomization.yaml
└── overlays/
    ├── dev/kustomization.yaml   # namespace, replicas et tag d'image de dev
    └── prod/kustomization.yaml  # squelette (namespace, replicas), image ECR à venir
```

## Principe base + overlays

La **base** décrit ce qui est commun à tous les environnements. Chaque **overlay** ne modifie que ce qui change (namespace, nombre de replicas, tag d'image) avec Kustomize, sans dupliquer les manifests.

| Environnement | Namespace | Replicas | Image |
|---|---|---|---|
| base | (aucun) | 2 | `ghcr.io/dhia-sel/convertix:0.1.0` |
| dev | `convertix-dev` | 1 | `ghcr.io/dhia-sel/convertix:0.1.0` |
| prod | `convertix-prod` | 2 | hérite de la base, image ECR à venir |

## Rendre les manifests localement

Prérequis : `kubectl` (Kustomize est intégré).

```powershell
kubectl kustomize base
kubectl kustomize overlays/dev
kubectl kustomize overlays/prod
```

## Mise en route sur un cluster kind

Prérequis : Docker, `kind`, `kubectl`.

```powershell
kind create cluster --name convertix
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s
kubectl apply -f argocd/application-dev.yaml
```

Vérification :

```powershell
kubectl get applications.argoproj.io -n argocd
kubectl get pods -n convertix-dev
```

L'Application doit passer à `Synced` et `Healthy`, et un pod `convertix` doit tourner dans `convertix-dev`. Le namespace est créé par ArgoCD (`CreateNamespace=true`).

## Flux GitOps

1. Un changement de version se fait par un **commit** sur `main` (par exemple le `newTag` de l'image dans `overlays/dev/kustomization.yaml`).
2. ArgoCD détecte le commit et synchronise le cluster (`argocd/application-dev.yaml`).
3. Une ressource supprimée du dépôt est supprimée du cluster (`prune`), et une modification manuelle faite avec `kubectl` est annulée (`selfHeal`).
4. **Rollback** : `git revert` du commit fautif, puis push. Aucune commande `kubectl` n'est nécessaire.

## Tests réalisés (dev, cluster kind)

| Propriété | Test | Résultat |
|---|---|---|
| Déploiement par commit | `count: 1` passé à `count: 2` dans `overlays/dev`, commit `3e32ac4`, push | un 2e pod créé par ArgoCD, sans `kubectl apply` |
| Correction de dérive (`selfHeal`) | `kubectl scale deployment convertix -n convertix-dev --replicas=5` | le déploiement est revenu à `2/2` immédiatement, ArgoCD a rétabli la valeur de Git |
| Rollback | `git revert HEAD`, commit `6439ee7`, push | retour à 1 pod, application `Synced` et `Healthy` |

Test de l'application déployée :

```powershell
kubectl port-forward -n convertix-dev svc/convertix 8080:80
curl.exe http://localhost:8080/healthz
```

## Choix de sécurité du Deployment

- Exécution en non-root (`runAsNonRoot`, UID/GID `65532`) avec un profil seccomp `RuntimeDefault`.
- `allowPrivilegeEscalation: false`, toutes les capabilities retirées (`drop: ALL`).
- Système de fichiers racine en lecture seule (`readOnlyRootFilesystem: true`).
- Token de ServiceAccount non monté (`automountServiceAccountToken: false`).
- Ressources déclarées : requests `25m` CPU / `16Mi` mémoire, limite mémoire `64Mi`.
- Sondes `liveness` et `readiness` sur `/healthz`.

## État du projet

- [x] Base Kustomize (Deployment, Service)
- [x] Overlay dev
- [x] Application ArgoCD dev
- [x] Synchronisation ArgoCD testée sur un cluster kind (commit, `selfHeal`, rollback)
- [x] Squelette de l'overlay prod (namespace, replicas)
- [ ] Mise à jour automatique du tag d'image par la CI
- [ ] Image ECR dans l'overlay prod et déploiement AWS