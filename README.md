# Kind - Kubernetes-in-Docker
Este projeto visa criar um cluster kuberntes com kind para estudos. Ele roda um nginx com ingress TLS, tem redirecionamento http para https e pertmite paths /$(contecto).

# Procedimento Completo

🛠️ Ambiente Kind com HTTPS via Ingress
✅ Objetivo
Criar um cluster local com Kind, expor uma aplicação via Ingress com domínio amigável (nginx.local) e acesso HTTPS com certificado confiável via mkcert.

🔷 1. Criar o cluster Kind com portas 80 e 443 mapeadas
📄 Arquivo: kind-config.yaml

yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
      - containerPort: 443
        hostPort: 8443
  - role: worker
  - role: worker		

---
kind create cluster --config kind-config.yaml
🔷 2. Instalar o Ingress Controller (ingress-nginx)
bash
Copiar
Editar
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/kind/deploy.yaml

⚠️ Aguarde os pods ficarem prontos:
$kubectl get pods -n ingress-nginx

---
🔷 3. Permitir pods no nó control-plane
$kubectl taint nodes --all node-role.kubernetes.io/control-plane-

---
🔷 4. Adicionar entrada no /etc/hosts (Linux e Windows)
🧩 Adicione esta linha em ambos os sistemas:

127.0.0.1 nginx.local
Linux (WSL): /etc/hosts
Windows: C:\Windows\System32\drivers\etc\hosts

---
🔷 5. Criar namespace hml
$kubectl create namespace hml

---
🔷 6. Criar o Deployment e Service da aplicação
📄 Arquivo: nginx-deployment.yaml

yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: hml
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - containerPort: 80
📄 Arquivo: nginx-service.yaml
---
yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: hml
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80

$kubectl apply -f nginx-deployment.yaml
$kubectl apply -f nginx-service.yaml

---
🔷 7. Criar o recurso Ingress
📄 Arquivo: nginx-ingress.yaml

yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  namespace: hml
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - nginx.local
      secretName: nginx-local-tls
  rules:
    - host: nginx.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx
                port:
                  number: 80


$kubectl apply -f nginx-ingress.yaml

---
🔷 8. Instalar mkcert e gerar certificado confiável
▶️ Instalar mkcert (no WSL/Linux)

$sudo apt install libnss3-tools
$curl -JLO https://github.com/FiloSottile/mkcert/releases/latest/download/mkcert-linux-amd64
chmod +x mkcert-linux-amd64
$sudo mv mkcert-linux-amd64 /usr/local/bin/mkcert
$mkcert -install

---
🔷 9. Gerar certificado para nginx.local

$mkcert nginx.local

Isso gerará:

nginx.local.pem
nginx.local-key.pem

---
🔷 10. Criar o Secret TLS no Kubernetes

$kubectl create secret tls nginx-local-tls \
  --cert=nginx.local.pem \
  --key=nginx.local-key.pem \
  -n hml

---
✅ 11. Acessar no navegador
Abra:

https://nginx.local:8443
✔️ O navegador não exibirá alerta de site inseguro, pois o certificado foi confiado via mkcert.

📌 Considerações finais
Para reutilizar com outra aplicação, basta alterar o Deployment e o Service mantendo o mesmo Ingress e Secret.

Essa estrutura garante um ambiente seguro e local para testes com HTTPS real.

Ideal para simular produção com redirecionamento HTTP → HTTPS.
