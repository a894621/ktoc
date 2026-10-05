# LAB 4 - Services et découverte de services

**Durée estimée : 60 minutes**

### Objectifs

* Comprendre pourquoi les Pods ont besoin d'un **Service** (IP stable, load balancing, DNS).
* Utiliser les types `ClusterIP`, `NodePort` et `LoadBalancer`.
* Relier Countvisit à **PostgreSQL** grâce au DNS interne.
* Diagnostiquer un Service qui n'a pas d'endpoints.

**Documentation utile :** [Services](https://kubernetes.io/docs/concepts/services-networking/service/) · [DNS pour Services et Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/) · [Connecter une application à un Service](https://kubernetes.io/docs/tutorials/services/connect-applications-service/)

**Prérequis :** Lab 3 terminé (Deployment `webapp`, v1, 3 réplicas, namespace `countvisit` par défaut). Vous travaillez dans `~/labs/countvisit`.

---

## Partie 1 : Service ClusterIP (15 min)

Les Pods sont éphémères : leur IP change à chaque recréation. Un **Service** donne une IP et un nom DNS stables devant un ensemble de Pods choisis par **selector de labels**.

**1.** Relevez les IP de vos Pods `webapp`. Supprimez-en un : son IP change-t-elle ?

<details><summary>Correction</summary>

```bash
# 1
kubectl get pods -o wide
kubectl delete pod <un-pod>; kubectl get pods -o wide     # nouvelle IP
```

</details>


**2.** Écrivez `webapp-service.yaml` : un Service `webapp` de type `ClusterIP`, qui expose le port **80** et redirige vers le port **5000** des Pods `app=webapp`. Appliquez-le.

<details><summary>Correction</summary>

```bash
# 2  webapp-service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  type: ClusterIP
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 5000
```
```bash
kubectl apply -f webapp-service.yaml
```

</details>


**3.** Quels sont les *endpoints* du Service ? Supprimez un Pod et surveillez la liste : que constatez-vous ?

<details><summary>Correction</summary>

```bash
# 3
kubectl describe service webapp                # ligne Endpoints : 3 IP:5000
kubectl get endpointslices -l kubernetes.io/service-name=webapp
kubectl delete pod <un-pod>; kubectl get endpointslices -l kubernetes.io/service-name=webapp -o yaml | grep -A1 addresses
```
Le Service suit automatiquement les Pods qui correspondent au selector (EndpointSlice mis à jour en continu).

</details>


**4.** Testez le Service **depuis l'intérieur du cluster** avec un Pod temporaire (image busybox, `wget`). Essayez les noms `webapp`, `webapp.countvisit` et `webapp.countvisit.svc.cluster.local`. Lequel fonctionne depuis un autre namespace (ex. `default`) ?

<details><summary>Correction</summary>

```bash
# 4
kubectl run tmp --rm -it --restart=Never --image=public.ecr.aws/wizetraining/busybox:latest -- wget -qO- http://webapp
kubectl run tmp --rm -it --restart=Never -n default --image=public.ecr.aws/wizetraining/busybox:latest -- wget -qO- http://webapp.countvisit
```
Dans le même namespace, `webapp` suffit. Depuis un autre namespace, il faut au minimum `webapp.countvisit`. Le nom complet est `<service>.<namespace>.svc.cluster.local`.

</details>

---

## Partie 2 : NodePort et LoadBalancer (10 min)

`ClusterIP` n'est joignable que depuis le cluster. Pour y accéder depuis votre poste :

* **NodePort** : ouvre un port (30000-32767) sur **tous** les nœuds.
* **LoadBalancer** : demande une IP externe à l'infrastructure (ici fournie par le cluster de lab).

**1.** Passez le Service en `NodePort` (modifiez le YAML et ré-appliquez ou créez un nouveau service avec un nouveau nom). Quel est le port alloué ? Ouvrez l'application sur `http://<IP-d'un-worker>:<nodePort>`. Essayez avec l'IP d'un autre worker.

<details><summary>Correction</summary>

```bash
# 1 : type: NodePort dans webapp-service.yaml
kubectl apply -f webapp-service.yaml
kubectl get service webapp            # 80:3xxxx/TCP
kubectl get nodes -o wide             # IP des workers (192.168.56.11-13)
# http://192.168.56.11:3xxxx  et  http://192.168.56.12:3xxxx : même application
```

</details>

**2.** Passez-le en `LoadBalancer`. Que contient la colonne `EXTERNAL-IP` ? Ouvrez l'application sur cette IP, port 80. Rafraîchissez plusieurs fois la page : que change le nom du conteneur affiché ?

<details><summary>Correction</summary>

```bash
# 2 : type: LoadBalancer
kubectl apply -f webapp-service.yaml
kubectl get service webapp -w         # EXTERNAL-IP passe de <pending> à une IP
```
Le Service `LoadBalancer` inclut un NodePort. La page répartit les requêtes entre les 3 Pods : le "conteneur" affiché varie. À ce stade la page affiche toujours "Échec de connexion", il manque la base de données.

</details>

> Laissez le Service en `LoadBalancer` (ou `NodePort` si l'IP externe reste `<pending>`) : vous en aurez besoin pour voir l'application dans les labs suivants.

---

## Partie 3 : PostgreSQL et découverte de services 

L'application cherche par défaut un serveur nommé **`postgres`** sur le port 5432, avec l'utilisateur `postgres` et le mot de passe `postgres` (variables `PG_*`).

**1.** Créez `postgres-deployment.yaml` : Deployment `postgres`, 1 réplica, image `public.ecr.aws/wizetraining/postgres:16-alpine`, label `app: postgres`, port 5432. L'image PostgreSQL refuse de démarrer sans la variable d'environnement `POSTGRES_PASSWORD` : donnez-lui la valeur attendue par l'application. Appliquez, puis vérifiez dans les logs que la base est prête.

<details><summary>Correction</summary>

```yaml
# postgres-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  labels:
    app: postgres
spec:
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
        - name: POSTGRES_PASSWORD
          value: postgres
```

```bash
kubectl apply -f postgres-deployment.yaml
kubectl logs deploy/postgres | tail -3        # "database system is ready to accept connections"
```

</details>

**2.** Créez `postgres-service.yaml` : quel **nom** et quel **type** de Service faut-il pour que l'application trouve sa base sans configuration ? Pourquoi ce type ?

<details><summary>Correction</summary>

```yaml
# postgres-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres          # c'est ce nom que l'application résout (PG_HOST par défaut)
spec:
  type: ClusterIP         # seule l'application doit joindre la base : pas d'exposition externe
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
```

```bash
kubectl apply -f postgres-deployment.yaml -f postgres-service.yaml
```

</details>

**3.** Rechargez la page de l'application plusieurs fois. Que se passe-t-il ? Vérifiez aussi les logs d'un Pod `webapp`. Depuis un Pod `webapp`, quelle IP est résolue pour `postgres` ?

<details><summary>Correction</summary>


```bash
# 3 : la page affiche "Connecté à la base de données" et le compteur s'incrémente à chaque rechargement
kubectl logs deploy/webapp | tail             # "Connexion réussie ! Visiteur n°..."
kubectl exec deploy/webapp -- python -c "import socket; print(socket.gethostbyname('postgres'))"
```

</details>

**4.** Supprimez le Pod PostgreSQL, attendez qu'il soit recréé, rechargez la page. **Que devient le compteur ? Pourquoi ?**

<details><summary>Correction</summary>


```bash
# 4
kubectl delete pod -l app=postgres
```
Après la recréation, le compteur **repart à 1** : un conteneur est éphémère et ses données disparaissent avec lui. C'est le point de départ du Lab 6 (stockage persistant). Pas besoin de redémarrer l'application : elle se reconnecte à chaque page.

</details>

---

## Partie 4 : Diagnostic - un Service sans endpoints

C'est l'une des pannes les plus courantes. Créez le Service suivant, puis tentez de joindre le port 80 comme en Partie 1 :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-broken
spec:
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 5000
```

**1.** Pourquoi la requête échoue-t-elle ? Quelle commande le montre immédiatement ?

**2.** Corrigez-le, puis supprimez-le.

<details><summary>Correction</summary>

```bash
kubectl describe service webapp-broken      # Endpoints: <none>
kubectl get pods --show-labels              # les Pods ont app=webapp, pas app=web-app
```
Le selector ne correspond à aucun Pod : le Service n'a aucune cible. Corrigez `app: webapp`, ré-appliquez, puis `kubectl delete service webapp-broken`.

**Réflexe de diagnostic :** `Endpoints: <none>` = vérifier le selector du Service contre les labels des Pods, puis que les Pods sont `Ready`.

</details>

> **État attendu en fin de lab :** Service `webapp` (LoadBalancer ou NodePort) + Deployment `postgres` + Service `postgres` ; la page Countvisit affiche "Connecté à la base de données" et un compteur. Vous avez constaté que ce compteur n'est pas durable.
