
-----

# Projeto: Pipeline CI/CD com GitOps (GitHub Actions + ArgoCD)

Este repositório contém o código-fonte de uma aplicação FastAPI (`hello-app`) e serve como o gatilho de Integração Contínua (CI) para um fluxo GitOps.

O objetivo deste projeto é demonstrar um pipeline DevSecOps moderno onde uma alteração no código (`git push`) dispara automaticamente o build, teste e publicação de uma imagem Docker, que é então proposta (via Pull Request) para ser implantada em um cluster Kubernetes gerenciado pelo ArgoCD.

## 🏛️ Arquitetura da Solução

O fluxo de trabalho é baseado em dois repositórios:

1.  **`hello-app` (Este repositório):** Contém o código da aplicação (FastAPI) e o pipeline de CI (GitHub Actions). A responsabilidade dele é **construir a imagem**.
2.  **`hello-manifests` (Repositório de CD):** Contém os manifestos YAML do Kubernetes. A responsabilidade dele é **declarar o estado desejado** da aplicação.

O fluxo é o seguinte:

1.  Um desenvolvedor faz um `push` no `hello-app`.
2.  O **GitHub Actions** é acionado, constrói uma nova imagem Docker com uma tag única (o hash do commit).
3.  A Action envia a imagem para o **Docker Hub**.
4.  A Action, então, clona o repositório `hello-manifests` e abre um **Pull Request** automático para atualizar o `deployment.yaml` com a nova tag da imagem.
5.  Um mantenedor aprova e faz o *merge* do PR no `hello-manifests`.
6.  O **ArgoCD**, que monitora o `hello-manifests`, detecta a mudança e automaticamente "puxa" a nova imagem para o cluster **Kubernetes**, completando o ciclo.

## 🛠️ Tecnologias Utilizadas

  * **GitHub** (para hospedagem do Git)
  * **GitHub Actions** (para Integração Contínua - CI)
  * **Docker / Docker Hub** (para containerização e registro de imagens)
  * **Rancher Desktop** (para o cluster Kubernetes local)
  * **ArgoCD** (para Entrega Contínua - CD e GitOps)
  * **FastAPI (Python)** (para a aplicação de exemplo)

-----

## 🚀 Passo a Passo da Implementação

### 1\. Estrutura da Aplicação (`hello-app`)

Os arquivos principais neste repositório são:

**`main.py`**

```python
from fastapi import FastAPI
import os

app = FastAPI()

# Pega a versão da variável de ambiente para demonstração
app_version = os.environ.get("APP_VERSION", "1.0.0")

@app.get("/")
async def root():
    return {"message": "Hello World - teste V2", "version": app_version}
```

**`requirements.txt`**

```
fastapi
uvicorn
```

**`Dockerfile`**

```dockerfile
# Use uma imagem base do Python
FROM python:3.9-slim

# Defina o diretório de trabalho
WORKDIR /app

# Copie o arquivo de dependências e instale
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copie o código da aplicação
COPY main.py .

# Exponha a porta que a aplicação usará
EXPOSE 80

# Comando para rodar a aplicação com uvicorn
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "80"]
```

### 2\. Estrutura dos Manifestos (`hello-manifests`)

No outro repositório, mantemos os arquivos de estado do Kubernetes (dentro de uma pasta `k8s/`):

**`k8s/deployment.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app
  labels:
    app: hello-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-app
  template:
    metadata:
      labels:
        app: hello-app
    spec:
      containers:
        - name: hello-app
          # Esta linha será atualizada pelo CI/CD
          image: pedrooHSN-dev/hello-app:initial
          ports:
            - containerPort: 80
          
          # Esta variável nos ajuda a ver a versão sendo atualizada
          env:
            - name: APP_VERSION
              value: "initial" # Esta linha também será atualizada pelo pipeline

          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
```

**`k8s/service.yaml`**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-app-service
spec:
  selector:
    app: hello-app
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
```

### 3\. O Pipeline de CI/CD (`.github/workflows/cicd.yaml`)

Este é o cérebro da automação, o arquivo final que combina as melhores práticas de build (com `buildx`) e criação de PR (com `peter-evans/create-pull-request`).

```yaml
name: CI-CD Pipeline

on:
  push:
    branches:
      - main

jobs:
  build-and-push:
    name: Build and Push to Docker Hub
    runs-on: ubuntu-latest
    
    outputs:
      image_tag: ${{ steps.generate_tag.outputs.tag }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Generate short SHA tag
        id: generate_tag
        run: echo "tag=$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ secrets.DOCKER_USERNAME }}/hello-app:${{ steps.generate_tag.outputs.tag }}
            ${{ secrets.DOCKER_USERNAME }}/hello-app:latest
          platforms: linux/amd64,linux/arm64

  create-pull-request:
    name: Create Pull Request in Manifests Repo
    needs: build-and-push
    runs-on: ubuntu-latest
    
    steps:
      - name: Set up SSH
        uses: webfactory/ssh-agent@v0.7.0
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}
          
      - name: Checkout manifests repository
        uses: actions/checkout@v4
        with:
          repository: PedrooHSN-Dev/hello-manifests
          ssh-key: true
          path: hello-manifests
          
      - name: Update manifest files
        run: |
          cd hello-manifests
          cd k8s  # Entra na subpasta dos manifestos
          
          NEW_TAG=${{ needs.build-and-push.outputs.image_tag }}
          
          # Atualiza a tag da imagem no deployment.yaml
          sed -i 's|image:.*|image: ${{ secrets.DOCKER_USERNAME }}/hello-app:'$NEW_TAG'|' deployment.yaml
          # Atualiza a tag da versão na variável de ambiente
          sed -i 's|value:.*|value: "'$NEW_TAG'"|' deployment.yaml

      - name: Create Pull Request
        uses: peter-evans/create-pull-request@v5
        with:
          path: hello-manifests
          token: ${{ secrets.GH_PAT }} # Usa o Personal Access Token
          commit-message: 'ci: update image to version ${{ needs.build-and-push.outputs.image_tag }}'
          title: 'Auto: Update image to version ${{ needs.build-and-push.outputs.image_tag }}'
          body: 'Automated PR to update image tag from CI pipeline.'
          branch: 'update-image-${{ needs.build-and-push.outputs.image_tag }}'
          delete-branch: true
```

### 4\. Configuração de Segurança (Secrets e Chaves)

Para o pipeline funcionar, as seguintes configurações são necessárias:

1.  **Secrets no `hello-app`** (`Settings > Secrets and variables > Actions`):

      * `DOCKER_USERNAME`: Nome de usuário do Docker Hub.
      * `DOCKER_PASSWORD`: Token de Acesso do Docker Hub.
      * `SSH_PRIVATE_KEY`: A chave SSH **privada** usada para dar `push` no `hello-manifests`.
      * `GH_PAT`: Um Personal Access Token (PAT) do GitHub com escopo `repo` para criar o Pull Request.

2.  **Deploy Key no `hello-manifests`** (`Settings > Deploy keys`):

      * A chave SSH **pública** correspondente à privada acima.
      * **Importante:** A permissão "Allow write access" (Permitir acesso de escrita) deve estar marcada.

### 5\. Configuração do ArgoCD

No cluster Kubernetes, com o ArgoCD instalado, configuramos um novo App para monitorar o repositório `hello-manifests`.

  * **Repository URL:** `https://github.com/PedrooHSN-Dev/hello-manifests.git`
  * **Path:** `k8s` (a subpasta onde os manifestos estão)
  * **Cluster:** `https://kubernetes.default.svc`
  * **Namespace:** `default`

-----

## ⚠️ Erros

\<details\>
\<summary\>\<b\>Erro 3: \<code\>[remote rejected] (refused to allow integration to create or update pull request)\</code\>\</b\>\</summary\>

  * **Sintoma:** O pipeline conseguia fazer o `push` do branch, mas falhava na etapa final de `Create Pull Request`.
  * **Causa:** O `GITHUB_TOKEN` (padrão) não tem permissão para criar Pull Requests em *outro* repositório.
  * **Solução:** 1.  Gerar um **Personal Access Token (PAT)** no GitHub com escopo `repo`.
    2\.  Adicionar esse token como um *secret* no `hello-app` (ex: `GH_PAT`).
    3\.  Atualizar o `cicd.yaml` para usar `token: ${{ secrets.GH_PAT }}` na action `peter-evans/create-pull-request`.

\</details\>

-----

## 🏆 Prova de Sucesso: O Fluxo em Ação

Para validar o pipeline, fizemos uma alteração final no `main.py` para testar o fluxo completo.

#### 1\. A Mudança no Código

A mensagem no `main.py` foi alterada para "Hello World - teste V2":

<img width="1389" height="227" alt="Screenshot_1" src="https://github.com/user-attachments/assets/f1ac6270-7619-4fd6-a99d-35d629680e64" />

#### 2\. O Pipeline de CI (GitHub Actions)

O `git push` dessa mudança disparou o pipeline, que foi executado com sucesso, construindo e publicando a nova imagem.

<img width="1389" height="452" alt="Screenshot_2" src="https://github.com/user-attachments/assets/f4716e09-aecb-4115-99dc-560a1048a9e6" />

#### 3\. O Deploy de CD (ArgoCD)

Após o Pull Request ser aprovado e feito o *merge*, o ArgoCD detectou a mudança no repositório `hello-manifests`, puxou a nova imagem e atualizou os pods no Kubernetes. A aplicação estabilizou em `Synced` (Sincronizada) e `Healthy` (Saudável), servindo a nova versão.

<img width="1389" height="813" alt="Screenshot_3" src="https://github.com/user-attachments/assets/0b9c839c-c825-42f6-9b7b-5f25c45f9787" />

#### 4\. Testando a Aplicação Localmente

Para aceder à aplicação que está rodando no cluster Rancher Desktop, usamos `port-forward` (já que o serviço é do tipo `ClusterIP`), direcionando para a porta `8081` (para não conflitar com a porta `8080` do ArgoCD):

```bash
kubectl port-forward svc/hello-app-service -n default 8081:80
```

Acessando `http://localhost:8081` no navegador, a nova mensagem do `main.py` é exibida com sucesso:

<img width="366" height="97" alt="Screenshot_4" src="https://github.com/user-attachments/assets/81f3aee3-ce15-4e66-bd61-0f02e04dd8ec" />
