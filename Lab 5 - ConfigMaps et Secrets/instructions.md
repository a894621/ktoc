# LAB 5 - ConfigMaps et Secrets

**Durée estimée : 50 minutes**

### Objectifs

* Séparer la **configuration** et les **mots de passe** de l'image et des manifestes.
* Utiliser un **ConfigMap** (variables d'environnement) et un **Secret** (variables et fichier monté).
* Comprendre les limites d'un Secret (base64 n'est pas du chiffrement) et la propagation des changements.

**Documentation utile :** [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/) · [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) · [Utiliser un Secret dans un Pod](https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/) · [Variables d'environnement depuis un ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)

**Prérequis :** fin du Lab 4 (`webapp` + `postgres`, namespace `countvisit`, dossier `~/labs/countvisit`).

**L'application "Entreprise" :** image `public.ecr.aws/wizetraining/webapp-count-secure:v1`. Elle lit `PG_HOST`, `PG_PORT`, `PG_USER`, `PG_DB` dans l'environnement, mais lit le **mot de passe dans un fichier** (chemin donné par `PG_PASSWORD_FILE`, défaut `/etc/app/secrets/pg_password`). Voir la fiche dans le [README](../README.md).

### Le code de l'application (Version Entreprise) est ci-dessous :

<details>
<summary>Code de l'appli webapp-count-secure:v1 </summary>

Code de l'image `public.ecr.aws/wizetraining/webapp-count-secure:v1`

```python
import time
import socket
import logging
import json
import os
import psycopg2
from flask import Flask, Response, jsonify, request

# --------------------------------------------------
# Configuration du logging (Format standard Entreprise)
# --------------------------------------------------
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger("app")

app = Flask(__name__)

# --------------------------------------------------
# Configuration PostgreSQL dynamique (12-factor compliant)
# --------------------------------------------------
PG_HOST = os.getenv('PG_HOST', 'postgres')
PG_PORT = os.getenv('PG_PORT', '5432')
PG_USER = os.getenv('PG_USER', 'postgres')
PG_DB = os.getenv('PG_DB', 'postgres')

logger.info(f"Configuration PostgreSQL: {PG_HOST}:{PG_PORT} (User: {PG_USER}, DB: {PG_DB})")

# --------------------------------------------------
# Gestion du mot de passe via Docker/K8s Secret
# --------------------------------------------------
def get_pg_password():
    secret_path = os.getenv("PG_PASSWORD_FILE", "/etc/app/secrets/pg_password")
    try:
        with open(secret_path, 'r') as secret_file:
            password = secret_file.read().strip()
            # On ne logue JAMAIS le mot de passe, juste le succès
            logger.info(f"Mot de passe PostgreSQL chargé avec succès depuis {secret_path}.")
            return password
    except IOError:
        logger.warning(
            f"Secret introuvable dans {secret_path}. "
            "Tentative de connexion sans mot de passe."
        )
        return ""

# --------------------------------------------------
# Sondes Kubernetes (Liveness & Readiness)
# --------------------------------------------------
@app.route('/healthz')
def healthz():
    """Liveness Probe: Vérifie si le conteneur tourne."""
    return jsonify({"status": "alive"}), 200

@app.route('/readyz')
def readyz():
    """
    Readiness Probe: Normalement on teste la DB ici. 
    Mais pour ton lab (où on veut voir l'UI chercher la DB en temps réel), 
    on doit dire à K8s que l'UI est prête à être affichée.
    """
    return jsonify({"status": "ready", "ui": "available"}), 200

# --------------------------------------------------
# Fonction métier : Connexion et incrément (SSE)
# --------------------------------------------------
def stream_connection_and_count():
    """Générateur Server-Sent Events (SSE)."""
    password = get_pg_password()
    retries = 5
    
    while True:
        logger.info(f"Tentative de connexion à PostgreSQL (Essais restants : {retries})...")
        yield f"data: {json.dumps({'status': 'loading', 'message': f'Recherche de la base de données... ({retries} essais restants)'})}\n\n"
        
        try:
            conn = psycopg2.connect(
                host=PG_HOST, port=PG_PORT, user=PG_USER, password=password, dbname=PG_DB,
                connect_timeout=2
            )
            conn.autocommit = True
            with conn.cursor() as cur:
                cur.execute("CREATE TABLE IF NOT EXISTS hits (id serial PRIMARY KEY, count integer);")
                cur.execute("INSERT INTO hits (id, count) VALUES (1, 1) ON CONFLICT (id) DO UPDATE SET count = hits.count + 1 RETURNING count;")
                count = cur.fetchone()[0]
            conn.close()

            logger.info(f"Connexion réussie ! Visiteur n°{count}")
            yield f"data: {json.dumps({'status': 'success', 'count': count, 'message': 'Connecté à PostgreSQL (Sécurisé)'})}\n\n"
            break

        except psycopg2.OperationalError as exc:
            if retries == 0:
                logger.error(f"Échec critique PostgreSQL : {exc}")
                yield f"data: {json.dumps({'status': 'error', 'message': 'Echec de connexion à PostgreSQL'})}\n\n"
                break
            retries -= 1
            time.sleep(1.5)

@app.route('/stream')
def stream():
    """Route asynchrone pour les événements en temps réel."""
    return Response(stream_connection_and_count(), mimetype='text/event-stream')

# --------------------------------------------------
# Route Principale (Chargement immédiat)
# --------------------------------------------------
@app.route('/')
def hello():
    hostname = socket.gethostname()

    html_response = f"""
    <!DOCTYPE html>
    <html lang="fr">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Compteur Sécurisé (PostgreSQL)</title>
        <style>
            body {{ font-family: -apple-system, sans-serif; text-align: center; margin-top: 50px; background: #f8f9fa; }}
            .container {{ padding: 30px; border: 1px solid #e9ecef; border-radius: 12px; display: inline-block; background: #fff; box-shadow: 0 4px 6px rgba(0,0,0,0.05); }}
            .hostname {{ font-size: 0.9em; color: #6c757d; margin-top: 25px; padding: 10px; background: #f8f9fa; border-radius: 6px; }}
            .db-status {{ margin-top: 20px; font-size: 1.1em; }}
            .blinking {{ animation: blinker 1s linear infinite; color: #ff9800; font-weight: bold; }}
            @keyframes blinker {{ 50% {{ opacity: 0; }} }}
        </style>
    </head>
    <body>
        <div class="container">
            <h1>Application d'Entreprise 🚀</h1>
            <p>Vous êtes le visiteur numéro <strong id="count-display" style="font-size: 1.2em;">...</strong>.</p>
            <div class="hostname">Instance : <code>{hostname}</code></div>
            <div class="db-status" id="db-status-display"><span class="blinking">Initialisation de la connexion... 📡</span></div>
        </div>

        <script>
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
                    source.close();
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
    app.run(host="0.0.0.0", port=5000)
```

</details>

---

## Partie 1 : Le problème (5 min)

Affichez la définition du Deployment `postgres` en YAML et retrouvez le mot de passe. Que constatez vous ? Et quel est le problème avec cette méthode de configuration ?

<details><summary>Correction</summary>

```bash
kubectl get deployment postgres -o yaml | grep -B1 -A1 POSTGRES_PASSWORD
```
Le mot de passe est en clair dans le manifeste, donc dans Git, dans l'historique, et visible par quiconque lit le Deployment. De plus l'application utilise le compte superutilisateur `postgres`. Nous allons corriger les deux.

</details>

---

## Partie 2 : ConfigMap - la configuration non sensible (15 min)

**1.** Écrivez `webapp-configmap.yaml` : un ConfigMap `webapp-config` avec les clés `PG_HOST=postgres`, `PG_PORT=5432` et `PG_DB=app_db`. Appliquez-le et vérifiez son contenu.

*(Astuce : `kubectl create configmap ... --from-literal=... --dry-run=client -o yaml` génère le YAML.)*

<details><summary>Correction</summary>

```yaml
# webapp-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-config
data:
  PG_HOST: postgres
  PG_PORT: "5432"
  PG_DB: app_db
```
```bash
kubectl apply -f webapp-configmap.yaml
kubectl describe configmap webapp-config
```

</details>

**2.** Modifiez le Deployment `webapp` pour :
* utiliser l'image `public.ecr.aws/wizetraining/webapp-count-secure:v1`,
* charger **toutes** les clés du ConfigMap en variables d'environnement (`envFrom`).

<details><summary>Correction</summary>

Dans `webapp-deployment.yaml`, section du conteneur `webapp` :
```yaml
      containers:
      - name: webapp-count
        image: public.ecr.aws/wizetraining/webapp-count-secure:v1
        ports:
        - containerPort: 5000
        envFrom:
        - configMapRef:
            name: webapp-config
```

```bash
kubectl apply -f webapp-deployment.yaml
```

</details>

**3.** Vérifiez que les variables s'affiche à l'intérieur du conteneur `webapp-count`.

*Dans le Navigateur vous constaterez que l'application a perdu accès à la base de donnée. Vous finaliserez l'accès à la base en Partie 4 : tant que le Secret n'existe pas, l'application affichera une erreur de connexion.*

<details><summary>Correction</summary>

```bash
kubectl exec -it deploy/webapp -- env | grep ^PG_
```

Résultat attendu

```bash
PG_DB=app_db
PG_HOST=postgres
PG_PORT=5432
```

</details>

---

## Partie 3 : Secret - les identifiants (15 min)

**1.** Créez un fichier local `secrets/pg_password.txt` contenant un mot de passe de votre choix (**sans retour à la ligne final**, option `-n` de `echo`). Il ne doit jamais être commité dans Git.

<details><summary>Correction</summary>

```bash
mkdir -p secrets
echo -n "ChangeMe-Postgres123" > secrets/pg_password.txt
```

</details>

**2.** Créez un Secret `pg-secret` de type `kubernetes.io/basic-auth` contenant :
* `username` : `app_user`
* `password` : le contenu du fichier précédent (option `--from-file`).

*(Le type `basic-auth` impose ces deux clés ; l'opérateur PostgreSQL du Lab 10 les attend exactement ainsi.)*

<details><summary>Correction</summary>

```bash
kubectl create secret generic pg-secret \
  --type=kubernetes.io/basic-auth \
  --from-literal=username=app_user \
  --from-file=password=./secrets/pg_password.txt
```

</details>

**3.** Affichez le Secret en YAML. Pouvez-vous retrouver le mot de passe ? Que peut-on en conclure sur la sécurité d'un Secret ?

<details><summary>Correction</summary>

```bash
kubectl get secret pg-secret -o yaml
kubectl get secret pg-secret -o jsonpath='{.data.password}' | base64 -d; echo
```
Les valeurs sont **encodées en base64, pas chiffrées** : toute personne autorisée à lire le Secret l'obtient en clair. La protection repose sur : le **RBAC** (qui peut lire les Secrets), le chiffrement d'etcd au repos et, en production, un gestionnaire externe (Vault, External Secrets...). Et surtout : ne jamais mettre un Secret dans Git.

</details>

---

## Partie 4 : Brancher le tout (15 min)

**1.** Modifiez `postgres-deployment.yaml` pour qu'il n'ait plus de mot de passe en clair :
* `POSTGRES_USER` et `POSTGRES_PASSWORD` viennent des clés `username` et `password` de `pg-secret` (`secretKeyRef`),
* `POSTGRES_DB` vient de la clé `PG_DB` du ConfigMap `webapp-config` (`configMapKeyRef`).


<details><summary>Correction</summary>

```yaml
# postgres-deployment.yaml (section conteneur)
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
```

</details>

**2.** Modifiez le Deployment `webapp` :
* `PG_USER` vient de la clé `username` de `pg-secret`,
* le Secret est **monté comme un volume** dans `/etc/app/secrets`, la clé `password` devant apparaître comme le fichier `pg_password`.

<details><summary>Correction</summary>

```yaml
# webapp-deployment.yaml (spec.template.spec)
      containers:
      - name: webapp-count
        image: public.ecr.aws/wizetraining/webapp-count-secure:v1
        ports:
        - containerPort: 5000
        envFrom:
        - configMapRef:
            name: webapp-config
        env:
        - name: PG_USER
          valueFrom:
            secretKeyRef:
              name: pg-secret
              key: username
        volumeMounts:
        - name: pg-password
          mountPath: /etc/app/secrets
          readOnly: true
      volumes:
      - name: pg-password
        secret:
          secretName: pg-secret
          items:
          - key: password
            path: pg_password
```

</details>

**3.** Appliquez, attendez la fin des rollouts, puis vérifiez dans le navigateur le titre "Application d'Entreprise" et la connexion à la base. Vérifiez dans le Pod `webapp` : le fichier `/etc/app/secrets/pg_password` existe, mais aucune variable d'environnement ne contient le mot de passe.

<details><summary>Correction</summary>

```bash
kubectl apply -f postgres-deployment.yaml -f webapp-deployment.yaml
kubectl rollout status deployment/postgres
kubectl rollout status deployment/webapp

kubectl exec deploy/webapp -- cat /etc/app/secrets/pg_password; echo
kubectl exec deploy/webapp -- env | grep ^PG_      # PG_HOST, PG_PORT, PG_DB, PG_USER ; pas de mot de passe
```
Le compteur repart à 1 : PostgreSQL a été recréé (toujours sans volume, Lab 6).

</details>

**4.** *Bonus.* Changez `PG_PORT` dans le ConfigMap puis relisez la variable dans un Pod existant. Le Pod voit-il la nouvelle valeur ? Qu'en est-il d'un fichier monté depuis un Secret ou ConfigMap ? Comment forcer la prise en compte ?

<details><summary>Correction</summary>

**Bonus :** une variable d'environnement est figée au démarrage du conteneur : il faut `kubectl rollout restart deployment/webapp`. Les fichiers montés (volume) sont, eux, mis à jour automatiquement après quelques dizaines de secondes (sauf avec `subPath`). Dans les deux cas, l'application doit savoir relire sa configuration.

</details>

> **État attendu en fin de lab :** l'application "Entreprise" fonctionne ; plus aucun mot de passe en clair dans vos manifestes ; `pg-secret` (basic-auth) et `webapp-config` existent. Fichiers : `webapp-configmap.yaml` ajouté, deux Deployments mis à jour.
