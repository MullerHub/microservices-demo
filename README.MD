
# Resumo Técnico

As modificações no manifesto adaptam a aplicação de um ambiente de produção ideal para um ambiente de desenvolvimento/teste com recursos limitados. Elas trocam a detecção rápida de falhas por uma maior resiliência e tolerância a lentidões, resultando em uma implantação mais estável para os fins deste projeto.


# Documentação: Implantação de Microserviços com Argo CD em Ambiente Local

Este documento detalha o procedimento técnico para configurar um ambiente de desenvolvimento local em um MacBook M1 e implantar uma aplicação de microserviços (Online Boutique) utilizando GitOps com Argo CD.

## Sumário

1.  [Escolha da Arquitetura Local: Docker Desktop vs. Rancher](#1-escolha-da-arquitetura-local-docker-desktop-vs-rancher)
2.  [Pré-requisitos](#2-pré-requisitos)
3.  [Etapa 1: Configuração do Ambiente Kubernetes Local](#etapa-1-configuração-do-ambiente-kubernetes-local)
4.  [Etapa 2: Ins talação do Argo CD](#etapa-2-instalação-do-argo-cd)
5.  [Etapa 3: Declaração e Implantação da Aplicação](#etapa-3-declaração-e-implantação-da-aplicação)
6.  [Etapa 4: Acesso aos Serviços](#etapa-4-acesso-aos-serviços)

---

### 1. Escolha da Arquitetura Local: Docker Desktop vs. Rancher

A arquitetura para este projeto utiliza o cluster Kubernetes embutido no **Docker Desktop**, evitando deliberadamente o uso do **Rancher**.

**Justificativa:** Durante a fase de setup inicial, a tentativa de utilizar Rancher em um ambiente de **MacBook M1 com 8GB de RAM** apresentou problemas críticos que impediram o progresso:

* **Conflitos de Baixo Nível:** Foram encontrados erros de sistema e instabilidade, possivelmente relacionados a incompatibilidades ou limitações da camada de virtualização do Rancher sobre a arquitetura **Apple Silicon (M1)**.
* **Consumo Excessivo de Recursos:** O Rancher demonstrou um consumo de memória elevado. Em uma máquina com 8GB de RAM, isso resultou em uso massivo de **swap de disco**, causando uma degradação severa de performance em todo o sistema e falhas na inicialização dos componentes do cluster.

A abordagem com Docker Desktop Kubernetes provou-se mais estável, com menor sobrecarga de recursos e diretamente compatível com o hardware, tornando-se a escolha técnica viável para este projeto.

---

### 2. Pré-requisitos

* Hardware: MacBook com Apple Silicon (M1/M2/M3).
* Software:
    * [Homebrew](https://brew.sh/) instalado.
    * [Docker Desktop para Mac](https://www.docker.com/products/docker-desktop/) instalado.

---

### Etapa 1: Configuração do Ambiente Kubernetes Local

Utilizamos o cluster Kubernetes embutido no Docker Desktop para simplicidade e eficiência de recursos.

#### 1.1. Habilitar o Kubernetes no Docker Desktop

1.  Abra o Docker Desktop.
2.  Navegue até **Settings > Kubernetes**.
3.  Marque a caixa **Enable Kubernetes**.
4.  Clique em **Apply & Restart**. O Docker fará o download dos componentes e iniciará um cluster de nó único.

#### 1.2. Verificar o Contexto do `kubectl`

O Docker Desktop deve configurar o contexto automaticamente. Verifique se o `kubectl` está apontando para o cluster correto.

`kubectl config current-context`


A saída esperada é `docker-desktop`

## Etapa 2: Instalação do Argo CD

2.1. Instalar a CLI do Argo CD

A CLI é usada para gerenciar a instância do Argo CD.

`brew install argocd`

2.2. Instalar Componentes do Argo CD no Cluster

Crie a namespace e aplique o manifesto de instalação padrão.

# 1. Criar a namespace
`kubectl create namespace argocd`

# 2. Aplicar o manifesto de instalação
`kubectl apply -n argocd -f [https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml](https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml)`

2.3. Obter Senha de Acesso

A senha inicial do usuário admin é gerada automaticamente e armazenada em um Secret.

# O comando extrai e decodifica a senha
`kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d`

Copie esta senha para uso posterior.

### Etapa 3: Declaração e Implantação da Aplicação

Configuramos o Argo CD para monitorar um repositório Git e implantar os manifestos contidos nele.

### 3.1. Criar a Aplicação no Argo CD

Use o comando argocd app create com todos os parâmetros especificados para o projeto. Este comando registra a aplicação mas não a sincroniza, pois a sync-policy é manual.

`argocd app create online-boutique \
    --repo [https://github.com/MullerHub/gitops-microservices](https://github.com/MullerHub/gitops-microservices) \
    --path k8s \
    --dest-server [https://kubernetes.default.svc](https://kubernetes.default.svc) \
    --dest-namespace online-boutique \
    --project default \
    --sync-policy manual \
    --sync-option PruneLast=true \
    --sync-option CreateNamespace=true`

### 3.1. Criar a Aplicação via Interface Gráfica (UI)

Em vez de usar a linha de comando, a aplicação foi configurada diretamente na interface web do Argo CD, que é mais visual e intuitiva.

Acesse a UI do Argo CD através do port-forward que será detalhado na Etapa 4.1. A URL é https://localhost:8080. Utilize o usuário admin e a senha obtida no passo 2.3.

Na página principal, clique no botão + NEW APP.

Preencha o formulário com as seguintes informações para configurar a aplicação online-boutique:

Application Name: `online-boutique`

Project Name: `default`

Sync Policy: `Manual`

### Source

Repository URL: https://github.com/MullerHub/gitops-microservices

Revision: `HEAD`

Path: `k8s`

### Destination

Cluster URL: `https://kubernetes.default.svc (valor padrão in-cluster)`

Namespace: `online-boutique`

Marque a opção Auto-Create Namespace para que o Argo CD crie a namespace caso ela não exista.

Clique em CREATE no topo da página. A aplicação será registrada no Argo CD, mas aguardará a sincronização manual.

## 3.1.1. Em caso de falha na criação da aplicação

Se a criação via interface gráfica ou CLI falhar, uma alternativa é aplicar o manifesto da aplicação diretamente com kubectl. Importe o arquivo online-boutique-monolitico.yaml com as especificações da aplicação Argo CD e aplique-o com o comando:
`kubectl apply -n argocd -f online-boutique-monolitico.yaml`


### 3.2. Sincronizar a Aplicação Manualmente

Como a política de sincronização é manual, execute o seguinte comando para instruir o Argo CD a implantar os manifestos do repositório no cluster.

argocd app sync online-boutique
Após a execução, o Argo CD aplicará os manifestos do diretório /k8s do repositório, criando todos os Deployments, Services, e outros recursos na namespace online-boutique.

### Etapa 4: Acesso aos Serviços

Para acessar tanto a UI do Argo CD quanto a aplicação online-boutique a partir da máquina local, utilizamos kubectl port-forward. Cada serviço requer seu próprio túnel, executado em um terminal separado.

### 4.1. Acesso ao Argo CD (Terminal 1)

Execute o comando a seguir em um terminal para criar o túnel para o servidor do Argo CD.

`kubectl port-forward svc/argocd-server -n argocd 8080:443`
URL de Acesso: https://localhost:8080

Usuário: admin

Senha: A senha obtida na Etapa 2.3.

### 4.2. Acesso à Aplicação Online Boutique (Terminal 2)

Execute o comando a seguir em um segundo terminal para criar o túnel para o serviço de frontend da aplicação.

`kubectl port-forward svc/frontend -n online-boutique 9090:80`
URL de Acesso: http://localhost:9090

Ambos os terminais devem permanecer em execução para manter o acesso aos serviços e toda aplicação estara funcionando se chegou até aqui.

---

### Modificações no Manifesto Kubernetes

Este documento descreve as alterações aplicadas ao manifesto Kubernetes original do projeto "Online Boutique". As modificações foram implementadas para garantir a estabilidade e o desempenho da aplicação durante a execução em ambientes com recursos computacionais limitados, como máquinas de desenvolvimento local (MacBook M1 8GB) e clusters de emulação para testes de estresse.

As alterações se concentram em duas áreas principais: **alocação de recursos (CPU e memória)** e **ajuste dos health probes (Liveness e Readiness)**.

OBS: tem a pasta *MODULOS* da qual é o manifesto original mas dividio em partes de modulos para testes progressivos e com menor uso de memoria, toda documentação abaixo foi escrita sobre o nosso manifesto chamado ` online-boutique-monolitico.yaml `


### 1. Aumento de Alocação de Recursos (CPU/Memória)

Para evitar que os pods sejam terminados (`OOMKilled`) ou sofram `CPU throttling` sob carga, os `requests` e `limits` de recursos foram aumentados para a maioria dos microserviços. Isso garante que cada serviço tenha recursos suficientes para iniciar e operar de forma estável. (Aqui era uma entrega opcional, mas no começo da depuração foi quase obrigatorio em nosso caso)

A tabela abaixo resume a alteração padrão aplicada à maioria dos `Deployments`:

| Recurso           | Configuração Original | Configuração Modificada | Justificativa Técnica                                |
| :---------------- | :-------------------- | :---------------------- | :--------------------------------------------------- |
| **CPU Requests** | `100m`                | `250m`                  | Garante mais tempo de CPU alocado pelo scheduler.    |
| **CPU Limits** | `200m`                | `500m`                  | Permite picos de uso de CPU sem throttling.          |
| **Memory Requests** | `64Mi`                | `256Mi`                 | Aloca mais memória garantida, evitando `OOMKilled`. |
| **Memory Limits** | `128Mi`               | `512Mi`                 | Permite picos de uso de memória sem `OOMKilled`.    |

> **Nota:** Alguns serviços como `loadgenerator` e `redis-cart` já possuíam valores mais altos e foram ajustados de forma similar para garantir a proporção.

### 2. Ajuste nos Health Probes (Liveness/Readiness)

Em ambientes com recursos restritos, os serviços podem demorar mais para inicializar. Os `probes` originais eram muito agressivos, fazendo com que o Kubernetes reiniciasse pods saudáveis, mas lentos. Os parâmetros foram ajustados para serem mais tolerantes.

| Parâmetro             | Configuração Original (ou Padrão) | Configuração Modificada | Justificativa Técnica                                                                                        |
| :-------------------- | :-------------------------------- | :---------------------- | :----------------------------------------------------------------------------------------------------------- |
| `initialDelaySeconds` | `10` ou não definido              | `60`                    | Aumenta o tempo de espera inicial antes do primeiro probe, permitindo que o contêiner inicialize completamente. |
| `periodSeconds`       | `5` ou `10`                       | `30`                    | Reduz a frequência das verificações, diminuindo a sobrecarga no serviço e no kubelet.                        |
| `failureThreshold`    | `3` (padrão)                      | `5`                     | Aumenta o número de falhas consecutivas necessárias antes de marcar o pod como "unhealthy", tolerando falhas temporárias. |