Installer Prometheus pour monitorer le cluster Minikube
-------------------------------------------------------

Pour comprendre les stratégies de déploiement et mise à jour d’application dans Kubernetes (deployment and rollout strategies) nous allons installer puis mettre à jour une application d’exemple et observer comment sont gérées les requêtes vers notre application en fonction de la stratégie de déploiement choisie.

Pour cette observation on peut utiliser un outil de monitoring. Nous utiliserons ce TP comme prétexte pour installer une des stack les plus populaires et intégrée avec kubernetes : Prometheus et Grafana. Prometheus est un projet de la Cloud Native Computing Foundation.

Prometheus est un serveur de métriques c’est à dire qu’il enregistre des informations précises (de petite taille) sur différents aspects d’un système informatique et ce de façon périodique en effectuant généralement des requêtes vers les composants du système (metrics scraping).


### Installer Prometheus avec Helm

Installez Helm si ce n’est pas déjà fait. Sur Ubuntu : `sudo snap install helm --classic`

*   Créons un namespace pour prometheus et grafana : `kubectl create namespace monitoring`
    
*   Ajoutez le dépot de chart **Prometheus** et **kube-state-metrics**: `helm repo add prometheus-community https://prometheus-community.github.io/helm-charts` puis `helm repo add kube-state-metrics https://kubernetes.github.io/kube-state-metrics` puis mise à jours des dépots helm `helm repo update`.
    
*   Installez ensuite le chart prometheus :
    
```
    helm install \
      --namespace=monitoring \
      --version=19.0.0 \
      --set=service.type=NodePort \
      prometheus \
      prometheus-community/prometheus
```   

### kube-state-metrics et le monitoring du cluster

Le chart officiel installe par défaut en plus de Prometheus, kube-state-metrics qui est une intégration automatique de kubernetes et prometheus.

Une fois le chart installé vous pouvez visualisez les informations dans Lens, dans la premiere section du menu de gauche `Cluster`.

### Déployer notre application d’exemple (goprom) et la connecter à prometheus

Nous allons installer une petite application d’exemple en go.

*   Téléchargez le code de l’application et de son déploiement depuis github: `git clone https://gitlab.com/eps_devops/k8s-deployment-strategies.git`

Nous allons d’abord construire l’image docker de l’application à partir des sources. Cette image doit être stockée dans le registry de minikube pour pouvoir être ensuite déployée dans le cluster. En mode développement Minikube s’interface de façon très fluide avec la ligne de commande Docker grace à quelques variable d’environnement : `minikube docker-env`

*   Changez le contexte de docker cli pour pointer vers minikube avec `eval` et la commande précédente.

réponse:
```
    eval $(minikube docker-env)
    docker system info | grep Name # devrait afficher minikube si le contexte docker est correctement défini.
```    

*   Allez dans le dossier `goprom_app` et “construisez” l’image docker de l’application avec le tag `usernamme/goprom`.

réponse:
```
    cd goprom_app
    docker build -t username/goprom .
```    

*   Allez dans le dossier de la première stratégie `recreate` et ouvrez le fichier `app-v1.yml`. Notez que `image:` est à `username/goprom` et qu’un paramètre `imagePullPolicy` est défini à `Never`. Ainsi l’image sera récupéré dans le registry local du docker de minikube ou sont stockées les images buildées localement plutôt que récupéré depuis un registry distant.
    
*   Appliquez ce déploiement kubernetes:
    

réponse:
```
    cd '../k8s-strategies/1 - recreate'
    kubectl apply -f app-v1.yaml
```   

### Observons notre application et son déploiement kubernetes

*   Explorez le fichier de code go de l’application `main.go` ainsi que le fichier de déploiement `app-v1.yml`. Quelles sont les routes http exposées par l’application ?

réponse:

*   L’application est accessible sur le port `8080` du conteneur et la route `/`.
*   L’application expose en plus deux routes de diagnostic (`probe`) kubernetes sur le port `8086` sur `/live` pour la `liveness` et `/ready` pour la `readiness` (cf [https://kubernetes.io/docs/](https://kubernetes.io/docs/))
*   Enfin, `goprom` expose une route spécialement pour le monitoring Prometheus sur le port `9101` et la route `/metrics`

*   Faites un forwarding de port `Minikube` pour accéder au service `goprom` dans votre navigateur.

réponse:
```
    minikube service goprom
```   

*   Faites un forwarding de port pour accéder au service `goprom-metrics` dans votre navigateur. Quelles informations récupère-t-on sur cette route ?

réponse:
```
    minikube service goprom-metrics
```   

*   Pour tester le service `prometheus-server` nous avons besoin de le mettre en mode NodePort (et non ClusterIP par défaut). Modifiez le service dans Lens pour changer son type.
    
*   Exposez le service avec Minikube (n’oubliez pas de préciser le namespace monitoring).
    
*   Vérifiez que prometheus récupère bien les métriques de l’application avec la requête PromQL : `sum(rate(http_requests_total{app="goprom"}[5m])) by (version)`.
    
*   Quelle est la section des fichiers de déploiement qui indique à prometheus ou récupérer les métriques ?
    

réponse:

    apiVersion: apps/v1
    kind: Deployment
    ...
    spec:
    ...
      template:
        metadata:
    ...
          annotations:
            prometheus.io/scrape: "true"
            prometheus.io/port: "9101"
    

### Installer et configurer Grafana pour visualiser les requêtes

Grafana est une interface de dashboard de monitoring facilement intégrable avec Prometheus. Elle va nous permettre d’afficher un histogramme en temps réel du nombre de requêtes vers l’application.

Créez un secret Kubernetes pour stocker le loging admin de grafana.

    cat <<EOF | kubectl apply -n monitoring -f -
    apiVersion: v1
    kind: Secret
    metadata:
      namespace: monitoring
      name: grafana-auth
    type: Opaque
    data:
      admin-user: $(echo -n "admin" | base64 -w0)
      admin-password: $(echo -n "admin" | base64 -w0)
    EOF
    

Ensuite, installez le chart Grafana en précisant quelques paramètres:

    helm repo add grafana https://grafana.github.io/helm-charts
    helm repo update
    helm install \
      --namespace=monitoring \
      --version=6.45.0 \
      --set=admin.existingSecret=grafana-auth \
      --set=service.type=NodePort \
      --set=service.nodePort=32001 \
      grafana \
      grafana/grafana
    

Maintenant Grafana est installé vous pouvez y acccéder en forwardant le port du service grace à Minikube:

     minikube service grafana -n monitoring
    

Pour vous connectez utilisez, username: `admin`, password: `admin`.

Il faut ensuite connecter Grafana à Prometheus, pour ce faire ajoutez une `DataSource`:

    Name: prometheus
    Type: Prometheus
    Url: http://prometheus-server
    Access: Server
    

Créer une dashboard avec un Graphe. Utilisez la requête prometheus (champ query suivante):

    sum(rate(http_requests_total{app="goprom"}[5m])) by (version)
    

Pour avoir un meilleur aperçu de la version de l’application accédée au fur et à mesure du déploiement, ajoutez `{{version}}` dans le champ `legend`.
