# LAB 9 - Helm : installer et packager une application

**Durée estimée : 70 minutes**

### Objectifs

* Utiliser la CLI Helm : dépôts, recherche, inspection des valeurs, simulation (`template`), installation.
* Gérer le cycle de vie d'une release : `upgrade`, `history`, `rollback`, `uninstall`.

**Documentation utile :** [Helm - Démarrage rapide](https://helm.sh/docs/intro/quickstart/) · [Commandes Helm](https://helm.sh/docs/helm/) · [Guide des templates](https://helm.sh/docs/chart_template_guide/getting_started/) · [Artifact Hub](https://artifacthub.io/)

**Prérequis :** fin du Lab 8, binaire `helm` installé sur votre poste.

Helm est le "gestionnaire de paquets" de Kubernetes. Un **chart** est un paquet : des templates YAML + un fichier `values.yaml` de paramètres. Une **release** est une installation d'un chart dans le cluster, avec son historique de révisions.

---

## Partie 1 : Exploration et ajout de dépôts (Repositories)

Helm utilise un système de catalogues distants (similaire à `apt` sous Ubuntu). Par défaut, Helm ne connaît aucun dépôt. Nous allons utiliser le dépôt très populaire de Bitnami (qui maintient d'excellentes charts communautaires).

1. **Vérifier vos dépôts actuels :**
Commencez par lister les dépôts auxquels votre client Helm a actuellement accès.

<details>
<summary>Correction</summary>

```bash
helm repo list
# Si c'est votre première utilisation, Helm vous répondra sûrement :
# Error: no repositories to show

```

</details>

2. **Ajouter le dépôt Bitnami :**
Ajoutez le dépôt officiel de Bitnami et nommez-le `bitnami`.
*(L'URL est : `https://charts.bitnami.com/bitnami`)*

<details>
<summary>Correction</summary>

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami

# Vérifiez à nouveau :
helm repo list

```

</details>

3. **Mettre à jour le catalogue :**
C'est une étape cruciale (souvent oubliée par les débutants). Comment forcer Helm à récupérer la liste des dernières versions disponibles sur ce dépôt ?

<details>
<summary>Correction</summary>

```bash
helm repo update bitnami
# Ou simplement `helm repo update` pour mettre à jour tous vos dépôts d'un coup.

```

</details>

4. **Rechercher une application :**
Cherchez la chart `nginx` dans les dépôts que vous venez d'ajouter.

<details>
<summary>Correction</summary>

```bash
helm search repo bitnami/nginx

```

</details>

5. **Rechercher une application avec les version:**
Cherchez toute les version de la chart `nginx` dans les dépôts que vous venez d'ajouter.

<details>
<summary>Correction</summary>

```bash
helm search repo bitnami/nginx -l

```

</details>

---

## Partie 2 : Inspection de la Chart (Le mode "Safe")

Avant de déployer un package communautaire sur votre cluster, il est de bonne pratique de comprendre ce qu'il contient et quelles sont les variables configurables.

1. **Télécharger la chart localement (Pull) :**
Au lieu de l'installer directement, téléchargez la chart `bitnami/nginx` sur votre machine pour l'analyser.

<details>
<summary>Correction</summary>

```bash
# Cette commande télécharge un fichier .tgz dans votre dossier courant
helm pull bitnami/nginx

```

</details>

2. **Désarchiver et examiner le code source :**
Décompressez l'archive téléchargée et ouvrez le dossier généré dans votre IDE (VS Code, vi, etc.) pour observer la structure (`Chart.yaml`, `values.yaml`, dossier `templates/`).

<details>
<summary>Correction</summary>

```bash
# Remplacer x.x.x par la version téléchargée
tar xvf nginx-*.tgz

# Examinez le dossier
ls -la nginx/

```

</details>

3. **Extraire les valeurs par défaut :**
Sans forcément télécharger la chart, Helm permet de lire le fichier `values.yaml` directement depuis le dépôt distant. Affichez ces valeurs, et sauvegardez-les dans un fichier local nommé `my-values.yaml`.

<details>
<summary>Correction</summary>

```bash
# Pour juste afficher à l'écran :
helm show values bitnami/nginx

# Pour afficher ET sauvegarder dans un fichier local en même temps :
helm show values bitnami/nginx | tee my-values.yaml
# (Ou plus simplement : helm show values bitnami/nginx > my-values.yaml)

```

</details>

4. **Modifier la configuration :**
Ouvrez votre fichier `my-values.yaml`. Cherchez la ligne `replicaCount: 1` et changez-la par `replicaCount: 2`.

---

## Partie 3 : Simulation et Déploiement

1. **Le rendu à blanc (Templating) :**
Avant d'envoyer quoi que ce soit à Kubernetes, demandez à Helm de générer le YAML final en fusionnant la Chart avec votre fichier `my-values.yaml`. C'est l'outil de débogage ultime.

<details>
<summary>Correction</summary>

```bash
# Si vous utilisez le dossier local téléchargé à l'étape 2 :
helm template mon-test-nginx ./nginx -f my-values.yaml

# Si vous utilisez le dépôt distant (plus courant) :
helm template mon-test-nginx bitnami/nginx -f my-values.yaml

# Observez la sortie : vous verrez le YAML pur qui sera envoyé à Kubernetes.
# Cherchez la section Deployment, vous devriez voir 'replicas: 2'.

```

</details>

2. **L'installation finale :**
Déployez l'application avec les caractéristiques suivantes :
* Nom de la release : `my-web-server`
* Chart : `bitnami/nginx`
* Namespace : `web-ns` (faites en sorte que Helm le crée automatiquement s'il n'existe pas)
* Fichier de valeurs : Votre `my-values.yaml`



<details>
<summary>Correction</summary>

```bash
helm install my-web-server bitnami/nginx -n web-ns --create-namespace -f my-values.yaml

```

</details>

---

## Partie 4 : Audit, Dépannage et Rollback

L'application est en ligne. Voyons comment Helm nous aide à l'exploiter au quotidien.

1. **Lister les Releases :**
Comment vérifier que votre application est bien reconnue par Helm dans le namespace `web-ns` ?

<details>
<summary>Correction</summary>

```bash
helm list -n web-ns
# Vous verrez la révision (REVISION: 1), le statut (deployed) et la version de l'app.

```

</details>

2. **Faire le lien avec Kubernetes :**
Helm a créé de nombreux objets K8s. Quelle commande `kubectl` permet de lister *uniquement* les ressources (Pods, Services, Deployments) appartenant à votre release précise, sans être pollué par le reste du cluster ?

<details>
<summary>Correction</summary>

Helm ajoute automatiquement des labels standards à tout ce qu'il déploie. Le label `app.kubernetes.io/instance` contient le nom de votre release.

```bash
kubectl get all -n web-ns -l app.kubernetes.io/instance=my-web-server

```

</details>

3. **L'Audit complet (La commande "Sauve-qui-peut") :**
Imaginez que vous arriviez sur un cluster que vous ne connaissez pas. Comment récupérer TOUTES les informations d'une release (Les valeurs utilisées par votre collègue, les manifestes finaux et les notes de déploiement) ?

<details>
<summary>Correction</summary>

```bash
helm get all my-web-server -n web-ns

```

</details>

4. **Provoquer une panne et faire un Rollback :**
Simulons une erreur humaine. Mettez à jour votre release avec une image qui n'existe pas, puis utilisez le rollback pour réparer la production.

<details>
<summary>Correction</summary>

**Étape A : Casser l'application (Upgrade)**

```bash
# On force l'utilisation d'un tag d'image invalide
helm upgrade my-web-server bitnami/nginx -n web-ns --set image.tag="cette-version-n-existe-pas"

# Vérifiez que le pod est en erreur (ImagePullBackOff)
kubectl get pods -n web-ns

```

**Étape B : Constater la nouvelle révision**

```bash
helm history my-web-server -n web-ns
# Vous êtes maintenant à la Révision 2.

```

**Étape C : Le Rollback sauveur**

```bash
# On demande à Helm de revenir à la Révision 1 (l'état stable)
helm rollback my-web-server 1 -n web-ns

# Vérifiez l'historique : Une Révision 3 a été créée, reprenant l'état de la Rev 1 !
helm history my-web-server -n web-ns

# Les pods redémarrent avec la bonne image instantanément.
kubectl get pods -n web-ns

```

</details>

**Nettoyage avant le prochain lab :**
Maintenant que vous savez à quel point c'est facile, désinstallez toute l'application et ses composants en une seule ligne :

```bash
helm uninstall my-web-server -n web-ns
kubectl delete namespace web-ns

```