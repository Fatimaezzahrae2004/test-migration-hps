🚀 POC Migration PowerCARD V4 : Ingress vers Gateway API

Ce projet constitue une Preuve de Concept (POC) pour la modernisation de l'infrastructure réseau de PowerCARD V4. L'objectif est de remplacer l'ancien contrôleur Ingress NGINX par la nouvelle Gateway API de Kubernetes via NGINX Gateway Fabric.

📋 Sommaire

Prérequis

Installation de l'Infrastructure

Configuration TLS

Déploiement

Tests des Protocoles

HTTP/1.1 & Routage

HTTP/2 (h2) via HTTPS

TCP (Flux Binaires)

gRPC (Haute Performance)

🛠 Prérequis <a name="prérequis"></a>

Kubernetes (Docker Desktop, Minikube ou K3s)

Helm (Gestionnaire de paquets)

OpenSSL (Pour la génération de certificats)

grpcurl (Pour les tests gRPC)

⚙️ Installation de l'Infrastructure <a name="installation"></a>

1. Installer les CRD Gateway API (Experimental)

Les ressources Gateway API (TCPRoute, GRPCRoute) nécessitent l'installation des définitions de ressources :

kubectl apply -f [https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/experimental-install.yaml](https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/experimental-install.yaml)


2. Installer NGINX Gateway Fabric

helm install ngf oci://ghcr.io/nginxinc/charts/nginx-gateway-fabric `
  --set service.extraPorts[0].name=tcp-9000 `
  --set service.extraPorts[0].port=9000 `
  --set service.extraPorts[0].targetPort=9000 `
  --set service.extraPorts[0].protocol=TCP `
  -n nginx-gateway --create-namespace


🔒 Configuration TLS <a name="configuration-tls"></a>

Le protocole HTTP/2 et le gRPC nécessitent une connexion sécurisée.

# Générer le certificat auto-signé
openssl req -x509 -nodes -days 365 -newkey rsa:2048 `
  -keyout demo.key -out demo.crt `
  -subj "/CN=demo.local/O=PWC-Migration"

# Créer le secret dans Kubernetes
kubectl create secret tls demo-tls-secret --key demo.key --cert demo.crt


🚀 Déploiement <a name="déploiement"></a>

Déployez l'application avec tous les protocoles activés :

helm upgrade --install demo-app ./charts/demo-app `
  --set gateway.enabled=true `
  --set tls.enabled=true `
  --set tcp.enabled=true `
  --set grpc.enabled=true


🧪 Tests des Protocoles <a name="tests-des-protocoles"></a>

1. HTTP/1.1 & Routage <a name="http11"></a>

Vérifie que la Gateway redirige correctement le trafic Web standard.

Tunnel : kubectl port-forward -n nginx-gateway svc/ngf-nginx-gateway-fabric 8080:80

Commande : curl.exe -H "Host: demo.local" http://localhost:8080

2. HTTP/2 (h2) <a name="http2"></a>

Vérifie le multiplexage binaire via HTTPS.

Tunnel : kubectl port-forward -n nginx-gateway svc/ngf-nginx-gateway-fabric 8443:443

Vérification Browser : Accédez à https://localhost:8443 -> F12 -> Network -> Colonne "Protocol" : h2.

3. TCP (Flux Binaires) <a name="tcp"></a>

Simule un flux bancaire (ISO8583) sur le port 9000.

Tunnel : kubectl port-forward -n nginx-gateway svc/ngf-nginx-gateway-fabric 9000:9000

Commande : Test-NetConnection -ComputerName localhost -Port 9000

Succès : TcpTestSucceeded : True

4. gRPC <a name="grpc"></a>

Vérifie la communication haute performance entre microservices PowerCARD.

Tunnel : kubectl port-forward -n nginx-gateway svc/ngf-nginx-gateway-fabric 8445:443

Commande : ./grpcurl.exe -insecure -authority demo.local localhost:8445 list

Succès : Affiche grpcbin.GRPCBin et grpc.reflection.v1alpha.ServerReflection.

📁 Structure du Projet

/charts/demo-app/values.yaml : Configuration centralisée.

/templates/gateway.yaml : Point d'entrée multi-port (80, 443, 9000).

/templates/httproute.yaml : Routage L7 (Web).

/templates/tcproute.yaml : Routage L4 (Flux bruts).

/templates/grpcroute.yaml : Routage L7 spécifique (HTTP/2 binaire).
