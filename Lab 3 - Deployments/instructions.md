# LAB 3 - Deployments : déployer, scaler, mettre à jour

**Durée estimée : 60 minutes**

### Objectifs

* Déployer l'application **Countvisit** avec un Deployment et comprendre la chaîne Deployment → ReplicaSet → Pods.
* Observer l'auto-réparation et le scaling.
* Décrire le Deployment en YAML (mode déclaratif).
* Mettre à jour l'application (rolling update), diagnostiquer une mise à jour ratée et revenir en arrière.

**Documentation utile :** [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) · [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/) · [`kubectl rollout`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/)

**L'application :** image `public.ecr.aws/wizetraining/webapp-count:v1`, port **5000** (voir la fiche dans le [README](../README.md)). Sans base de données, elle affiche "Échec de connexion" : c'est normal pour ce lab, la base arrive au Lab 4.

---

## Partie 1 : Premier Deployment en mode impératif

**1.** Créez le namespace `countvisit` et faites-en votre namespace par défaut (vu au Lab 2).

<details><summary>Correction</summary>

```bash
# 1
kubectl create namespace countvisit
kubectl config set-context --current --namespace=countvisit
```

</details>

**2.** Créez un Deployment `webapp` de **2 réplicas** avec l'image Countvisit v1 (option `--port` pour déclarer le port 5000).

<details><summary>Correction</summary>

```bash
# 2
kubectl create deployment webapp --image=public.ecr.aws/wizetraining/webapp-count:v1 --replicas=2 --port=5000
```

</details>

**3.** Quelles ressources ont été créées par cette seule commande ? Comment le nom des Pods est-il construit ?

<details><summary>Correction</summary>

```bash
# 3
kubectl get deployment,replicaset,pods
```

Le Deployment crée un **ReplicaSet** (`webapp-<hash>`) qui crée les **Pods** (`webapp-<hash>-<suffixe>`). Le Deployment gère les versions, le ReplicaSet maintient le nombre de Pods.

</details>

**4.** Supprimez un Pod à la main sur un autre terminal et observez (`-w`) ce qui se passe. Que fait le Deployment ?

<details><summary>Correction</summary>

```bash
# 4
kubectl get pods -w                      # dans un 1er terminal
kubectl delete pod <un-des-pods>         # dans un 2e terminal
```
Le ReplicaSet constate 1 Pod de moins que souhaité et en recrée un immédiatement : c'est la boucle de réconciliation.

</details>

**5.** Passez à **4 réplicas** avec `kubectl scale`, puis consultez les événements du Deployment (`describe`). Revenez à 2.

<details><summary>Correction</summary>

```bash
# 5
kubectl scale deployment webapp --replicas=4
kubectl describe deployment webapp       # section Events : "Scaled up replica set ... to 4"
kubectl scale deployment webapp --replicas=2
```

</details>

---

## Partie 2 : Passer au mode déclaratif

**1.** Créez le dossier `~/labs/countvisit` : vous y rangerez tous les manifestes de l'application. Générez-y le YAML du Deployment (`--dry-run=client -o yaml`) dans `webapp-deployment.yaml`.

details><summary>Correction</summary>

```bash
mkdir -p ~/labs/countvisit && cd ~/labs/countvisit
kubectl delete deployment webapp
kubectl create deployment webapp --image=public.ecr.aws/wizetraining/webapp-count:v1 --replicas=3 --port=5000 \
  --dry-run=client -o yaml > webapp-deployment.yaml
```

</details>

**2.** Modifiez le fichier pour obtenir : replicas `3`, label `app: webapp` (sur le Deployment, le selector et le template), conteneur nommé `webapp`, `containerPort: 5000`. Quelle règle lie `selector.matchLabels` et `template.metadata.labels` ?

details><summary>Correction</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  labels:
    app: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp          # doit correspondre au selector ci-dessus
    spec:
      containers:
      - name: webapp
        image: public.ecr.aws/wizetraining/webapp-count:v1
        ports:
        - containerPort: 5000
```

Le `selector.matchLabels` doit **sélectionner** les labels du template : c'est ainsi que le ReplicaSet reconnaît "ses" Pods.

</details>

**3.** Supprimez d'abord le Deployment créé à la main (pour repartir proprement : un objet créé en impératif n'a pas été "déclaré"), puis appliquez votre fichier.

**4.** Passez à 5 réplicas en modifiant le **fichier** et en le ré-appliquant. Quel est l'avantage par rapport à `kubectl scale` ?

<details><summary>Correction</summary>

```bash
kubectl apply -f webapp-deployment.yaml
# modifier replicas: 5 dans le fichier puis :
kubectl apply -f webapp-deployment.yaml
```
Le fichier reste la source de vérité (versionnable dans Git, relisible, rejouable) alors que `kubectl scale` modifie le cluster sans laisser de trace. Remettez `replicas: 3` à la fin.

</details>

---

## Partie 3 : Mises à jour et retour arrière

**1.** Quelle est la stratégie de mise à jour par défaut ? Quelles valeurs de `maxSurge` et `maxUnavailable` sont appliquées ? (`describe`, ou `kubectl explain deployment.spec.strategy`)

<details><summary>Correction</summary>

```bash
# 1  -> RollingUpdate, 25% max unavailable, 25% max surge
kubectl describe deployment webapp | grep -E "StrategyType|RollingUpdateStrategy"
```

</details>


**2.** Ajoutez dans le manifeste l'annotation `kubernetes.io/change-cause: "Version initiale v1"` (dans `metadata` du Deployment) et ré-appliquez.

<details><summary>Correction</summary>

```bash
# 2
kubectl annotate deployment webapp kubernetes.io/change-cause="Version initiale v1" --overwrite
```
(ou directement dans le YAML : `metadata.annotations."kubernetes.io/change-cause"`.) L'ancienne option `--record` est dépréciée au profit de cette annotation.

</details>

**3.** Passez l'application en **v2** (image `public.ecr.aws/wizetraining/webapp-count:v2`) : modifiez le YAML (ou `kubectl set image`, attention au nom du conteneur) et mettez à jour l'annotation `change-cause`. Pendant la mise à jour, observez dans un autre terminal `kubectl get rs -w` et `kubectl rollout status`. Combien de ReplicaSets existent ensuite ? Que sont devenus les Pods v1 ?

<details><summary>Correction</summary>

```bash
# 3
kubectl set image deployment/webapp webapp=public.ecr.aws/wizetraining/webapp-count:v2
kubectl annotate deployment webapp kubernetes.io/change-cause="Passage en v2" --overwrite
kubectl rollout status deployment/webapp
kubectl get rs
```
Rolling update : le nouveau ReplicaSet monte progressivement pendant que l'ancien descend (jamais plus de 25 % indisponible), **sans coupure**. Il reste 2 ReplicaSets : l'ancien est conservé à 0 Pod pour permettre le rollback.

</details>

**4.** Affichez l'historique des révisions, puis revenez à la v1 avec un rollback.

<details><summary>Correction</summary>

```bash
# 4
kubectl rollout history deployment webapp
kubectl rollout undo deployment webapp                 # revient à la révision précédente
kubectl rollout undo deployment webapp --to-revision=1 # ou à une révision précise
```

</details>

**5.** *Simulez une erreur humaine* : déployez l'image `...webapp-count:v999` (n'existe pas). Que deviennent les Pods ? Les anciens Pods v1 sont-ils arrêtés ? Réparez en revenant à la révision précédente.

<details><summary>Correction</summary>

```bash
# 5
kubectl set image deployment/webapp webapp=public.ecr.aws/wizetraining/webapp-count:v999
kubectl get pods                    # nouveaux Pods en ErrImagePull / ImagePullBackOff
kubectl rollout status deployment/webapp --timeout=30s   # bloqué
kubectl describe pod <pod-en-erreur> | tail
kubectl rollout undo deployment webapp
```
Les anciens Pods v1 sont **conservés** tant que les nouveaux ne sont pas `Ready` : le service n'est pas coupé. C'est pourquoi le rolling update est sûr.

</details>

**6.** *Bonus.* Passez la stratégie à `Recreate`, déployez la v2 et comparez le comportement des Pods avec le rolling update. Quel est le risque ? Remettez ensuite `RollingUpdate`.

<details><summary>Correction</summary>

```bash
# 6  (strategy: { type: Recreate } dans le YAML)
```
Avec `Recreate`, tous les anciens Pods sont supprimés **avant** de créer les nouveaux : coupure de service assurée. À réserver aux applications qui ne supportent pas deux versions simultanées.

</details>

**Nettoyage / état de départ du Lab 4 :** le Deployment `webapp` doit tourner en **v1**, 3 réplicas, avec `RollingUpdate`. Mettez à jour `webapp-deployment.yaml` pour qu'il reflète cet état.

> **État attendu en fin de lab :** `kubectl get deployment webapp` affiche `3/3`, le fichier `~/labs/countvisit/webapp-deployment.yaml` correspond à ce qui tourne.
