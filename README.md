
1. Installation de Helm
Télécharger et installer Helm

bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

    Télécharge le script d’installation de Helm 3 et l’exécute pour installer Helm sur Ubuntu.

bash
helm version

    Vérifie que Helm est bien installé et affiche la version.

2. Ajout des dépôts Helm

bash
helm repo add stable https://charts.helm.sh/stable

    Ajoute le dépôt “stable” de charts Helm.

bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts

    Ajoute les dépôts pour la stack Prometheus et pour Grafana.

bash
helm repo update

    Met à jour la liste des charts disponibles dans tous les dépôts ajoutés.

bash
helm search repo prometheus-community
helm search repo grafana
helm search repo stable

    Recherche les charts disponibles dans ces dépôts.

3. Installation de Prometheus + Grafana (kube-prometheus-stack)

bash
helm upgrade --install prometheus \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace

    Installe (ou met à jour) la stack kube-prometheus-stack dans le namespace monitoring.

    Crée le namespace monitoring s’il n’existe pas.

bash
kubectl --namespace monitoring get pods -l "release=prometheus"
kubectl get pods -n monitoring

    Liste les pods de la stack Prometheus/Grafana dans le namespace monitoring.

4. Accès à Prometheus et Grafana
Port-forward (accès local depuis la machine)

bash
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090

    Redirige le port 9090 de ta machine vers le service Prometheus dans le cluster.

bash
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80

    Redirige le port 3000 de ta machine vers le service Grafana (port 80 du service).

Changer le type de Service en LoadBalancer

bash
kubectl patch svc prometheus-grafana \
  -n monitoring \
  -p '{"spec": {"type": "LoadBalancer"}}'

    Transforme le service Grafana de ClusterIP en LoadBalancer pour avoir une IP/DNS externe (ELB).

bash
kubectl patch svc prometheus-kube-prometheus-prometheus \
  -n monitoring \
  -p '{"spec": {"type": "LoadBalancer"}}'

    Transforme le service Prometheus en LoadBalancer (quand le pod Prometheus est en Running).

bash
kubectl get svc -n monitoring

    Affiche les Services dans monitoring, leurs types et l’EXTERNAL-IP (LoadBalancer).

Récupérer le mot de passe Grafana

bash
kubectl --namespace monitoring get secrets prometheus-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d ; echo

    Lit le secret prometheus-grafana et décode le mot de passe admin.

5. Gestion des pods et des nœuds (problème “Too many pods”)

bash
kubectl get nodes

    Liste les nœuds du cluster et leur statut.

bash
kubectl describe node ip-192-168-80-13.eu-west-1.compute.internal | grep -i pods

    Affiche la capacité et le nombre de pods autorisés sur ce nœud.

bash
kubectl get pods --all-namespaces -o wide --field-selector spec.nodeName=ip-192-168-80-13.eu-west-1.compute.internal

    Liste tous les pods qui tournent sur ce nœud.

bash
kubectl get pods -A

    Liste tous les pods de tous les namespaces.

bash
kubectl describe pod NOM_DU_POD -n NAMESPACE

    Affiche les détails (events, status) d’un pod (utilisé pour voir les erreurs “FailedScheduling / Too many pods”).

bash
kubectl delete namespace argocd

    Supprime tout le namespace argocd (et tous les pods dedans) pour libérer des places de pods.

bash
helm uninstall prometheus -n monitoring
kubectl delete namespace monitoring

    Désinstalle la release prometheus et supprime le namespace monitoring pour libérer des pods.

6. Installation et utilisation de Argo CD
Installer ArgoCD (manifest officiel)

bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

    Crée le namespace argocd et déploie ArgoCD en utilisant le manifeste officiel.

bash
kubectl get pods -n argocd

    Liste les pods ArgoCD (server, repo-server, application-controller, etc.).

Exposer ArgoCD en LoadBalancer

bash
kubectl get svc -n argocd

    Liste les services ArgoCD (notamment argocd-server).

bash
kubectl patch svc argocd-server \
  -n argocd \
  -p '{"spec": {"type": "LoadBalancer"}}'

    Transforme le service argocd-server en LoadBalancer pour obtenir une URL externe.

Récupérer le mot de passe admin ArgoCD

bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 --decode ; echo

    Lit le secret initial admin et affiche le mot de passe pour l’utilisateur admin.

CLI ArgoCD (sur la machine Ubuntu)

bash
curl -sSL -o argocd-linux-amd64 \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
argocd version

    Télécharge et installe le binaire argocd (CLI), puis vérifie la version.

bash
argocd login <ELB_ARGOCD> --username admin --password <MOT_DE_PASSE> --grpc-web

    Connecte le CLI ArgoCD au serveur ArgoCD exposé via LoadBalancer.

bash
kubectl config get-contexts

    Affiche les contextes kubeconfig (pour savoir le nom du contexte à ajouter dans ArgoCD).

bash
argocd cluster add steven-admin@steven-cluster.eu-west-1.eksctl.io \
  --name steven-cluster

    Ajoute ton cluster EKS “steven-cluster” dans ArgoCD en utilisant le contexte kubeconfig.

7. Manifests Kubernetes pour ton application reddit-clone
Deployment

text
apiVersion: apps/v1
kind: Deployment
metadata:
  name: reddit-clone-deployment
  labels:
    app: reddit-clone-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: reddit-clone-app
  template:
    metadata:
      labels:
        app: reddit-clone-app
    spec:
      containers:
        - name: reddit-clone-app
          image: stephdeve/reddit-clone-pipeline-ci:1.0.0-13
          resources:
            limits:
              cpu: "1"
            requests:
              cpu: "500m"
          ports:
            - containerPort: 3000

    Déploie ton application reddit-clone avec l’image Docker spécifiée.

Service

text
apiVersion: v1
kind: Service
metadata:
  name: reddit-clone-service
  labels:
    app: reddit-clone-app
spec:
  selector:
    app: reddit-clone-app
  ports:
    - port: 3000
      targetPort: 3000
  type: LoadBalancer

    Expose ton application reddit-clone via un service LoadBalancer sur le port 3000.

bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl get pods -n default
kubectl get svc -n default
kubectl describe pod reddit-clone-deployment-... -n default

    Applique les manifests, puis vérifie les pods et services, et inspecte le pod si nécessaire.

8. Pipeline Jenkins (CD GitOps pour reddit-clone)

Jenkinsfile que tu peux mettre dans reddit-gitops pour automatiser la mise à jour du tag d’image :

groovy
pipeline {
    agent any

    environment {
        APP_NAME  = "stephdeve/reddit-clone-pipeline-ci"
        // Tag dynamique basé sur le numéro de build Jenkins
        IMAGE_TAG = "1.0.0-${env.BUILD_NUMBER}"
    }

    stages {
        stage("Cleanup Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Checkout from SCM") {
            steps {
                git branch: 'main',
                    credentialsId: 'github',
                    url: 'https://github.com/stephdeve/reddit-gitops'
            }
        }

        stage("Update the Deployment Image Tag") {
            steps {
                sh """
                    echo 'Avant modification:'
                    cat deployment.yaml | grep image || true

                    sed -i "s|image: .*|image: ${APP_NAME}:${IMAGE_TAG}|g" deployment.yaml

                    echo 'Après modification:'
                    cat deployment.yaml | grep image || true
                """
            }
        }

        stage("Push the changed deployment file to GitHub") {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                    sh """
                        git config --global user.name "stephdeve"
                        git config --global user.email "stephdeve6@gmail.com"
                        git add deployment.yaml || true

                        if git diff --cached --quiet; then
                          echo 'Aucun changement à committer, on continue sans erreur.'
                        else
                          git commit -m "Updated Deployment Manifest to ${IMAGE_TAG}"
                          git push https://${GIT_USER}:${GIT_PASS}@github.com/stephdeve/reddit-gitops main
                        fi
                    """
                }
            }
        }
    }
}


    Pré-requis (cluster EKS, kubectl, Jenkins, Docker, ArgoCD).

    Installation de Helm et des repos.

    Déploiement de Prometheus/Grafana.

    Installation d’ArgoCD et exposition en LoadBalancer.

    Déploiement de l’app reddit-clone (Deployment + Service).

    Pipeline Jenkins (CI pour builder/pusher l’image, CD pour mettre à jour deployment.yaml).

    Notes sur les limitations de pods et les solutions (augmenter les nœuds, supprimer des namespaces, etc.).


Voici une version de README.md compacte et structurée, que tu peux coller dans ton repo GitHub et adapter au besoin.
Reddit Clone – EKS, Argo CD, Prometheus, Grafana, Jenkins

Projet de labo pour apprendre à déployer une application Node.js (“reddit-clone”) sur Amazon EKS avec :

    Helm pour installer Prometheus + Grafana

    Argo CD pour le GitOps

    Jenkins pour la mise à jour automatique du manifeste (deployment.yaml)

1. Prérequis

    Un cluster EKS créé (par exemple via eksctl)

    Une instance Ubuntu (EC2) avec :

        kubectl configuré sur le cluster (contexte steven-admin@steven-cluster.eu-west-1.eksctl.io)

        helm installé

    Un serveur Jenkins (pour CI/CD)

    Un compte GitHub avec les repos :

        reddit-clone (code app + Dockerfile)

        reddit-gitops (manifests Kubernetes + Jenkinsfile)

2. Installation de Helm

Sur l’instance Ubuntu (EC2) :

bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

helm version

3. Ajout des dépôts Helm

bash
helm repo add stable https://charts.helm.sh/stable
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

4. Déploiement Prometheus + Grafana (kube-prometheus-stack)

Installer la stack d’observabilité :

bash
helm upgrade --install prometheus \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace

Vérifier les pods :

bash
kubectl get pods -n monitoring

5. Exposer Grafana et Prometheus
En LoadBalancer

bash
kubectl patch svc prometheus-grafana \
  -n monitoring \
  -p '{"spec": {"type": "LoadBalancer"}}'

kubectl patch svc prometheus-kube-prometheus-prometheus \
  -n monitoring \
  -p '{"spec": {"type": "LoadBalancer"}}'

kubectl get svc -n monitoring

    Utiliser l’EXTERNAL-IP / DNS pour accéder aux UIs depuis le navigateur.

Mot de passe admin Grafana

bash
kubectl --namespace monitoring get secrets prometheus-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d ; echo

    Login : admin

    Mot de passe : valeur affichée par la commande.

6. Installation de Argo CD

Créer le namespace et installer Argo CD :

bash
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

Vérifier :

bash
kubectl get pods -n argocd

Exposer l’UI ArgoCD en LoadBalancer :

bash
kubectl patch svc argocd-server \
  -n argocd \
  -p '{"spec": {"type": "LoadBalancer"}}'

kubectl get svc -n argocd

Mot de passe admin ArgoCD :

bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 --decode ; echo

7. CLI Argo CD et ajout du cluster

Installer le CLI ArgoCD sur Ubuntu :

bash
curl -sSL -o argocd-linux-amd64 \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd

argocd version

Récupérer le nom du contexte kubeconfig :

bash
kubectl config get-contexts

Se connecter au serveur ArgoCD :

bash
argocd login <ELB_ARGOCD> \
  --username admin \
  --password <MOT_DE_PASSE> \
  --grpc-web

Ajouter le cluster EKS dans ArgoCD :

bash
argocd cluster add steven-admin@steven-cluster.eu-west-1.eksctl.io \
  --name steven-cluster

8. Manifests Kubernetes pour l’app reddit-clone
Deployment (deployment.yaml)

text
apiVersion: apps/v1
kind: Deployment
metadata:
  name: reddit-clone-deployment
  labels:
    app: reddit-clone-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: reddit-clone-app
  template:
    metadata:
      labels:
        app: reddit-clone-app
    spec:
      containers:
        - name: reddit-clone-app
          image: stephdeve/reddit-clone-pipeline-ci:1.0.0-13
          resources:
            limits:
              cpu: "1"
            requests:
              cpu: "500m"
          ports:
            - containerPort: 3000

Service (service.yaml)

text
apiVersion: v1
kind: Service
metadata:
  name: reddit-clone-service
  labels:
    app: reddit-clone-app
spec:
  selector:
    app: reddit-clone-app
  ports:
    - port: 3000
      targetPort: 3000
  type: LoadBalancer

Appliquer à la main (pour tester) :

bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

kubectl get pods -n default
kubectl get svc -n default

Dans ArgoCD, créer une Application pointant vers le repo reddit-gitops et le chemin contenant ces fichiers.
9. Pipeline Jenkins – CD GitOps (mise à jour du tag image)

Jenkinsfile dans le repo reddit-gitops :

groovy
pipeline {
    agent any

    environment {
        APP_NAME  = "stephdeve/reddit-clone-pipeline-ci"
        // Tag dynamique basé sur le numéro de build Jenkins
        IMAGE_TAG = "1.0.0-${env.BUILD_NUMBER}"
    }

    stages {
        stage("Cleanup Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Checkout from SCM") {
            steps {
                git branch: 'main',
                    credentialsId: 'github',
                    url: 'https://github.com/stephdeve/reddit-gitops'
            }
        }

        stage("Update the Deployment Image Tag") {
            steps {
                sh """
                    echo 'Avant modification:'
                    cat deployment.yaml | grep image || true

                    sed -i "s|image: .*|image: ${APP_NAME}:${IMAGE_TAG}|g" deployment.yaml

                    echo 'Après modification:'
                    cat deployment.yaml | grep image || true
                """
            }
        }

        stage("Push the changed deployment file to GitHub") {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                    sh """
                        git config --global user.name "stephdeve"
                        git config --global user.email "stephdeve6@gmail.com"
                        git add deployment.yaml || true

                        if git diff --cached --quiet; then
                          echo 'Aucun changement à committer, on continue sans erreur.'
                        else
                          git commit -m "Updated Deployment Manifest to ${IMAGE_TAG}"
                          git push https://${GIT_USER}:${GIT_PASS}@github.com/stephdeve/reddit-gitops main
                        fi
                    """
                }
            }
        }
    }
}

    Dans Jenkins, créer un credential github (username/password ou token) avec accès au repo reddit-gitops.

10. Notes sur la limite de pods

Sur EKS, surtout avec de petites instances, tu peux voir des erreurs :

    Too many pods. No preemption victims found for incoming pod.

Commandes utiles :

bash
kubectl describe node <NOM_DU_NOEUD> | grep -i pods
kubectl get pods -A
kubectl describe pod <NOM_DU_POD> -n <NAMESPACE>

Solutions :

    Supprimer temporairement des namespaces (par ex. argocd ou monitoring) pour libérer des pods.

    Augmenter le nombre de nœuds / le type d’instance du node group dans EKS.

