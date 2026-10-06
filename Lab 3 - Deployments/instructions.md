# LAB 3 - Deployments : déployer, scaler, mettre à jour

**Durée estimée : 60 minutes**

### Objectifs

* Déployer l'application **Countvisit** avec un Deployment et comprendre la chaîne Deployment → ReplicaSet → Pods.
* Observer l'auto-réparation et le scaling.
* Décrire le Deployment en YAML (mode déclaratif).
* Mettre à jour l'application (rolling update), diagnostiquer une mise à jour ratée et revenir en arrière.

**Documentation utile :** [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) · [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/) · [`kubectl rollout`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/)

**L'application :** image `public.ecr.aws/wizetraining/webapp-count:v1`, port **5000** (voir la fiche dans le [README](../README.md)). Sans base de données, elle affiche "Échec de connexion" : c'est normal pour ce lab, la base arrive au Lab 4.

### Le code de l'application est ci-dessous :

<details><summary>Code Webapp-Count v1</summary>

```python
import time
import socket
import logging
import json
import os
import psycopg2
from flask import Flask, Response

app = Flask(__name__)

# Configuration des logs pour voir l'activité en temps réel
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(message)s')

# Paramètres de connexion PostgreSQL (Docker Compose résoudra le hostname 'postgres')
PG_HOST = os.getenv('PG_HOST', 'postgres')
PG_PORT = os.getenv('PG_PORT', '5432')
PG_USER = os.getenv('PG_USER', 'postgres')
PG_PASSWORD = os.getenv('PG_PASSWORD', 'postgres')
PG_DB = os.getenv('PG_DB', 'postgres')

def stream_connection_and_count():
    """
    Générateur Server-Sent Events (SSE).
    Tente de se connecter à PostgreSQL, incrémente le compteur, et envoie 
    l'état en temps réel au navigateur ainsi qu'aux logs.
    """
    retries = 5
    while True:
        # 1. On log en console et on notifie le navigateur de la tentative
        logging.info(f"Tentative de connexion à PostgreSQL (Essais restants : {retries})...")
        yield f"data: {json.dumps({'status': 'loading', 'message': f'Recherche de la base de données... ({retries} essais restants)'})}\n\n"
        
        try:
            # 2. Tentative de connexion
            conn = psycopg2.connect(
                host=PG_HOST, port=PG_PORT, user=PG_USER, password=PG_PASSWORD, dbname=PG_DB,
                connect_timeout=2
            )
            conn.autocommit = True
            with conn.cursor() as cur:
                # On s'assure que la table existe
                cur.execute("CREATE TABLE IF NOT EXISTS hits (id serial PRIMARY KEY, count integer);")
                # On insère la ligne si elle n'existe pas, sinon on l'incrémente
                cur.execute("INSERT INTO hits (id, count) VALUES (1, 1) ON CONFLICT (id) DO UPDATE SET count = hits.count + 1 RETURNING count;")
                count = cur.fetchone()[0]
            conn.close()

            # 3. Succès ! On notifie les logs et le navigateur
            logging.info(f"Connexion réussie ! Visiteur n°{count}")
            yield f"data: {json.dumps({'status': 'success', 'count': count, 'message': 'Connecté à la base de données'})}\n\n"
            break

        except psycopg2.OperationalError as exc:
            # 4. Echec de la tentative
            if retries == 0:
                logging.error("Échec critique : Impossible de joindre PostgreSQL.")
                yield f"data: {json.dumps({'status': 'error', 'message': 'Echec de connexion à la base de données'})}\n\n"
                break
            retries -= 1
            # Pause pour éviter de spammer et pour laisser le temps de voir l'animation côté navigateur
            time.sleep(1.5)

@app.route('/stream')
def stream():
    """Route asynchrone qui envoie les événements au navigateur en temps réel."""
    return Response(stream_connection_and_count(), mimetype='text/event-stream')

@app.route('/')
def hello():
    # L'affichage du HTML est IMMÉDIAT. Il n'y a plus de code bloquant ici.
    hostname = socket.gethostname()

    html_response = f"""
    <!DOCTYPE html>
    <html lang="fr">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Compteur de Visites</title>
        <style>
            body {{ font-family: sans-serif; text-align: center; margin-top: 50px; }}
            .container {{ padding: 20px; border: 1px solid #ccc; border-radius: 8px; display: inline-block; }}
            .hostname {{ font-size: 0.9em; color: #555; margin-top: 20px; }}
            .db-status {{ margin-top: 15px; font-size: 1.1em; }}
            /* Effet visuel clignotant pour le fun */
            .blinking {{ animation: blinker 1s linear infinite; color: #ff9800; font-weight: bold; }}
            @keyframes blinker {{ 50% {{ opacity: 0; }} }}
        </style>
    </head>
    <body>
        <div class="container">
            <h1>Bonjour !</h1>
            
            <p>Vous êtes le visiteur numéro <strong id="count-display">...</strong>.</p>
            <div class="hostname">Vous êtes sur le conteneur : {hostname}</div>
            <div class="db-status" id="db-status-display"><span class="blinking">Initialisation de la connexion... 📡</span></div>
        </div>

        <script>
            // Connexion à la route SSE pour recevoir les événements Python en direct
            const source = new EventSource('/stream');
            
            source.onmessage = function(event) {{
                const data = JSON.parse(event.data);
                const statusDisplay = document.getElementById('db-status-display');
                const countDisplay = document.getElementById('count-display');
                
                if (data.status === 'loading') {{
                    statusDisplay.innerHTML = '<span class="blinking">' + data.message + ' 📡</span>';
                }} else if (data.status === 'success') {{
                    statusDisplay.innerHTML = '<strong style="color:green;">' + data.message + '</strong>';
                    countDisplay.innerText = data.count;
                    source.close(); // Le travail est terminé, on ferme la connexion
                }} else if (data.status === 'error') {{
                    statusDisplay.innerHTML = '<span style="color:red;">' + data.message + '</span>';
                    countDisplay.innerText = 'N/A';
                    source.close();
                }}
            }};
        </script>
    </body>
    </html>
    """
    return html_response

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=True)
```

</details>

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

<details><summary>Correction</summary>

```bash
mkdir -p ~/labs/countvisit && cd ~/labs/countvisit
kubectl delete deployment webapp
kubectl create deployment webapp --image=public.ecr.aws/wizetraining/webapp-count:v1 --replicas=3 --port=5000 \
  --dry-run=client -o yaml > webapp-deployment.yaml
```

</details>

**2.** Modifiez le fichier pour obtenir : replicas `3`, label `app: webapp` (sur le Deployment, le selector et le template), conteneur nommé `webapp`, `containerPort: 5000`. Quelle règle lie `selector.matchLabels` et `template.metadata.labels` ?

<details><summary>Correction</summary>

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
kubectl set image deployment/webapp webapp-count=public.ecr.aws/wizetraining/webapp-count:v2
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
kubectl set image deployment/webapp webapp-count=public.ecr.aws/wizetraining/webapp-count:v999
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
