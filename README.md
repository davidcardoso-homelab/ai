
# IA - Homelab

Projeto para orquestração de serviços de inteligência artificial no cluster Kubernetes, utilizando ArgoCD para GitOps e aplicações open source para inferência local de modelos de linguagem.

## Estrutura do Projeto

```
ai/
├── chromadb/
├── ntfy/
├── ollama/
├── odysseus/
├── searxng/
└── setup/
```

Cada aplicação possui:
- Manifests Kubernetes para Deployment, Service, Ingress e Persistent Volumes.
- Arquivos de configuração para integração com ArgoCD.

O diretório `setup/` contém configurações do projeto e repositório para o ArgoCD.

## Aplicações

- **Ollama**: Motor de inferência local para modelos de linguagem. Executa o modelo **Gemma3 4B** e expõe uma API REST compatível com OpenAI em `ollama.homelab`.
- **Odysseus**: Workspace de IA self-hosted com suporte a chat, agentes, pesquisa, documentos e integração com modelos locais via Ollama. Acessível em `odysseus.homelab`.
- **ChromaDB**: Banco de dados vetorial utilizado pelo Odysseus para armazenamento e busca de embeddings (RAG). Serviço interno ao cluster.
- **SearXNG**: Mecanismo de busca privado utilizado pelo Odysseus para pesquisa web nos modos de agente e pesquisa profunda. Serviço interno ao cluster.
- **ntfy**: Servidor de notificações push utilizado pelo Odysseus para alertas e notificações. Acessível em `ntfy.homelab`.

## Volumes e Armazenamento

- Utiliza volumes persistentes locais (`local-path`) para modelos e dados das aplicações.
- Estrutura de armazenamento:
  - `/volumes/ssd/k3s/ollama` — modelos baixados pelo Ollama (50Gi)
  - `/volumes/ssd/k3s/odysseus` — dados e banco de dados do Odysseus (2Gi)
  - `/volumes/ssd/k3s/chromadb` — índices vetoriais do ChromaDB (5Gi)
  - `/volumes/ssd/k3s/ntfy` — cache de notificações do ntfy (1Gi)

Exemplo de PersistentVolume:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ollama-models-pv
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-path
  hostPath:
    path: /volumes/ssd/k3s/ollama
```

## Modelos

O modelo **Gemma3 4B** (~2,5 GB) é baixado automaticamente na primeira inicialização do Ollama via lifecycle hook. Para adicionar outros modelos manualmente:

```sh
kubectl exec -n ai deploy/ollama -- ollama pull <modelo>
```

## Gerenciamento via ArgoCD

- O projeto é gerenciado pelo ArgoCD, facilitando o versionamento e o deploy contínuo das aplicações.
- Configurações do projeto e repositório estão em `setup/project.yaml` e `setup/repository.yaml`.

## Passos para Instalação

1. **Altere o arquivo `setup/repository.yaml`:**
   - Insira sua chave SSH privada no campo `sshPrivateKey`.

2. **Altere o arquivo `odysseus/setup/secret.yaml`:**
   - Defina a senha do administrador no campo `ODYSSEUS_ADMIN_PASSWORD`.

3. **Crie os diretórios de armazenamento no host:**
   ```sh
   sudo mkdir -p /volumes/ssd/k3s/ollama
   sudo mkdir -p /volumes/ssd/k3s/odysseus
   sudo mkdir -p /volumes/ssd/k3s/chromadb
   sudo mkdir -p /volumes/ssd/k3s/ntfy
   ```

4. **Aplique os manifests do setup:**
   ```sh
   kubectl apply -f setup/
   ```

5. **Aplique os secrets de cada aplicação:**
   ```sh
   kubectl apply -f searxng/setup/secret.yaml
   kubectl apply -f odysseus/setup/secret.yaml
   ```

6. **Aplique os manifests de cada aplicação:**
   ```sh
   kubectl apply -f chromadb/setup/application.yaml
   kubectl apply -f searxng/setup/application.yaml
   kubectl apply -f ntfy/setup/application.yaml
   kubectl apply -f ollama/setup/application.yaml
   kubectl apply -f odysseus/setup/application.yaml
   ```

7. **Acompanhe o deploy pelo ArgoCD:**
   - Acesse a interface do ArgoCD e sincronize as aplicações.

8. **Acesse as aplicações:**
   - Ollama API: `http://ollama.homelab`
   - Odysseus: `http://odysseus.homelab`
   - ntfy: `http://ntfy.homelab`

## Observações

- Não comite senhas ou chaves privadas em repositórios públicos — use `SealedSecrets`, `ExternalSecrets` ou mantenha os arquivos de secret fora do repositório.
- O Odysseus depende do Ollama, ChromaDB, SearXNG e ntfy. Certifique-se de que os quatro estejam em execução antes de iniciar o Odysseus.
- O `secret_key` do SearXNG é gerenciado via `searxng/setup/secret.yaml` (não commitado). Defina um valor aleatório em `SEARXNG_SECRET` antes de aplicar.
- Para upgrades das aplicações, siga as recomendações oficiais de cada projeto.

---
Projeto mantido por [davidcardoso-homelab](https://github.com/davidcardoso-homelab/).
