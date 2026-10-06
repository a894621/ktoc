# LAB 6 - Stockage persistant et StatefulSet

**Durée estimée : 80 minutes**

### Objectifs

* Comprendre PV, PVC et StorageClass (provisionnement statique puis dynamique).
* Déployer PostgreSQL avec un **StatefulSet** pour que le compteur survive à la perte d'un Pod.
* Comprendre ce que le StatefulSet fait... et ce qu'il ne fait pas (réplication).

**Documentation utile :** [Volumes persistants](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) · [StorageClasses](https://kubernetes.io/docs/concepts/storage/storage-classes/) · [StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) · [Tutoriel StatefulSet](https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/)

**Prérequis :** fin du Lab 5. Au Lab 4, vous avez vu le compteur retomber à 1 quand PostgreSQL redémarre : le système de fichiers d'un conteneur est éphémère.

- **PersistentVolume (PV)** : un morceau de stockage du cluster (le "disque").
- **PersistentVolumeClaim (PVC)** : la demande de stockage d'un utilisateur (la "réservation").
- **StorageClass** : un "catalogue" qui crée les PV automatiquement à la demande d'un PVC.

---

## Partie 1 : PV et PVC statiques (15 min)

Dans cette partie, vous jouez le rôle de l'administrateur qui provisionne le disque à la main.

**1.** Créez un PV `pv-demo` : 1Gi, accès `ReadWriteOnce`, type `hostPath` sur `/mnt/pv-demo`, avec `storageClassName: ""` (pas de classe, pour empêcher tout provisionnement dynamique).

<details><summary>Correction</summary>

```yaml
# pv-demo.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-demo
spec:
  capacity:
    storage: 1Gi
  accessModes: [ReadWriteOnce]
  storageClassName: ""
  hostPath:
    path: /mnt/pv-demo
```

```bash
kubectl apply -f pv-demo.yaml
kubectl get pv                       # PV passe est "Available"
```

</details>

**2.** Créez un PVC `pvc-demo` qui réclame 1Gi avec la même configuration. Que devient le PV (statut) ?

<details><summary>Correction</summary>

```yaml
# pvc-demo.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-demo
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: ""
  resources:
    requests:
      storage: 1Gi
```

```bash
kubectl apply -f pvc-demo.yaml
kubectl get pv,pvc                      # PV passe de "Available" à "Bound"
```

</details>

**3.** Créez un Pod `pod-pv` (busybox) qui monte ce PVC dans `/data` et exécute `sh -c "echo bonjour >> /data/out.txt && sleep 3600"`. Sur quel nœud tourne-t-il ?

<details><summary>Correction</summary>

```yaml
# pod-pv.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-pv
spec:
  containers:
  - name: app
    image: public.ecr.aws/wizetraining/busybox:latest
    command: ["sh", "-c", "echo bonjour >> /data/out.txt && sleep 3600"]
    volumeMounts:
    - name: vol
      mountPath: /data
  volumes:
  - name: vol
    persistentVolumeClaim:
      claimName: pvc-demo
```
```bash
kubectl apply -f pod-pv.yaml
kubectl get pod pod-pv -o wide
```

</details>

**4.** Supprimez le Pod, recréez-le : retrouvez-vous le fichier avec les deux lignes ? Si le Pod atterrit sur un autre nœud, que se passe-t-il, et que doit en déduire un administrateur sur `hostPath` ?

<details><summary>Correction</summary>

```bash
kubectl delete pod pod-pv && kubectl apply -f pod-pv.yaml
kubectl exec pod-pv -- cat /data/out.txt
```
Avec `hostPath`, les données sont **sur le disque d'un nœud précis** : si le Pod est replanifié ailleurs, il voit un répertoire vide. C'est acceptable pour un lab, jamais en production : on utilise un stockage réseau/CSI.

</details>

Nettoyez : `kubectl delete pod pod-pv && kubectl delete pvc pvc-demo && kubectl delete pv pv-demo`.

---

## Partie 2 : StorageClass et provisionnement dynamique (15 min)

En pratique, personne ne crée les PV à la main : une **StorageClass** les fabrique à la demande.

**1.** Listez les StorageClasses (vues au Lab 0). Quel est le `PROVISIONER`, la `RECLAIMPOLICY` et le `VOLUMEBINDINGMODE` de la classe par défaut ? Que signifient ces deux dernières valeurs ?

<details><summary>Correction</summary>

```bash
# 1
kubectl get storageclass
kubectl describe storageclass <nom>
```
* `reclaimPolicy: Delete` : quand le PVC est supprimé, le PV **et les données** sont supprimés (avec `Retain`, ils sont conservés pour récupération manuelle).
* `volumeBindingMode:` 
  * `WaitForFirstConsumer` => le volume n'est créé qu'une fois un Pod planifié, pour le créer au bon endroit ;
  * `Immediate` => créé dès le PVC.

</details>

**2.** Créer un StorageClass `sc-demo.yaml` :
* Nom du SC `sc-demo-<votre nom>`.
* Provisionner : `longhorn` (ou celui de votre cluster).
* Reclaim Policy : `Delete`. Quel est l'impact ?
* Mode de Binding : `WaitForFirstConsumer`. Quel est l'impact ?

<details><summary>Correction</summary>

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: sc-demo-toto
provisioner: <que mettrez-vous ?> # Exemple pour Longhorn
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```
</details>


**3.** Créez un PVC `pvc-dynamic` de 1Gi dont le `storageClassName` est celui que vous venez de créer (NB: Si `storageClassName` est vide/absent alors c'est le StorageClass par défaut qui s'applique). Que se passe-t-il dans `kubectl get pvc,pv` ? Si le PVC reste `Pending`, pourquoi ?

<details><summary>Correction</summary>

```yaml
# pvc-dynamic.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-dynamic
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
  storageClassName: sc-demo-toto
```
```bash
kubectl apply -f pvc-dynamic.yaml
kubectl get pvc                    # Pending si WaitForFirstConsumer : normal, aucun Pod ne l'utilise encore
```

</details>

**3.** Créez un Pod busybox qui monte ce PVC dans `/data`. Observez le PVC, puis le PV qui a été créé automatiquement.

<details><summary>Correction</summary>

Créez un Pod identique à `pod-pv` en remplaçant `claimName: pvc-demo` par `pvc-dynamic`, puis `kubectl get pvc,pv` : le PVC est `Bound` et un PV a été créé tout seul.

</details>

**4.** Supprimez le Pod, puis le PVC. Que devient le PV ? Quel est l'impact de `reclaimPolicy: Delete` ?

<details><summary>Correction</summary>

```bash
kubectl delete pod <pod> ; kubectl delete pvc pvc-dynamic
kubectl get pv                     # le PV a disparu (reclaimPolicy: Delete)
```

</details>

---

## Partie 3 : PostgreSQL avec un StatefulSet (35 min)

Un **StatefulSet** est un Deployment pour applications à état : chaque Pod a une **identité stable** (`postgres-0`) et **son propre volume** créé via `volumeClaimTemplates`, qui le suit même après un redémarrage.

**1.** Supprimez le Deployment `postgres` du Lab 5 (gardez le Service `postgres`, il pointera vers le nouveau Pod car il sélectionne `app=postgres`).

<details><summary>Correction</summary>

```bash
kubectl delete deployment postgres
```

</details>

**2.** Créez `postgres-statefulset.yaml` contenant :
* un Service **headless** `postgres-headless` (`clusterIP: None`, port 5432, selector `app: postgres`),
* un StatefulSet `postgres` : 1 réplica, `serviceName: postgres-headless`, mêmes image, variables d'environnement (Secret + ConfigMap) et label `app: postgres` que le Deployment,
* un `volumeClaimTemplates` nommé `data` de 1Gi (`ReadWriteOnce`, classe par défaut), monté dans `/var/lib/postgresql/data` avec `subPath: pgdata` (PostgreSQL refuse un répertoire de données qui contient `lost+found`).


<details><summary>Correction</summary>

```yaml
# postgres-statefulset.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None                 # pas d'IP virtuelle : le DNS renvoie directement les Pods
  selector:
    app: postgres
  ports:
  - port: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: public.ecr.aws/wizetraining/postgres:16-alpine
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: pg-secret
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: pg-secret
              key: password
        - name: POSTGRES_DB
          valueFrom:
            configMapKeyRef:
              name: webapp-config
              key: PG_DB
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
          subPath: pgdata
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

</details>

**3.** Appliquez. Quels PVC et quels Pods ont été créés ? Comment s'appelle le PVC ?

<details><summary>Correction</summary>

```bash
kubectl apply -f postgres-statefulset.yaml
kubectl get pods,pvc,pv                       # pod postgres-0, PVC data-postgres-0
```

</details>

**4.** **Test de persistance.** Rechargez la page de l'application jusqu'à environ 5. Supprimez le Pod `postgres-0`, attendez sa recréation, rechargez. Le compteur continue-t-il ?

<details><summary>Correction</summary>

```bash
# 4
kubectl delete pod postgres-0
kubectl get pods -w
```
Le compteur **reprend où il en était** : Kubernetes rattache le même PVC (`data-postgres-0`) au Pod recréé sous le même nom.


</details>

**5.** Supprimez même le StatefulSet entier (`kubectl delete statefulset postgres`), puis ré-appliquez-le. Le compteur est-il toujours là ? Pourquoi ?

<details><summary>Correction</summary>

```bash
# 5
kubectl delete statefulset postgres
kubectl apply -f postgres-statefulset.yaml
```
Les données sont toujours là : **supprimer un StatefulSet ne supprime pas ses PVC** (protection volontaire des données).

</details>

**6.** Depuis un Pod `webapp`, résolvez `postgres-0.postgres-headless`. Quelle est l'utilité d'un Service headless ?

<details><summary>Correction</summary>


```bash
# 6
kubectl exec deploy/webapp -- python -c "import socket; print(socket.gethostbyname('postgres-0.postgres-headless'))"
```
Un Service headless donne à **chaque Pod** un nom DNS individuel et stable (`<pod>.<service>`) : indispensable quand les membres d'un cluster (base de données) doivent se joindre un par un (primaire/réplicas).

</details>

---

## Partie 4 : La limite du StatefulSet (15 min)

Pour la haute disponibilité, on veut plusieurs instances de PostgreSQL.

**1.** Passez le StatefulSet à **3 réplicas**. Attendez les Pods `postgres-1` et `postgres-2`.

<details><summary>Correction</summary>

```bash
kubectl scale statefulset postgres --replicas=3
kubectl get pods -l app=postgres -w
```

</details>

**2.** Rechargez la page de l'application une dizaine de fois, en notant le compteur. Que constatez-vous ?

<details><summary>Correction</summary>

**Constat :** le compteur est incohérent (15, puis 1, puis 16, puis 2...).

</details>

**3.** Expliquez pourquoi, en vous aidant de `kubectl get pvc` et de ce que fait le Service `postgres` entre les 3 Pods.

<details><summary>Correction</summary>

**Explication :** le Service `postgres` répartit chaque connexion entre les 3 Pods. Chaque Pod a **son propre volume vierge** (3 PVC distincts) et sa propre base : ce sont 3 PostgreSQL indépendants qui ne se connaissent pas. Le StatefulSet gère l'**infrastructure** (identités stables, volumes dédiés) mais ne connaît pas le métier de PostgreSQL : il ne configure ni la réplication, ni le choix du primaire, ni le failover.

</details>

**4.** Revenez à 1 réplica et supprimez les PVC devenus inutiles (`data-postgres-1`, `data-postgres-2`).

<details><summary>Correction</summary>

```bash
kubectl scale statefulset postgres --replicas=1
kubectl delete pvc data-postgres-1 data-postgres-2     # le scale down ne supprime pas les PVC
```
Pour obtenir un vrai cluster PostgreSQL (primaire + réplicas synchronisés + bascule automatique) il faut une "intelligence" qui connaît PostgreSQL : un **Opérateur**. C'est l'objet du Lab 10.

</details>

> **État attendu en fin de lab :** StatefulSet `postgres` à 1 réplica avec le PVC `data-postgres-0` ; l'application fonctionne et le compteur survit à la suppression de `postgres-0`.
