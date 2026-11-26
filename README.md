# k8s-day-gitops

Déploiement GitOps de l'application `hello-web` sur Kubernetes avec **Helm** et **Argo CD**.
Le cluster se configure à partir de ce dépôt : tout changement passe par Git, et les modifications faites à la main dans le cluster sont annulées.

## Contenu du dépôt

```
.
├── root-app.yaml                 # Application racine (app of apps) : déploie tout le dossier apps/
├── apps/                         # Manifestes Argo CD gérés par root-app.yaml
│   ├── project-hello.yaml        # AppProject "hello" : source = ce dépôt, destinations = hello-*
│   ├── applicationset.yaml       # hello-web en dev, staging et prod
│   ├── podinfo.yaml              # chart tiers installé depuis un dépôt Helm
│   └── hello-web-oci.yaml        # hello-web installé depuis le chart publié sur GHCR
├── argocd/
│   ├── values.yaml               # valeurs du chart officiel argo/argo-cd
│   └── argocd-self-managed.yaml  # Application qui fait gérer Argo CD par lui-même
└── hello-web/                    # chart Helm de l'application
    ├── Chart.yaml
    ├── values.yaml               # valeurs par défaut (dev, staging)
    ├── values-prod.yaml          # surcharges de prod
    └── templates/
        ├── _helpers.tpl          # nom complet et labels app.kubernetes.io/*
        ├── deployment.yaml
        ├── service.yaml
        ├── migrations-job.yaml   # hook PreSync exécuté avant chaque synchronisation
        └── NOTES.txt
```

## Le chart `hello-web`

Il déploie l'image `nginxdemos/hello` avec un Deployment et un Service. Le tag d'image est toujours figé (par défaut `appVersion`, jamais `latest`).

| Valeur | Rôle | Valeur actuelle |
|---|---|---|
| `replicaCount` | nombre de répliques | `4` |
| `image.repository` | image | `nginxdemos/hello` |
| `image.tag` | tag (vide = `appVersion` du chart) | `0.4-plain-text` |
| `service.port` | port du Service | `80` |
| `greeting` | injecté dans la variable d'environnement `GREETING` | `Hello from Git` |
| `resources` | requêtes et limites du conteneur | `10m` / `32Mi`, limite `64Mi` |
| `probes.enabled` | sondes readiness et liveness | `true` |

Vérifier le chart en local :

```bash
helm lint ./hello-web
helm template hello-web ./hello-web
# validation stricte des objets rendus par le serveur API
helm template hello-web ./hello-web | kubectl apply --server-side --dry-run=server -f -
```

## Environnements

| Application | Namespace | Source | Valeurs |
|---|---|---|---|
| `hello-web-dev` | `hello-dev` | ce dépôt, dossier `hello-web/` | `values.yaml` |
| `hello-web-staging` | `hello-staging` | ce dépôt, dossier `hello-web/` | `values.yaml` |
| `hello-web-prod` | `hello-prod` | ce dépôt, dossier `hello-web/` | `values.yaml` + `values-prod.yaml` |
| `hello-web-oci` | `hello-oci` | `ghcr.io/yasser-zamzami/charts`, chart `hello-web` 0.2.0 | dans l'Application |
| `podinfo` | `podinfo` | `https://stefanprodan.github.io/podinfo` | dans l'Application |

Toutes les Applications sont en synchronisation automatique avec `prune` et `selfHeal`.

## Installation

Prérequis : un cluster Kubernetes (Docker Desktop ou kind), `kubectl` et `helm` v4.

1. Installer Argo CD avec une version de chart figée :
   ```bash
   helm repo add argo https://argoproj.github.io/argo-helm
   helm repo update
   helm upgrade --install argocd argo/argo-cd -n argocd --create-namespace \
     --version 10.20.0 -f argocd/values.yaml --wait
   ```
2. Amorcer tout le reste d'une seule commande :
   ```bash
   kubectl apply -f root-app.yaml
   ```
   Les sync waves garantissent l'ordre : le projet `hello` et la déclaration du registre OCI (vague `-1`) sont créés avant les Applications (vague `0`).
3. Facultatif : laisser Argo CD gérer sa propre installation :
   ```bash
   kubectl apply -f argocd/argocd-self-managed.yaml
   ```

## Accès

Interface Argo CD sur https://localhost:8080 (utilisateur `admin`) :

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
# mot de passe initial
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
```

Sous PowerShell, pour le mot de passe :

```powershell
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String((kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}')))
```

Application `hello-web` (dev) sur http://localhost:8081 :

```bash
kubectl -n hello-dev port-forward svc/hello-web 8081:80
```

podinfo sur http://localhost:9898 :

```bash
kubectl -n podinfo port-forward svc/podinfo 9898:9898
```

## Déployer un changement

1. Modifier `hello-web/values.yaml` (ou `values-prod.yaml` pour la prod).
2. `git commit` puis `git push`.
3. Argo CD interroge le dépôt toutes les 20 s (`timeout.reconciliation` dans `argocd/values.yaml`) et applique le changement. Le bouton **Refresh** de l'interface évite d'attendre.

Revenir en arrière : `git revert <commit>` puis `git push`.

Comportements à connaître :

- Une modification faite à la main (`kubectl scale`, `kubectl delete`…) est annulée en quelques secondes (`selfHeal`).
- Un fichier supprimé de Git est supprimé du cluster (`prune`).
- Ajouter un environnement revient à ajouter un élément à la liste du générateur dans `apps/applicationset.yaml`.
- Modifier `argocd/values.yaml` reconfigure Argo CD lui-même.
- Le projet `hello` refuse tout déploiement hors des namespaces `hello-*`, par exemple dans `kube-system`.
- Le Job `migrations-job.yaml` s'exécute à chaque synchronisation, avant le Deployment et le Service.

## Publier une nouvelle version du chart

1. Augmenter `version` dans `hello-web/Chart.yaml`.
2. Empaqueter et pousser sur GHCR :
   ```bash
   helm package ./hello-web
   helm registry login ghcr.io -u <utilisateur-github>
   helm push hello-web-<version>.tgz oci://ghcr.io/yasser-zamzami/charts
   ```
3. Mettre la nouvelle version dans `targetRevision` de `apps/hello-web-oci.yaml`, puis committer et pousser.

## Désinstallation

```bash
kubectl delete -f argocd/argocd-self-managed.yaml   # pas de finalizer : Argo CD reste installé
kubectl delete -f root-app.yaml                     # les finalizers suppriment les applications déployées
helm uninstall argocd -n argocd
kubectl delete namespace argocd
kubectl get crds -o name | grep argoproj.io | xargs kubectl delete   # le chart conserve ses CRD
```
