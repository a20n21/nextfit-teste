                                                         Teste Técnico DevOps - Next Fit

Este projeto tem como objetivo provisionar uma infraestrutura Kubernetes + Aplicação ASP.NET Core 100% em código pelo Terraform.

           Premissas do Projeto

Provisionamento da infraestrutura com Terraform
Criação de um cluster Kubernetes
Deploy de uma aplicação ASP.NET Core
Instalação e configuração do Istio no cluster

           Infraestrutura do Projeto
Cloud utilizada
Azure
Motivação
Utilizei a Cloud Azure, pois já venho há algum tempo aprofundando meus estudos na plataforma.
Custo
Possuo o voucher da Azure, então ficou mais viável realizar mais testes nas aplicações.

 Serviços Azure provisionados no Terraform
Resource Group
VNet + Subnet
AKS
ACR

            Provisionamento
Terraform Init
Terraform Plan
Terraform Apply

            Aplicação + Kubernetes

Framework utilizado
ASP.NET Core 8
Docker
Kubernetes (Deployment)
Kubernetes (Service)
Kubernetes (Ingress Controller) - Exposição
Kubernetes (Virtual Service) - Exposição
Service Mesh Istio
Sidecar
Ingress Gateway
Egress Gateway

           Estrutura do Projeto (Pastas)
Terraform
K8S
APP

           Docker + Build

Build da imagem
docker build -t meuapp:v1 .
Login no ACR
az acr login --name ******
Tag da imagem
docker tag meuapp:v1 ******.azurecr.io/meuapp:v1
Push da imagem
docker push ******.azurecr.io/meuapp:v1

          Deploy via Kubernetes

Obtenção de credenciais Azure
az aks get-credentials \
--resource-group nextfit-teste \
--name nextfit-aks

           Aplicação dos manifests

kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/ingress.yaml

        Implantação do Istio no Cluster
Na descrição do teste havia um campo para a implementação de Service Mesh nesse cluster.
Irei detalhar a implantação + ganhos futuros.

         Instalação do Istio
istioctl install --set profile=demo -y

         Verificação dos pods

kubectl get pods -n istio-system

Resultado:

istio-ingressgateway
istio-egressgateway
istiod

        Implantação do Sidecar

Habilitando namespace
kubectl label namespace default istio-injection=enabled
Restart no deployment
kubectl rollout restart deployment meuapp

Validação
kubectl get pods

Resultado
2/2 (APP + Sidecar Envoy)

            Gateway Istio

apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
 name: meuapp-gateway
spec:
 selector:
   istio: ingressgateway
 servers:
 - port:
     number: 80
     name: http
     protocol: HTTP
   hosts:
   - "*"

Função:

Receber tráfego externo
Entrar no Service Mesh

         Virtual Service Istio

apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
 name: meuapp-vs
spec:
 hosts:
 - "*"
 gateways:
 - meuapp-gateway
 http:
 - match:
   - uri:
       prefix: /
   route:
   - destination:
       host: meuapp-service
       port:
         number: 80

Função
Definir regras de roteamento
Direcionar tráfego para o service interno

        Validação do Service Mesh

istioctl proxy-status
kubectl get gateway
kubectl get virtualservice

              Acesso à Aplicação

curl http://20.226.129.179
Retorno
{
 "message": "Aplicação em produção para testes | Next Fit",
 "environment": "production",
 "timestamp": "2026-06-18T20:15:14.087719Z"
}

  Camadas de Segurança no Projeto

Registry privado no ACR
Princípio de menor privilégio no ACR (AcrPull)
Rede segmentada (VNet/Subnet)
Service interno (ClusterIP)
Sidecar proxy (Istio Envoy)
Infraestrutura 100% provisionada via Terraform

         Boas Práticas

Liveness Probe
 - Verifica se a aplicação está viva
 - Se falhar, o Kubernetes reinicia o container

Readiness Probe
 - Verifica se a aplicação está pronta para receber tráfego

Resources Requests
 - Define o mínimo de CPU e memória garantidos

Resources Limits
 - Define o máximo que o container pode consumir


          Fluxo de Tráfego da Aplicação

Internet
 Istio IngressGateway3
 Gateway
 VirtualService
 Service (ClusterIP)
 Pod (ASP.NET Core + Sidecar Envoy)

       Melhorias Futuras do Projeto
       
Stack de Observabilidade: Prometheus + Grafana (com Service Mesh facilita métricas detalhadas de tráfego)
Pipeline CI/CD com ArgoCD (GitOps)
Escalonamento de Pods (HPA)
Separação de ambientes: DEV, HOMOLOG, PROD
