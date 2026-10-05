# LAB 2 - Pods, namespaces et labels

**Durée estimée : 60 minutes**

### Objectifs

* Cloisonner avec les **namespaces** et fixer un namespace par défaut.
* Créer un **Pod** de manière impérative puis déclarative, et l'inspecter (`describe`, `logs`, `exec`, `port-forward`).
* Comprendre le réseau "à plat" de Kubernetes.
* Organiser et filtrer les objets avec les **labels**, **selectors** et **annotations**.

**Documentation utile :** [Pods](https://kubernetes.io/docs/concepts/workloads/pods/) · [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) · [Labels et selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) · [Annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/)

Images : `nginx` ou `public.ecr.aws/wizetraining/nginx:latest` et `busybox` ou `public.ecr.aws/wizetraining/busybox:latest`.

---

## Partie 1 : Opérations sur les Pods (Approche Impérative)

L'approche **impérative** permet d'exécuter des actions directes en ligne de commande.

**1. Lancer un serveur Web**
Lancez un Pod nommé `webserver` (image `nginx`) dans le namespace `default`. (En cas de limitation de registre, utilisez `public.ecr.aws/wizetraining/nginx:latest`).
* Récupérez son adresse IP et le nœud sur lequel il est hébergé.
* Décrivez le Pod

<details><summary>Correction</summary>

```bash
kubectl run webserver --image nginx
kubectl get pods -o wide
kubectl describe pod webserver
```

</details>

**2. Isolation par Namespace et Unicité**
Créez un namespace `k8s-lab`. Lancez-y un pod nommé `tools` (image `busybox`, commande `sleep 1000`).
Tentez ensuite de créer un second pod nommé `tools` dans ce même namespace. Que se passe-t-il ?

<details><summary>Correction</summary>

```bash
kubectl create ns k8s-lab
kubectl run tools --image busybox -n k8s-lab -- sleep 1000

# Tentative de doublon (va échouer) :
kubectl run tools --image busybox -n k8s-lab -- sleep 1000
```

*Erreur `AlreadyExists` : dans un même namespace, le nom d'une ressource doit être unique.*

</details>

**3. Communication Inter-Pods**
Depuis le pod `tools` (namespace `k8s-lab`), tentez de "pinger" l'IP du pod `webserver` (namespace `default`).
*L'isolation administrative par namespace bloque-t-elle la communication réseau des Pods inter namespace ?*

<details><summary>Correction</summary>

```bash
# Récupération de l'IP du webserver
kubectl get pod webserver -o wide

# Ping depuis tools
kubectl exec -it -n k8s-lab tools -- ping -c 3 <IP_WEBSERVER>

```

*Le ping fonctionne : par défaut, tous les pods peuvent communiquer entre eux quel que soit leur namespace.*

</details>

---

## Partie 3 : Approche Déclarative (YAML)

Définissons les ressources dans des fichiers YAML pour l'approche "Infrastructure as Code".

**1. Générer et appliquer un manifeste**
Générez le fichier `pod-tools2.yml` sans créer le pod directement (nom : `tools2`, image : `busybox`, commande : `sleep 1500`, namespace : `k8s-lab`). Appliquez-le ensuite.

```bash
# Génération sans création (--dry-run=client)
kubectl run tools2 --image busybox -n k8s-lab --dry-run=client -o yaml -- sleep 1500 > pod-tools2.yml

# Application du manifeste
kubectl apply -f pod-tools2.yml

# Vérification
kubectl get pods -n k8s-lab

```

---

## Partie 4 : Labels, Selectors et Annotations

Les **Labels** organisent les objets K8s et les **Selectors** permettent de les filtrer.

**1. Labelliser un Namespace et déployer des Pods étiquetés**

1. Créez un namespace `dev` (Si ce n'est pas encore fait) et ajoutez-lui le label `formation=k8s`.
2. Créez un pod `nginx1` dans `dev` avec le label `app=frontend`.
3. Ajoutez-lui le label `rel=beta` en ligne de commande.
4. Lancez un second pod `nginx2` dans `dev` avec les labels `app=frontend` et `rel=stable`.

<details><summary>Correction</summary>

```bash
# Namespace + label
kubectl create ns dev
kubectl label ns dev formation=k8s

```

</details>

Fichier `nginx-labels.yaml` :

<details><summary>Correction</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx1
  namespace: dev
  labels:
    app: frontend
spec:
  containers:
  - name: nginx
    image: nginx

```

```bash
kubectl apply -f nginx-labels.yaml

# Ajout du label manquant sur nginx1
kubectl label -n dev pod nginx1 rel=beta

# Création rapide de nginx2
kubectl run nginx2 --image=nginx -n dev --labels="app=frontend,rel=stable"

```

</details>

**2. Filtrage avec Selectors**
Dans le namespace `dev` :

* Affichez les pods en faisant apparaître les colonnes `app` et `rel`.
* Listez uniquement les pods avec `rel=stable`.
* Listez les pods qui *possèdent* la clé `rel`, puis ceux qui *ne la possèdent pas*.

<details><summary>Correction</summary>

```bash
# Afficher les labels en colonnes
kubectl get po -n dev -L app,rel

# Filtrer par valeur exacte
kubectl get po -n dev -l rel=stable

# Filtrer par présence / absence de clé
kubectl get po -n dev -l rel
kubectl get po -n dev -l '!rel'

```

</details>

**3. Annotations**
Ajoutez l'annotation `[wizetraining.com/equipe-responsable=](https://wizetraining.com/equipe-responsable=)"les K8S dev"` au pod `nginx1`.

<details><summary>Correction</summary>

```bash
kubectl annotate pod -n dev nginx1 wizetraining.com/equipe-responsable="les K8S dev"
kubectl describe pod -n dev nginx1

```

</details>

---

## Partie 5 : Ordonnancement avec NodeSelector

Utilisons les labels pour orienter le placement d'un Pod sur un nœud précis.

**1. Labelliser un Nœud et contraindre un Pod**

1. Ajoutez le label `disk=ssd` sur l'un de vos worker nodes.
2. Déployez un pod `pod-ssd` dans `dev` (image `nginx`) en exigeant qu'il s'exécute sur un nœud équipé de `disk=ssd`.

<details><summary>Correction</summary>

```bash
# Labelliser le worker
kubectl label node <nom-du-worker> disk=ssd

```

</details>

Fichier `pod-ssd.yaml` :

<details><summary>Correction</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-ssd
  namespace: dev
  labels:
    app: backend
spec:
  nodeSelector:
    disk: "ssd"
  containers:
  - name: nginx
    image: nginx

```

```bash
kubectl apply -f pod-ssd.yaml
kubectl get pod pod-ssd -n dev -o wide

```

</details>

---

## Partie 6 : Nettoyage des Ressources

**1. Suppression ciblée**

1. Supprimez le pod `pod-ssd` par son nom.
2. Supprimez tous les pods restants du namespace `dev` ayant le label `app=frontend` en une seule commande.

<details><summary>Correction</summary>

```bash
# Suppression directe
kubectl delete pod -n dev pod-ssd

# Suppression de masse via selector
kubectl delete po -n dev -l app=frontend

```

</details>