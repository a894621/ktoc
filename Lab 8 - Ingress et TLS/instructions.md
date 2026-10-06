# LAB 8 - Ingress et TLS

**Durée estimée : 50 minutes**

### Objectifs

* Publier l'application par **nom de domaine** avec un **Ingress** (routage HTTP de niveau 7).
* Router plusieurs applications par **chemin** avec réécriture d'URL.
* Activer **HTTPS** avec un certificat stocké dans un Secret TLS.

**Documentation utile :** [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) · [Ingress NGINX - rewrite](https://kubernetes.github.io/ingress-nginx/examples/rewrite/) · [Ingress NGINX - TLS](https://kubernetes.github.io/ingress-nginx/user-guide/tls/)

**Prérequis :** fin du Lab 7 (Service `webapp` dans `countvisit`). 

Un **Ingress** est un objet qui décrit des règles de routage (hôte, chemin → Service). Il ne fait rien tout seul : c'est l'**Ingress Controller** (ici NGINX) qui lit ces règles et configure un reverse-proxy, lui-même exposé par un Service `LoadBalancer`. On passe ainsi de *un LoadBalancer par application* à *un seul point d'entrée pour toutes*.

> **Pour information :** le projet Ingress NGINX est en fin de vie (maintenance arrêtée Mars 2026) ; le successeur recommandé par la communauté est l'API **Gateway**. Les concepts vus ici (hôte, chemin, TLS, backend) se retrouvent dans Gateway API.

---

## Partie 1 : Le point d'entrée (10 min)

### Installez le contrôleur Nginx via Helm.

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer
```

Vérifiez que le service du contrôleur est actif (`Running`)

**1.** Retrouvez l'`IngressClass` et l'adresse du contrôleur (Service du namespace `ingress-nginx`). Notez l'IP : appelons-la `<INGRESS_IP>`. *(Si `EXTERNAL-IP` est `<pending>`, utilisez l'IP d'un worker et le NodePort `http`.)*

<details><summary>Correction</summary>

```bash
kubectl get ingressclass
```

</details>

**2.** Interrogez `http://<INGRESS_IP>/` avec `curl`. Que répond-il, et pourquoi ?

<details><summary>Correction</summary>

```bash
kubectl get svc -n ingress-nginx
curl -i http://<INGRESS_IP>/          # 404 Not Found (nginx) : aucune règle ne correspond
```
Le 404 vient du contrôleur lui-même (backend par défaut) : il fonctionne mais ne connaît encore aucune règle.

</details>

**3.** Votre Service `webapp` n'a plus besoin d'être exposé directement : repassez-le en `ClusterIP` (modifiez `webapp-service.yaml` et appliquez).

<details><summary>Correction</summary>

```bash
# webapp-service.yaml : type: ClusterIP
kubectl apply -f webapp-service.yaml
```

</details>

---

## Partie 2 : Routage par nom de domaine (15 min)

**1.** Écrivez `webapp-ingress.yaml` : un Ingress `webapp` qui envoie `countvisit.lab.test` (chemin `/`) vers le Service `webapp`, port 80. N'oubliez pas `ingressClassName`.

<details><summary>Correction</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp
spec:
  ingressClassName: nginx            # nom relevé avec "kubectl get ingressclass"
  rules:
  - host: countvisit.lab.test
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp
            port:
              number: 80
```

</details>

**2.** Appliquez et vérifiez que l'Ingress a reçu une adresse.

<details><summary>Correction</summary>

```bash
kubectl apply -f webapp-ingress.yaml
kubectl get ingress webapp
```

</details>

**3.** Testez. Le domaine n'existe pas en DNS : au choix, utilisez `curl --resolve countvisit.lab.test:80:<INGRESS_IP> http://countvisit.lab.test/` ou ajoutez une ligne `<INGRESS_IP> countvisit.lab.test` à votre fichier `hosts` (`/etc/hosts` ; `C:\Windows\System32\drivers\etc\hosts`) puis ouvrez la page dans le navigateur. Que se passe-t-il avec `curl http://<INGRESS_IP>/` sans en-tête `Host` ?

<details><summary>Correction</summary>

```bash
curl --resolve countvisit.lab.test:80:<INGRESS_IP> http://countvisit.lab.test/
```
Sans en-tête `Host` correspondant, le contrôleur répond encore 404 : le **nom d'hôte** fait partie de la règle. En production, on crée un enregistrement DNS vers l'IP du contrôleur au lieu de modifier `hosts`.

</details>

---

## Partie 3 : Routage par chemin et réécriture (15 min)

Deux applications peuvent partager un même nom d'hôte et être distinguées par le chemin.

**1.** Déployez un second site : Deployment `static-site` (image nginx ECR, 1 réplica) et son Service `ClusterIP` sur le port 80 (commandes impératives `create deployment` et `expose`).

<details><summary>Correction</summary>

```bash
kubectl create deployment static-site --image=public.ecr.aws/wizetraining/nginx:latest --port=80
kubectl expose deployment static-site --port=80
```

</details>

**2.** Créez un Ingress `static-site` pour l'hôte `apps.lab.test` : le chemin `/static` doit mener au Service `static-site`.

<details><summary>Correction</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: static-site
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: apps.lab.test
    http:
      paths:
      - path: /static
        pathType: Prefix
        backend:
          service:
            name: static-site
            port:
              number: 80
```
```bash
kubectl apply -f static-site-ingress.yaml
```

</details>

**3.** Testez `http://apps.lab.test/static`. Si vous obtenez une erreur 404, regardez les logs du Pod `static-site` : quel chemin reçoit-il ? Corrigez avec l'annotation `nginx.ingress.kubernetes.io/rewrite-target: /`.

<details><summary>Correction</summary>

```bash
curl --resolve apps.lab.test:80:<INGRESS_IP> http://apps.lab.test/static
kubectl logs deploy/static-site | tail -3
```
Sans réécriture, nginx reçoit la requête `/static` qu'il ne connaît pas → 404. L'annotation `rewrite-target: /` remplace le chemin par `/` avant de transmettre à l'application. (Pour conserver le reste du chemin, utilisez des groupes de capture, cf. documentation.)

</details>

---

## Partie 4 : HTTPS avec un certificat auto-signé (10 min)

**1.** Générez un certificat auto-signé (`openssl req -x509 ...`) pour `countvisit.lab.test` (clé `tls.key`, certificat `tls.crt`, valable 365 jours).

<details><summary>Correction</summary>

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -subj "/CN=countvisit.lab.test" \
  -addext "subjectAltName=DNS:countvisit.lab.test"
```

</details>

**2.** Créez un Secret de type `tls` nommé `countvisit-tls` à partir de ces deux fichiers.

<details><summary>Correction</summary>

```bash
kubectl create secret tls countvisit-tls --cert=tls.crt --key=tls.key
```

</details>

**3.** Modifiez l'Ingress `webapp` pour activer TLS pour cet hôte avec ce Secret. Testez en HTTPS (`curl -k`, ou navigateur en acceptant l'alerte). Que répond maintenant l'accès en HTTP ? Observez le certificat présenté (`curl -vk`).

<details><summary>Correction</summary>

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -subj "/CN=countvisit.lab.test" \
  -addext "subjectAltName=DNS:countvisit.lab.test"

kubectl create secret tls countvisit-tls --cert=tls.crt --key=tls.key
```
```yaml
# webapp-ingress.yaml : ajouter sous spec
  tls:
  - hosts:
    - countvisit.lab.test
    secretName: countvisit-tls
```
```bash
kubectl apply -f webapp-ingress.yaml
curl -k --resolve countvisit.lab.test:443:<INGRESS_IP> https://countvisit.lab.test/
curl -i --resolve countvisit.lab.test:80:<INGRESS_IP> http://countvisit.lab.test/    # 308 -> redirection vers https
curl -vk --resolve countvisit.lab.test:443:<INGRESS_IP> https://countvisit.lab.test/ 2>&1 | grep -E "subject|issuer"
```
Le navigateur affiche une alerte car le certificat est auto-signé (émetteur = sujet). En production, on automatise l'émission de certificats de confiance avec **cert-manager** (Let's Encrypt par exemple). Supprimez les fichiers `tls.key` / `tls.crt` de votre dossier de travail à la fin.

</details>

> **État attendu en fin de lab :** `https://countvisit.lab.test` affiche Countvisit via l'Ingress, `http://apps.lab.test/static` affiche la page nginx, le Service `webapp` est en `ClusterIP`. Conservez `webapp-ingress.yaml`.
