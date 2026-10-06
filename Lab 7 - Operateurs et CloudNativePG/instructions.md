# LAB 07 - Opérateurs : CRD et PostgreSQL haute disponibilité avec CloudNativePG

**Durée estimée : 75 minutes**

### Objectifs

* Comprendre qu'un **Opérateur** = **CRD** (nouveaux types d'objets) + **contrôleur** (la logique qui agit).
* Déployer un cluster PostgreSQL à 3 instances avec **CloudNativePG (CNPG)**.
* Connecter Countvisit au service d'écriture, puis provoquer une **panne du primaire** et constater le failover.

**Documentation utile :** [CustomResourceDefinitions](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/) · [Le pattern Opérateur](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) · [CloudNativePG](https://cloudnative-pg.io/documentation/current/) · [CNPG - bootstrap](https://cloudnative-pg.io/documentation/current/bootstrap/)

**Prérequis :** fin du Lab 9 (opérateur CNPG installé, release `countvisit`, Secret `pg-secret` de type basic-auth créé au Lab 5).

---

## Partie 1 : Étendre l'API avec une CRD (Rappel)

L'API Server de Kubernetes est extensible : une **CustomResourceDefinition** lui apprend un nouveau type d'objet, qui est ensuite traité comme un objet natif.

**Scénario :** une plateforme de formation veut gérer des objets `Course`. Groupe `stable.k8s.wizetraining.io`, version `v1`, portée *namespaced*, avec `spec.title` (string) et `spec.durationHours` (integer), nom court `crs`.

**1.** Avant toute déclaration, lancez `kubectl get courses`. Quelle erreur obtenez-vous ?

<details><summary>Correction</summary>

```bash
kubectl get courses       # error: the server doesn't have a resource type "courses"
```

</details>

**2.** Écrivez `course-crd.yaml` (`apiVersion: apiextensions.k8s.io/v1`, `kind: CustomResourceDefinition`, le nom de la CRD doit être `<pluriel>.<groupe>`) et appliquez-le. Vérifiez avec `kubectl api-resources`.

<details><summary>Correction</summary>

```yaml
# course-crd.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: courses.stable.k8s.wizetraining.io
spec:
  group: stable.k8s.wizetraining.io
  scope: Namespaced
  names:
    plural: courses
    singular: course
    kind: Course
    shortNames: [crs]
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              title:
                type: string
              durationHours:
                type: integer
```
```bash
kubectl apply -f course-crd.yaml
kubectl api-resources | grep course
```

</details>

**3.** Créez un objet `Course` `kubernetes-fundamentals` (titre et durée de votre choix), modifiez-le avec `kubectl patch`, puis listez-le avec le nom court. Tentez un `durationHours` avec du texte : que se passe-t-il ?

<details><summary>Correction</summary>

```yaml
# my-course.yaml
apiVersion: stable.k8s.wizetraining.io/v1
kind: Course
metadata:
  name: kubernetes-fundamentals
spec:
  title: "Les fondamentaux de Kubernetes"
  durationHours: 14
```
```bash
kubectl apply -f my-course.yaml
kubectl patch course kubernetes-fundamentals --type=merge -p '{"spec":{"durationHours": 16}}'
kubectl get crs
kubectl patch course kubernetes-fundamentals --type=merge -p '{"spec":{"durationHours": "beaucoup"}}'
#   -> rejeté : le schéma OpenAPI valide l'objet
```

</details>

**4.** Que fait Kubernetes concrètement avec cet objet `Course` ? Que manque-t-il pour qu'il se passe quelque chose ?

<details><summary>Correction</summary>

Kubernetes se contente de **stocker et valider** l'objet dans etcd : c'est de la donnée. Rien ne se passe tant qu'aucun **contrôleur** ne surveille ces objets et n'agit. CRD + contrôleur = **Opérateur**. Nettoyage : `kubectl delete -f my-course.yaml -f course-crd.yaml`.

</details>

---

## Partie 2 : Installer et Observer l'opérateur(10 min)

### Installation de l'Opérateur CloudNativePG avec Helm

Avant de pouvoir demander un cluster PostgreSQL, nous devons installer l'intelligence qui va le gérer.
Nous utiliserons Helm pour l'installation.
Voir la documentation ici : [https://cloudnative-pg.io/docs](https://cloudnative-pg.io/docs)

Installation avec Helm : https://github.com/cloudnative-pg/charts

```bash
# 1. Ajouter le dépôt officiel de CloudNativePG
helm repo add cnpg https://cloudnative-pg.github.io/charts

# 2. Mettre à jour les dépôts locaux
helm repo update

# 3. Installer l'opérateur dans un namespace dédié
# L'option --create-namespace crée 'cnpg-system' s'il n'existe pas
helm upgrade --install cnpg-operator cnpg/cloudnative-pg \
  --namespace cnpg-system \
  --create-namespace
```

**Vérification :** Attendez que le pod de l'opérateur soit en état `Running`.
```bash
kubectl get pods -n cnpg-system
```

**1.** Listez les CRD du groupe CNPG et les ressources de l'API `postgresql.cnpg.io`. Quelle ressource allez-vous manipuler ?

<details><summary>Correction</summary>

```bash
kubectl get crd | grep cnpg
kubectl api-resources --api-group=postgresql.cnpg.io        # clusters, backups, poolers, ...
```

</details>

**2.** Où tourne le contrôleur de l'opérateur ? Que va-t-il surveiller, d'après vous ?

<details><summary>Correction</summary>

```bash
kubectl get pods -n cnpg-system                             # le contrôleur
```
Le contrôleur surveille les objets `Cluster` de **tous** les namespaces ; pour chacun, il crée et maintient Pods, PVC, Services, Secrets et configure la réplication PostgreSQL.


</details>

**3.** Utilisez `kubectl explain` pour retrouver à quoi sert le champ `spec.instances` d'un `Cluster`.

<details><summary>Correction</summary>

```bash
kubectl explain clusters.postgresql.cnpg.io.spec.instances
```

</details>

> **Pour les curieux :** c'est exactement le modèle d'**ECK** (Elastic Cloud on Kubernetes) : des CRD `Elasticsearch`, `Kibana`... et un opérateur qui gère le cycle de vie. [Documentation ECK](https://www.elastic.co/docs/deploy-manage/deploy/cloud-on-k8s).

---

## Partie 3 : Déployer un cluster PostgreSQL à 3 instances (25 min)

**1.** À l'aide de la [documentation de bootstrap](https://cloudnative-pg.io/documentation/current/bootstrap/), écrivez `postgres-cluster.yaml` : un `Cluster` nommé `pg-database` avec :
* 3 instances (1 primaire + 2 réplicas),
* image `ghcr.io/cloudnative-pg/postgresql:16`,
* stockage 1Gi,
* `bootstrap.initdb` : base `app_db`, propriétaire `app_user`, mot de passe fourni par votre Secret `pg-secret`.

<details><summary>Correction</summary>

```yaml
# postgres-cluster.yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: pg-database
spec:
  instances: 3
  imageName: ghcr.io/cloudnative-pg/postgresql:16
  storage:
    size: 1Gi
  bootstrap:
    initdb:
      database: app_db
      owner: app_user
      secret:
        name: pg-secret          # type basic-auth : username + password
```

</details>

**2.** Appliquez-le et observez les Pods, PVC et Services créés (`kubectl get pods -w`). Dans quel ordre les Pods apparaissent-ils ? Combien de PVC ?

<details><summary>Correction</summary>

```bash
kubectl apply -f postgres-cluster.yaml
kubectl get pods -w                        # job d'initialisation, puis pg-database-1, -2, -3 un par un
kubectl get pvc                            # 1 PVC par instance
kubectl get cluster
```

</details>

**3.** Listez les Services générés (`pg-database-rw`, `-ro`, `-r`). À quoi sert chacun ?

<details><summary>Correction</summary>

```bash
kubectl get svc | grep pg-database
```
* `pg-database-rw` : toujours le **primaire** (lectures + écritures) ;
* `pg-database-ro` : uniquement les **réplicas** (lectures, décharge le primaire) ;
* `pg-database-r` : n'importe quelle instance.

</details>

**4.** Retrouvez quel Pod est le primaire (label `cnpg.io/instanceRole`) et l'état global (`kubectl get cluster`).

<details><summary>Correction</summary>

```bash
kubectl get pods -L cnpg.io/instanceRole -l cnpg.io/cluster=pg-database
```

</details>

---

## Partie 4 : Connecter l'application (10 min)

**1.** Quel Service faut-il utiliser comme `PG_HOST` pour que l'application écrive dans la base ? Pourquoi pas directement le nom d'un Pod ?

<details><summary>Correction</summary>

Nous devons utiliser le service `pg-database-rw`. Si le maître tombe, l'opérateur promeut une réplique et met à jour ce service automatiquement pour qu'il pointe vers le nouveau maître. C'est la magie de l'opérateur !

</details>

**2.** Mettez à jour l'a pplication Web pour pointer sur le bon Service, puis prenez en compte le nouveau ConfigMap dans les Pods (rappel du Lab 5 : une variable d'environnement n'est lue qu'au démarrage). Vérifiez que la page affiche la connexion à la base et que le compteur repart à 1 (nouvelle base).

<details><summary>Correction</summary>

Il faut `pg-database-rw` : ce Service pointe **toujours vers le primaire actuel**, même après une bascule. Le nom d'un Pod changerait de rôle.

</details>

---

## Partie 5 : Simuler une panne du primaire (15 min)

**1.** Rechargez la page jusqu'à environ 10. Identifiez le Pod primaire.

**2.** Dans un terminal, surveillez les Pods (`-w`). Dans un autre, **supprimez le Pod primaire**. Pendant ce temps, continuez de recharger la page.

**3.** Qu'observez-vous : durée de l'interruption, Pod promu, valeur du compteur après la bascule ? Quel est le rôle des 5 tentatives de connexion de l'application ?

**4.** Que serait-il arrivé avec le StatefulSet du Lab 6 à 3 réplicas ? Complétez : réplication, failover, sauvegardes, mises à jour.

<details><summary>Correction</summary>

```bash
kubectl get pods -L cnpg.io/instanceRole -l cnpg.io/cluster=pg-database
kubectl get pods -w -l cnpg.io/cluster=pg-database
kubectl delete pod <pod-primaire>          # ou : kubectl delete pod -l cnpg.io/instanceRole=primary
kubectl get cluster                        # nouveau primaire
kubectl get endpointslices -l kubernetes.io/service-name=pg-database-rw
```
**Résultat attendu :** quelques secondes d'erreur ou d'attente (le temps de l'élection), puis le compteur reprend **exactement là où il était** : une réplique, synchronisée en continu, est promue primaire et le Service `-rw` est repointé automatiquement. L'application, qui réessaie sa connexion plusieurs fois, absorbe la bascule. L'ancien primaire revient ensuite comme réplica.

| | StatefulSet manuel | Opérateur CNPG |
|---|---|---|
| Réplication | Aucune (3 bases isolées) | Automatique (streaming replication) |
| Failover | Aucun | Automatique (promotion d'une réplique) |
| Sauvegardes | À scripter | Natives (objet `Backup`, S3/GCS/Azure) |
| Mises à jour de version | Risque de perte de données | Rolling update avec switchover |

L'opérateur encode le savoir-faire d'un DBA PostgreSQL dans un contrôleur.

</details>

> **État attendu en fin de lab :** `kubectl get cluster` affiche `pg-database` avec 3 instances saines, la release Helm `countvisit` utilise `pg-database-rw`, et le compteur a survécu à la perte du primaire.
