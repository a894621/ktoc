# Formation Kubernetes (2 jours) - Parcours des Labs

Fil rouge de la formation : on déploie **Countvisit**, une application web Python (Flask) qui compte les visiteurs dans une base **PostgreSQL**. Chaque lab ajoute une brique de production à cette application : on part d'un Pod isolé et on termine avec une application packagée (Helm), exposée en HTTPS, adossée à une base PostgreSQL haute disponibilité et déployée en GitOps.

## Progression

| # | Lab | Durée | Ce que vous savez faire en sortant |
|---|-----|-------|------------------------------------|
| 0 | Découverte du cluster | 20 min | Vous connecter au cluster, lire son architecture, ouvrir le dashboard |
| 1 | kubectl et API Kubernetes | 40 min | Lire un kubeconfig, identifier les composants du control plane, interroger l'API |
| 2 | Pods, namespaces et labels | 60 min | Créer/inspecter/déboguer des Pods, organiser et filtrer avec labels et selectors |
| 3 | Deployments | 60 min | Déployer Countvisit, scaler, mettre à jour (rolling update), revenir en arrière |
| 4 | Services et découverte de services | 60 min | Exposer et relier Countvisit à PostgreSQL (ClusterIP, NodePort, LoadBalancer, DNS) |
| 5 | ConfigMaps et Secrets | 50 min | Sortir configuration et mots de passe des manifestes |
| 6 | Stockage persistant et StatefulSet | 80 min | Faire survivre les données de PostgreSQL à la perte d'un Pod |
| 7 | Sondes de santé | 45 min | Configurer liveness/readiness/startup et comprendre leur effet sur les mises à jour |
| 8 | Ingress et TLS | 50 min | Publier l'application par nom de domaine, par chemin, en HTTPS |
| 9 | Helm | 70 min | Installer un chart public, packager Countvisit dans un chart, upgrade/rollback |
| 10 | Opérateurs et CloudNativePG | 75 min | Comprendre CRD + opérateur, déployer PostgreSQL HA et tester un failover |
| 11 | Ordonnancement et ressources (*complémentaire*) | 80 min | Placer des Pods (taints, affinity), dimensionner (requests/limits), poser des quotas |
| 12 | RBAC et ServiceAccounts (*complémentaire*) | 60 min | Créer un utilisateur, lui donner des droits précis, auditer avec `auth can-i` |
| 13 | GitOps avec ArgoCD (*complémentaire*) | 60 min | Déployer depuis Git, observer le self-healing |

**Planning conseillé**

* **Jour 1 (≈ 4h50 de labs)** : Labs 0 à 5 - on construit l'application et on la relie à sa base.
* **Jour 2 (≈ 5h20 de labs)** : Labs 6 à 10 - persistance, fiabilité, exposition, packaging, haute disponibilité.
* **Labs complémentaires (11, 12, 13)** : indépendants du fil rouge (sauf le 13 qui réutilise le chart du Lab 9), à placer si le groupe avance vite ou à donner en autonomie.

Les durées sont des estimations pour un participant qui lit les indications sans ouvrir les corrections. Si le temps manque, coupez d'abord les parties marquées *Bonus*.

## Conventions

* `kubectl` est aliasé en `k` (les labs écrivent `kubectl` en toutes lettres).
* Chaque participant dispose de **son propre cluster** (1 master + 3 workers, cf. Lab 0) : pas besoin de préfixer les noms par votre prénom.
* Les fichiers YAML de l'application sont rangés dans `~/labs/countvisit/` (créé au Lab 3). Gardez-les : les labs suivants les font évoluer.
* Namespaces utilisés : `dev` (Lab 2), `countvisit` (Labs 3 à 10, 13), puis un namespace dédié par lab complémentaire.
* Chaque lab se termine par un **état attendu** : vérifiez-le avant de passer au suivant. En cas de retard, les corrections des labs précédents permettent de rattraper rapidement.
* Les corrections sont dans des balises `<details>` : essayez d'abord seul, avec les indications et les liens vers la documentation.

## Images utilisées (registre AWS ECR public)

| Image | Usage |
|-------|-------|
| `public.ecr.aws/wizetraining/webapp-count:v1` et `:v2` | Application Countvisit, base de données configurée par variables d'environnement |
| `public.ecr.aws/wizetraining/webapp-count-secure:v1` | Countvisit "Entreprise" : mot de passe lu dans un fichier, routes `/healthz` et `/readyz` |
| `public.ecr.aws/wizetraining/postgres:16-alpine` | PostgreSQL |
| `public.ecr.aws/wizetraining/nginx:latest` | Serveur web de test |
| `public.ecr.aws/wizetraining/busybox:latest` | Pod outil (sleep, wget, nslookup) |

### Fiche de l'application Countvisit

* Écoute sur le port **5000**. La page `/` s'affiche immédiatement ; elle ouvre ensuite un flux `/stream` qui se connecte à PostgreSQL (5 tentatives), incrémente le compteur de la table `hits` et affiche le résultat. Chaque rafraîchissement de la page = 1 visite = 1 nouvelle connexion à la base.
* La page affiche aussi le nom du Pod qui a répondu (utile pour voir le load balancing).
* Configuration de `webapp-count` : variables `PG_HOST` (défaut `postgres`), `PG_PORT` (`5432`), `PG_USER` (`postgres`), `PG_PASSWORD` (`postgres`), `PG_DB` (`postgres`).
* Configuration de `webapp-count-secure` : mêmes variables **sauf** `PG_PASSWORD` : le mot de passe est lu dans le fichier indiqué par `PG_PASSWORD_FILE` (défaut `/etc/app/secrets/pg_password`). Expose en plus `/healthz` (liveness) et `/readyz` (readiness).
