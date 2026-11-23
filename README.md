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
    └── prod/kustomization.yaml  # à venir (dépend du déploiement AWS)
```

## Principe base + overlays

La **base** décrit ce qui est commun à tous les environnements. Chaque **overlay** ne modifie que ce qui change (namespace, nombre de replicas, tag d'image) avec Kustomize, sans dupliquer les manifests.

| Environnement | Namespace | Replicas | Image |
|---|---|---|---|
| base | (aucun) | 2 | `ghcr.io/dhia-sel/convertix:0.1.0` |
| dev | `convertix-dev` | 1 | `ghcr.io/dhia-sel/convertix:0.1.0` |
| prod | à venir | à venir | à venir (ECR) |

## Rendre les manifests localement

Prérequis : `kubectl` (Kustomize est intégré).

```powershell
kubectl kustomize base
kubectl kustomize overlays/dev
```

## Flux GitOps

1. Un changement de version se fait par un **commit** sur `main` (par exemple le `newTag` de l'image dans `overlays/dev/kustomization.yaml`).
2. ArgoCD détecte le commit et synchronise le cluster (`argocd/application-dev.yaml`).
3. Une ressource supprimée du dépôt est supprimée du cluster (`prune`), et une modification manuelle faite avec `kubectl` est annulée (`selfHeal`).
4. **Rollback** : `git revert` du commit fautif, puis push. Aucune commande `kubectl` n'est nécessaire.

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
- [ ] Synchronisation ArgoCD testée sur un cluster kind
- [ ] Mise à jour automatique du tag d'image par la CI
- [ ] Overlay prod et déploiement AWS (OIDC, ECR)