# flask_app — TP CI/CD avec Jenkins, Docker et Kubernetes

## But du TP

Mettre en place une chaîne **CI/CD complète** autour d'une petite application Flask :

1. **Tester** automatiquement le code à chaque build ;
2. **Construire** une image Docker de l'application ;
3. **Publier** cette image sur Docker Hub ;
4. **Déployer** l'application sur un cluster Kubernetes (minikube).

Le tout est piloté par un pipeline Jenkins décrit dans le `Jenkinsfile`.

```
GitHub ──► Jenkins ──► Tests ──► docker build ──► Docker Hub ──► kubectl apply ──► minikube
```

## L'application

Une API Flask minimale qui écoute sur le port **5000** :

| Route                | Réponse            |
|----------------------|--------------------|
| `/` et `/hello/`     | `Hello World!`     |
| `/hello/<username>`  | `Hello <username>!` |

## Structure du projet

```
.
├── app.py              # Application Flask
├── test.py             # Tests unitaires (unittest)
├── requirements.txt    # Dépendances Python
├── Dockerfile          # Image Docker (python:3.11-slim, utilisateur non-root)
├── Jenkinsfile         # Pipeline CI/CD (test → build → push → deploy)
└── k8s/
    ├── deployment.yml  # Deployment Kubernetes (1 réplica)
    └── service.yml     # Service NodePort (port 30080)
```

## Prérequis

- Python 3.11+
- Docker
- Jenkins (ici lancé dans un conteneur Docker)
- minikube et kubectl
- Un compte Docker Hub et un compte GitHub

## Démarrage rapide (au quotidien)

Une fois tout installé et configuré (voir sections suivantes), voici l'ordre de démarrage :

```bash
# 1. Docker doit tourner
sudo systemctl start docker

# 2. Démarrer le cluster Kubernetes
minikube start --driver=docker
kubectl get nodes                      # doit afficher "Ready"

# 3. Démarrer Jenkins (conteneur déjà créé)
docker start jenkins
# → ouvrir http://localhost:8080

# 4. Lancer le pipeline : job "flask-pipeline" → "Build Now"

# 5. Accéder à l'application déployée
curl http://$(minikube ip):30080/
```

Pour tout arrêter :

```bash
docker stop jenkins
minikube stop
```

## 1. Lancer l'application en local

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

python test.py      # lancer les tests
python app.py       # démarrer l'app sur http://localhost:5000
```

Vérification :

```bash
curl http://localhost:5000/
curl http://localhost:5000/hello/Tino
```

## 2. Lancer l'application avec Docker

```bash
docker build -t flask_hello:latest .
docker run --rm -p 5000:5000 flask_hello:latest
```

## 3. Démarrer minikube

```bash
minikube start --driver=docker
kubectl get nodes        # le nœud doit être "Ready"
```

Déploiement manuel (sans Jenkins) :

```bash
kubectl apply -f k8s/deployment.yml
kubectl apply -f k8s/service.yml
curl http://$(minikube ip):30080/
```

## 4. Configurer Jenkins

### 4.1 Lancer Jenkins

Jenkins tourne dans un conteneur qui a accès au Docker de l'hôte (nécessaire pour `docker build`
et pour les étapes `docker.image(...).inside`) :

```bash
docker run -d --name jenkins -p 8080:8080 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $(which docker):/usr/bin/docker:ro \
  --group-add $(stat -c %g /var/run/docker.sock) \
  jenkins/jenkins:lts
```

Ce `docker run` ne se fait **qu'une seule fois**. Ensuite :

```bash
docker start jenkins      # démarrer
docker stop jenkins       # arrêter
docker restart jenkins    # redémarrer (ex. après installation de plugins)
docker logs -f jenkins    # suivre les logs
docker ps                 # vérifier que le conteneur "jenkins" est "Up"
```

Les données (jobs, credentials, historique des builds) sont conservées dans le volume
`jenkins_home`, même si le conteneur est supprimé.

### 4.2 Premier démarrage de Jenkins

1. Ouvrir **http://localhost:8080**
2. Récupérer le mot de passe initial :
   ```bash
   docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
   ```
3. Choisir **Install suggested plugins**
4. Créer le compte administrateur
5. Installer les plugins manquants dans *Manage Jenkins → Plugins → Available plugins* :
   **Docker Pipeline**, **Credentials Binding**, **Plain Credentials** (Pipeline et Git sont
   déjà inclus dans les plugins suggérés), puis redémarrer Jenkins.

Vérifier que Jenkins peut utiliser Docker :

```bash
docker exec jenkins docker ps
```

### 4.3 Créer les credentials

Dans *Manage Jenkins → Credentials → System → Global credentials → Add Credentials* :

| ID                      | Type                   | Contenu                                   |
|-------------------------|------------------------|-------------------------------------------|
| `dockerhub_credentials` | Username with password | Identifiant et mot de passe / token Docker Hub |
| `kubeconfig`            | Secret file            | Fichier kubeconfig de minikube (voir ci-dessous) |

Générer le kubeconfig (certificats inclus dans le fichier) :

```bash
kubectl config view --minify --flatten > kubeconfig-jenkins
```

Charger ce fichier dans le credential `kubeconfig`, puis **supprimer le fichier local** : il
contient la clé d'accès au cluster et ne doit jamais être commité.

> À refaire après chaque `minikube delete`, car les certificats du cluster changent.

### 4.4 Créer le job

1. *New Item* → nom `flask-pipeline` → type **Pipeline**
2. *Definition* : **Pipeline script from SCM**
3. *SCM* : **Git** — URL `https://github.com/Fitiafenohaja/flask_app.git`
4. *Branch* : `*/main` — *Script Path* : `Jenkinsfile`
5. Enregistrer puis **Build Now**

## 5. Le pipeline

| Étape                 | Ce qu'elle fait |
|-----------------------|-----------------|
| **Test Flask**        | Installe les dépendances et lance `test.py` dans un conteneur `python:3.11` |
| **Build Docker Image**| Construit l'image `flask_hello:latest` |
| **Push Docker Hub**   | Pousse l'image `fitiafenohajaun/flask_hello_jenkins:latest` |
| **Deploy Kubernetes** | Applique `k8s/` puis redémarre le déploiement pour récupérer la nouvelle image |

L'étape de déploiement utilise `kubectl` dans un conteneur `alpine/k8s`, connecté au réseau
Docker `minikube` pour joindre l'API du cluster (`192.168.49.2:8443`).

Une fois le build terminé, l'application est accessible sur :

```bash
minikube service flask-hello-service --url
# ou
curl http://$(minikube ip):30080/
```

## Dépannage

- **`connection refused` sur `192.168.49.2:8443`** : minikube n'est pas démarré → `minikube start`.
- **minikube ne démarre pas (`apiserver process never appeared`)** : cluster corrompu →
  `minikube delete && minikube start`, puis régénérer le credential `kubeconfig`.
- **Jenkins inaccessible sur http://localhost:8080** : vérifier `docker ps` ; si le conteneur
  est arrêté → `docker start jenkins`, sinon consulter `docker logs jenkins`.
- **`permission denied` sur `/var/run/docker.sock` dans Jenkins** : le conteneur Jenkins doit
  appartenir au groupe du socket Docker (`--group-add`).
- **Le pod ne se met pas à jour** : l'image utilise le tag `latest`, c'est pourquoi le pipeline
  fait un `kubectl rollout restart`.
