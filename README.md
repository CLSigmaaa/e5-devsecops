# Maquette Kubernetes - Déploiement de 3 Applications (Django incluse)

Ce dépôt présente une maquette fonctionnelle de migration vers Kubernetes pour un client ayant 3 applications à exposer sur les ports 80, 8080 et 9090. L'une d'elles est critique, développée avec Django.

## 🎯 Objectifs

- Containeriser les 3 applications avec Docker, en **optimisant les images**.
- Déployer 3 applications dans **Kubernetes** avec :
  - Un **Deployment**
  - Un **Service**
  - **Au moins un Ingress**
- Séparer **chaque composant** dans un manifeste distinct.
- Permettre un **accès externe** aux applications.

## Structure du dépôt

```
k8s-maquette/
│
├── apps/
│   ├── app1/ (port 80)
│   ├── app2/ (port 8080)
│   └── app3/ (port 9090)
│
├── k8s/
│   ├── app1-deployment.yaml
│   ├── app1-service.yaml
│   ├── app2-deployment.yaml
│   ├── app2-service.yaml
│   ├── app3-deployment.yaml
│   ├── app3-service.yaml
│   ├── app3-ingress.yaml
```
## Installation et configuration de Minikube

### 1. Installer Kubernetes

#### Prérequis

Avant de commencer l'installation, assurez-vous d'avoir:

- 2 CPU ou plus
- 2GB de mémoire libre
- 20GB d'espace disque libre
- Une connexion Internet
- Un gestionnaire de conteneurs ou de machines virtuelles comme Docker, QEMU, Hyperkit, Hyper-V, KVM, Parallels, Podman, VirtualBox, ou VMware Fusion/Workstation

#### Installation de Minikube

Pour installer la dernière version stable de Minikube sur Linux x86-64 avec une installation binaire:

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

### 2. Créer le cluster avec Docker

Vérifiez les versions de Kubernetes compatibles avec Minikube:

```bash
minikube config defaults kubernetes-version
```

Créez votre cluster Minikube avec Docker:

```bash
# Testé le 28 novembre 2024
minikube start --listen-address=0.0.0.0 --memory=max --cpus=max --kubernetes-version=v1.31.0
minikube start --listen-address=0.0.0.0 --memory=max --cpus=max --kubernetes-version=v1.32.0
```

Vérifiez que tout est correctement installé:

```bash
minikube kubectl cluster-info
minikube status
```

Affichez la version de Kubernetes utilisée:

```bash
kubectl version
```

Vérifiez l'IP du cluster Minikube:

```bash
minikube ip
```

Pour supprimer un cluster:

```bash
minikube delete --purge
```

### 3. Activer l'autocomplétion

Documentation sur l'autocomplétion Minikube: [https://minikube.sigs.k8s.io/docs/commands/completion/](https://minikube.sigs.k8s.io/docs/commands/completion/)

Documentation complète sur l'autocomplétion: [https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#enable-shell-autocompletion](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#enable-shell-autocompletion)

Ajoutez ce code dans votre fichier `.bashrc` pour activer l'autocomplétion:

```bash
if command -v minikube &>/dev/null
then
  eval "$(minikube completion bash)"
fi
source /etc/bash_completion
source <(minikube completion bash)
alias kubectl="minikube kubectl --"
alias k="minikube kubectl --"
alias k="kubectl"
complete -F __start_kubectl k
source <(kubectl completion bash)

# Au lieu de faire un kubectl, on peut directement utiliser la lettre k
k get all --all-namespaces
```

## Étapes de réalisation

### 1. Applications
- App1 : Application Django sur le port 80
- App2 : Application Flask sur le port 8080
- App3 : Application Django sur le port 9090

Pour chaque application, il y aura un Dockerfile optimisé.

### 2. Création des images Docker
- Cloner les dépôts des applications dans le répertoire `apps/`.

```bash
git clone https://mon-repository.git apps/app1
```

- Rédaction d'un .dockerignore pour chaque application afin d'ignorer les fichiers inutiles lors de la construction de l'image Docker.

```dockerfile
# Exclure les fichiers Python inutiles
*.pyc
*.pyo
*.pyd
__pycache__/

# Exclure les fichiers de configuration locaux
*.env
.env.*
*.log

# Exclure les dossiers spécifiques
.git/
.vscode/
.idea/
node_modules/
dist/
build/

# Exclure les fichiers temporaires
*.swp
*.bak
*.tmp
.DS_Store
Thumbs.db
```

- Création d'un Dockerfile optimisé pour chaque application.

Dockerfile pour Django (app1 et app3):

```dockerfile
FROM python:3.9-slim

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# Set working directory
WORKDIR /app

# Install Python dependencies
COPY requirements.txt .
RUN pip install --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Collect static files and apply migrations
RUN python manage.py collectstatic --no-input && \
    python manage.py makemigrations && \
    python manage.py migrate

# Expose port and set command to run the application
EXPOSE 80
CMD ["gunicorn", "--config", "gunicorn-cfg.py", "core.wsgi"]
```

Dockerfile pour Flask (app2):

```dockerfile
FROM python:3.9-alpine

# Set the working directory
WORKDIR /app

# Copy the requirements file
COPY requirements.txt .

# Install the dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy the application files
COPY . .

# Expose the port the app runs on
EXPOSE 8080

# Define the command to run the application
CMD ["python", "run.py"]
```

- Construction des images Docker pour chaque application.

```bash
cd apps/app1
docker build -t app1:latest .
cd ../app2
docker build -t app2:latest .
cd ../app3
docker build -t app3:latest .
```

- Vérification des images Docker créées.

```bash
docker images
```

- Envoie des images vers le registre Docker.

Dans un premier temps, il faut les taguer avec le nom du registre Docker parce que par défaut, elles sont taguées avec le nom de l'hôte local.
```bash
docker tag app1:latest <docker-registry>/app1:latest
docker tag app2:latest <docker-registry>/app2:latest
docker tag app3:latest <docker-registry>/app3:latest
```

- Ensuite, il faut se connecter à Docker Hub.

```bash
docker login <docker-registry>
```

- Puis, il faut envoyer les images taguées vers le registre Docker.

```bash
docker push <docker-registry>/app1:latest
docker push <docker-registry>/app2:latest
docker push <docker-registry>/app3:latest
```

### 3. Déploiement dans Kubernetes
Pour chaque application, il y a un manifeste de déploiement et un manifeste de service.

Le manifeste de déploiement sert à déployer l'application dans Kubernetes, tandis que le manifeste de service permet d'exposer l'application à l'extérieur du cluster.

#### 3.1. Déploiement de l'application Django (app1)

Deployment pour app1 (app1-deployment.yaml):
Ce manifeste crée un déploiement pour l'application Django (app1) avec une réplique et expose le port 80.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
      - name: app1
        image: clsigmaaa/app1:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 80
```
Service pour app1 (app1-service.yaml):
Ce manifeste crée un service de type NodePort pour exposer l'application Django (app1) sur le port 80.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: app1
spec:
  selector:
    app: app1
  type: NodePort
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30001
```

#### 3.2. Déploiement de l'application Flask (app2)
Deployment pour app2 (app2-deployment.yaml):
Ce manifeste crée un déploiement pour l'application Flask (app2) avec une réplique et expose le port 8080.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app2
  template:
    metadata:
      labels:
        app: app2
    spec:
      containers:
      - name: app1
        image: clsigmaaa/app2:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8080

```
Service pour app2 (app2-service.yaml):
Ce manifeste crée un service de type LoadBalancer pour exposer l'application Flask (app2) sur le port 8080.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: app2
spec:
  selector:
    app: app2
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
```

#### 3.3. Déploiement de l'application Django (app3)
Deployment pour app3 (app3-deployment.yaml):
Ce manifeste crée un déploiement pour l'application Django (app3) avec une réplique et expose le port 9000.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app3
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app3
  template:
    metadata:
      labels:
        app: app3
    spec:
      containers:
      - name: app3
        image: clsigmaaa/app3:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 9000
```
Service pour app3 (app3-service.yaml):
Ce manifeste crée un service de type LoadBalancer pour exposer l'application Django (app3) sur le port 9000.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: app2
spec:
  selector:
    app: app2
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 9000
      targetPort: 9000
```

#### 3.4  Appliquer les manifestes

Pour appliquer les manifestes de déploiement et de service, utilisez la commande suivante:

```bash
kubectl apply -f ./k8s/
```

La sortie de la commande `kubectl get all` vous montrera les pods, les services et les déploiements créés.

```bash
kubectl get all
NAME                        READY   STATUS    RESTARTS   AGE
pod/app1-559d57b7c4-ssd26   1/1     Running   0          118s
pod/app2-768c6d58db-6kqmf   1/1     Running   0          118s
pod/app3-6bf87d8fd5-7rxmq   1/1     Running   0          118s

NAME                 TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
service/app1         NodePort       10.107.62.76     <none>        80:30001/TCP     118s
service/app2         LoadBalancer   10.102.213.245   <pending>     8080:32305/TCP   118s
service/app3         LoadBalancer   10.107.10.161    <pending>     9000:30254/TCP   118s
service/kubernetes   ClusterIP      10.96.0.1        <none>        443/TCP          2m32s

NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/app1   1/1     1            1           118s
deployment.apps/app2   1/1     1            1           118s
deployment.apps/app3   1/1     1            1           118s

NAME                              DESIRED   CURRENT   READY   AGE
replicaset.apps/app1-559d57b7c4   1         1         1       118s
replicaset.apps/app2-768c6d58db   1         1         1       118s
replicaset.apps/app3-6bf87d8fd5   1         1         1       118s
```

### 4. Accès externe aux applications

Plusieurs services ont été utilisés pour exposer les applications:
- **NodePort** pour app1 (port 30001)
- **LoadBalancer** pour app2 (port 8080) et app3 (port 9000)

Le NodePort permet d'accéder à l'application via l'adresse IP du nœud et le port spécifié (30001 dans ce cas).

Le LoadBalancer permet d'accéder à l'application via une adresse IP externe, mais cela dépend de la configuration de votre environnement Kubernetes (par exemple, si vous utilisez un fournisseur de cloud qui prend en charge les LoadBalancers).

Le Port Forwarding dans Kubernetes est une fonctionnalité qui permet de rediriger un port  localhost vers un pod dans le cluster Kubernetes. 

#### Commandes pour le port forwarding

Pour accéder à chaque application via le port forwarding, exécutez les commandes suivantes:

```bash
kubectl port-forward service/app1 8081:80
kubectl port-forward service/app2 8082:8080
kubectl port-forward service/app3 8083:9000
```

Cela redirige les ports locaux 8081, 8082, et 8083 vers les ports des services correspondants dans le cluster Kubernetes.

#### Accès aux applications

- **App1 (Django)** : [http://localhost:30001](http://localhost:30001)
- **App2 (Flask)** : [http://localhost:8080](http://localhost:8080)
- **App3 (Django)** : [http://localhost:9000](http://localhost:9000)

Vous pouvez maintenant tester vos applications en local via les URL ci-dessus.

```bash
kubectl port-forward service/app1 30001:80
kubectl port-forward service/app2 8080:8080
kubectl port-forward service/app3 9000:9000
```

Pour y accéder en même temps, il suffit de rajouter `&` à la fin de chaque commande:

```bash
kubectl port-forward service/app1 30001:80 &
kubectl port-forward service/app2 8080:8080 &
kubectl port-forward service/app3 9000:9000 &
```

### 5. Applications

- App1 (Django) ![alt text](image.png)
- App2 (Flask) ![alt text](image-1.png)
- App3 (Django) ![alt text](image-2.png)

### 6. Architecture

![alt text](image-3.png)