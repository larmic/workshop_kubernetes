# Ein Einstieg in Kubernetes

Dies ist ein kleiner Lightning Talk, der den Einstieg in Kubernetes ermöglichen soll.

## Requirements

* Docker
* Docker Desktop mit aktivierter Kubernetes Engine
* [kubectl](https://kubernetes.io/docs/reference/kubectl/) für die CLI-Kommunikation mit dem Cluster
* [Helm Package Manager](https://helm.sh/) für die Ingress-Installation
* Optional: [K9S](https://k9scli.io/) oder [Open Lens](https://github.com/MuhammedKalkan/OpenLens)

```shell
# Richten context für diesen Talk finden
➜  ~ kubectl config get-contexts
CURRENT   NAME                 CLUSTER              AUTHINFO             NAMESPACE
          docker-desktop       docker-desktop       docker-desktop       
*         k3s-pi5              k3s-pi5              k3s-pi5              
          neusta:skills:dev    neusta:skills:dev    neusta:skills:dev    
          neusta:skills:prod   neusta:skills:prod   neusta:skills:prod 

# Wechsel des contexts auf docker-desktop    
➜  ~ kubectl config use-context docker-desktop
Switched to context "docker-desktop".

# Context prüfen
➜  ~ kubectl get nodes                        
NAME             STATUS   ROLES           AGE    VERSION
docker-desktop   Ready    control-plane   170m   v1.32.2
```

## Weitere Hinweise und Tips

* siehe auch [kubectl Spickzettel](https://kubernetes.io/de/docs/reference/kubectl/cheatsheet/)
* geht etwas schief, kann der gesamte Namespace mit einem `kubectl delete namespace gamma-lightning-talk` abgeräumt
  werden! Danach müssen die Schritte allerdings wiederholt werden.

## Der Talk

Es werden Schrittweise die wesentlichen Dinge von Kubernetes erklärt

### Namespace

Ein Namespace in Kubernetes ist eine virtuelle Umgebung, die Ressourcen innerhalb eines Clusters isoliert. Namespaces ermöglichen es, Ressourcen zu gruppieren und Zugriffe zu kontrollieren.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: gamma-lightning-talk
```

```shell
# Namespaces anzeigen
➜  ~ kubectl get namespaces
NAME              STATUS   AGE
default           Active   171m
kube-node-lease   Active   171m
kube-public       Active   171m
kube-system       Active   171m

# Namespace hinzufügen
➜  ~ kubectl apply -f manifests/namespace.yaml
namespace/gamma-lightning-talk created

➜  ~ kubectl get namespaces
NAME                   STATUS   AGE
default                Active   173m
gamma-lightning-talk   Active   29s
kube-node-lease        Active   173m
kube-public            Active   173m
kube-system            Active   173m
```

### Deployment (inkl. Pod)

Ein Deployment verwaltet die Erstellung und Aktualisierung von Pods. Es stellt sicher, dass die gewünschte Anzahl von Pod-Repliken läuft und ermöglicht Rolling Updates.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: abfallkalender
  namespace: gamma-lightning-talk
spec:
  replicas: 1
  selector:
    matchLabels:
      app: abfallkalender
  template:
    metadata:
      labels:
        app: abfallkalender
    spec:
      containers:
        - name: abfall-kalender-api
          image: larmic/abfallkalender_api:latest
```

```shell
# Deployment hinzufügen
➜  ~ kubectl apply -f manifests/deployment.yaml
deployment.apps/abfallkalender created

# Pods anschauen und richtigen Namespace wählen
➜  ~ kubectl get pods -n gamma-lightning-talk
NAME                              READY   STATUS    RESTARTS   AGE
abfallkalender-69879fb4fd-np724   1/1     Running   0          2m27s
```

### Service

Ein Service definiert einen stabilen Endpunkt für den Zugriff auf Pods. Er ermöglicht die Kommunikation zwischen verschiedenen Anwendungen innerhalb des Clusters.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: abfallkalender
  namespace: gamma-lightning-talk
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30429
  selector:
    app: abfallkalender
```

```shell
➜  ~ curl http://localhost:30429/abfallkalender-api
openapi: 3.0.3

➜  ~ curl http://localhost:30429/abfallkalender-api/street/Langwedeler+Stra%C3%9Fe/number/3a/next
{"day_of_collection":"2025-06-14","garbage_types":["blue","yellow"]}
```

Kubernetes Service Types:
* ClusterIP (default): Macht den Service nur innerhalb des Clusters erreichbar
* NodePort: Öffnet einen Port auf jedem Node, der auf den Service weiterleitet (Port-Bereich: 30000-32767)
* LoadBalancer: Nutzt einen externen Load Balancer für den Zugriff auf den Service
* ExternalName: Leitet auf einen externen DNS-Namen weiter

```shell
# Service hinzufügen
➜  ~ kubectl apply -f manifests/service.yaml
service/abfallkalender created

# Service prüfen
➜  ~ kubectl get services -n gamma-lightning-talk
NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
abfallkalender   ClusterIP   10.99.139.216   <none>        80/TCP    17s
```

### Ingress

Ein Ingress ist ein API-Objekt, das HTTP/HTTPS-Routen von außerhalb des Clusters zu Services innerhalb des Clusters bereitstellt. Es fungiert als Eingangstor für externe Anfragen.

```shell
# Prüfen, ob ein Ingress verfügbar ist
# Docker Desktop Kubernetes bringt keinen eigenen mit
➜  ~ kubectl get ingressclass
No resources found

# Traefik Ingress Controller mit Helm installieren
➜  ~ helm repo add traefik https://traefik.github.io/charts
➜  ~ helm repo update
➜  ~ helm install traefik traefik/traefik

# Ingress controller prüfen
➜  ~ kubectl get ingressclass
NAME      CONTROLLER                      PARAMETERS   AGE
traefik   traefik.io/ingress-controller   <none>       23s
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: abfallkalender
  namespace: gamma-lightning-talk
spec:
  rules:
    - host: abfallkalender.gamma.neusta.de
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: abfallkalender
                port:
                  number: 80
```

```shell
➜  ~ kubectl apply -f manifests/ingress.yaml
ingress.networking.k8s.io/abfallkalender created

# Url auf dem Testsystem bekannt machen
➜  ~ less /etc/hosts
127.0.0.1 abfallkalender.gamma.neusta.de
```

```shell
➜  ~ curl http://abfallkalender.gamma.neusta.de/abfallkalender-api
openapi: 3.0.3

➜  ~ curl http://abfallkalender.gamma.neusta.de/abfallkalender-api/street/Langwedeler+Stra%C3%9Fe/number/3a/next
{"day_of_collection":"2025-06-14","garbage_types":["blue","yellow"]}
```