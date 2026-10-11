# Arquitetura do Kubernetes
 
Projeto da Unidade 9: desenho da arquitetura do Kubernetes (`kubernetes.drawio`), descrição das principais ferramentas e implantação de uma aplicação própria (imagem `pedroluiz02/meu_app_python:1.0`) em um cluster minikube.
 
**Integrantes:** Pedro Luiz e _(nome da dupla)_
 
## Estrutura do projeto
 
```
arquitetura_kubernetes/
├── kubernetes.drawio   # desenho da arquitetura (extensão "Drawio Integration" no VS Code)
├── README.md           # este arquivo
├── pod.yml             # implementação de um Pod
└── service.yml         # implementação de um Service (NodePort)
```
 
## Visão geral
 
O Kubernetes (K8s) é uma plataforma de orquestração de containers. Você descreve o **estado desejado** (quantos Pods, qual imagem, quais portas) em arquivos YAML, e o cluster trabalha continuamente para que o **estado real** seja igual ao desejado.
 
Um cluster é formado por um **Control Plane** (o "cérebro", que decide) e por um ou mais **Worker Nodes** (as máquinas que executam os containers).
 
## Control Plane
 
| Componente | Função |
|---|---|
| **kube-apiserver** | Porta de entrada do cluster. Expõe a API REST; o `kubectl`, o kubelet e os demais componentes conversam sempre com ele. Valida e grava as requisições. |
| **etcd** | Banco de dados chave-valor que guarda todo o estado do cluster (objetos criados, configurações). Só o API Server fala com ele. |
| **kube-scheduler** | Observa Pods novos sem Node e escolhe em qual Node cada um vai rodar, conforme recursos disponíveis e regras de afinidade. |
| **kube-controller-manager** | Executa os controllers (Node, ReplicaSet, Deployment, etc.) que comparam estado atual x desejado e corrigem diferenças, por exemplo recriando um Pod que caiu. |
 
## Worker Node
 
| Componente | Função |
|---|---|
| **kubelet** | Agente que roda em cada Node. Recebe do API Server a lista de Pods que devem rodar ali, pede ao runtime que os inicie e reporta o status. |
| **kube-proxy** | Mantém as regras de rede do Node para que o tráfego enviado a um Service chegue a um dos Pods correspondentes. |
| **Container Runtime** | Software que de fato executa os containers (containerd, Docker, CRI-O). |
 
## Objetos do Kubernetes
 
| Objeto | Descrição |
|---|---|
| **Pod** | Menor unidade implantável. Agrupa um ou mais containers que compartilham rede (mesmo IP) e volumes. Pods são efêmeros: se morrem, o IP muda. |
| **ReplicaSet** | Garante que um número definido de réplicas de um Pod esteja sempre em execução, criando ou removendo Pods conforme necessário. |
| **Deployment** | Gerencia ReplicaSets e permite atualizações graduais (rolling update), rollback e escalonamento de forma declarativa. |
| **Service** | Dá um endereço estável (IP e nome DNS) a um conjunto de Pods, selecionados por **labels**, e faz o balanceamento entre eles. Tipos: `ClusterIP` (interno), `NodePort` (porta em cada Node), `LoadBalancer` (IP externo em nuvem). |
| **Labels e Selectors** | Pares chave-valor colocados nos objetos; o selector de um Service ou ReplicaSet usa as labels para encontrar os Pods certos. |
| **Namespace** | Divisão lógica do cluster para isolar recursos (ex.: `default`, `kube-system`). |
 
## Ferramentas de linha de comando
 
| Ferramenta | Função |
|---|---|
| **kubectl** | CLI para conversar com o API Server: criar (`apply`), consultar (`get`, `describe`), ver logs (`logs`) e remover (`delete`) objetos. |
| **minikube** | Cria um cluster Kubernetes local (de um único Node) para estudo e testes. |
| **Docker / Docker Hub** | Docker constrói a imagem do container; o Docker Hub é o repositório de onde o cluster baixa a imagem. |
 
## Fluxo de funcionamento
 
1. O desenvolvedor escreve os manifestos YAML e executa `kubectl apply -f`.
2. O **kubectl** envia a requisição ao **kube-apiserver**, que valida e grava o estado desejado no **etcd**.
3. O **controller-manager** percebe o novo objeto e cria os Pods necessários (via ReplicaSet, quando há Deployment).
4. O **kube-scheduler** escolhe um Node para cada Pod ainda sem destino.
5. O **kubelet** do Node escolhido vê o Pod atribuído a ele e pede ao **container runtime** para baixar a imagem e iniciar o container.
6. O **Service** seleciona os Pods pela label e o **kube-proxy** encaminha o tráfego para eles.
7. O usuário acessa a aplicação pelo navegador via NodePort.
## Os manifestos
 
### pod.yml
 
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-python
  labels:
    app: app-python
spec:
  containers:
    - name: app-python
      image: pedroluiz02/meu_app_python:1.0
      ports:
        - containerPort: 5050
          name: http-service
```
 
### service.yml
 
```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-python-service
spec:
  type: NodePort
  selector:
    app: app-python
  ports:
    - name: app-service-port
      protocol: TCP
      port: 5050
      targetPort: http-service
      nodePort: 30050
```
 
O `selector` do Service (`app: app-python`) precisa ser igual à label do Pod; é assim que o Service encontra o Pod.
 
## Como executar no minikube
 
```bash
# 1. Iniciar o cluster
minikube start
 
# 2. Conferir o cluster
kubectl get nodes
 
# 3. Criar o Pod e o Service
kubectl apply -f pod.yml
kubectl apply -f service.yml
 
# 4. Verificar
kubectl get pods
kubectl get services
kubectl describe pod app-python
kubectl logs app-python
 
# 5. Acessar pelo navegador (abre a URL automaticamente)
minikube service app-python-service
 
# Alternativa: pegar a URL e abrir manualmente (http://<ip>:30050)
minikube service app-python-service --url
```
 
Se o navegador não conseguir acessar o IP do minikube (comum com o driver Docker no Windows/macOS), mantenha o comando `minikube service ...` rodando, pois ele cria um túnel e mostra uma URL local. Outra opção:
 
```bash
kubectl port-forward pod/app-python 5050:5050   # acessar em http://localhost:5050
```
 
Para limpar:
 
```bash
kubectl delete -f service.yml
kubectl delete -f pod.yml
minikube stop
```
 
## Evidências da execução
 
Adicione aqui os prints (pasta `evidencias/`):
 
- [ ] `kubectl get nodes` e `kubectl get pods` com o Pod em `Running`
- [ ] `kubectl get services` mostrando a porta `5050:30050/TCP`
- [ ] `kubectl describe pod app-python`
- [ ] Aplicação aberta no navegador