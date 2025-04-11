# Deploy Blue-Green com Ansible + Kubernetes (Kind)

## 📘 Tutorial Atualizado — Blue-Green Deployment com Ansible + Kubernetes (Kind)

### 🎯 **Objetivo**

Realizar o deploy de uma aplicação Flask utilizando a estratégia **Blue-Green Deployment**, com:

- **Automação via Ansible**
- **Orquestração via Kubernetes (Kind)**
- **Acesso direto via NodePort sem uso de `port-forward`**

---

## 🗂️ Estrutura do Projeto (`deploy-k8s-bg`)

```
arduino
CopiarEditar
deploy-k8s-bg/
├── ansible-k8s-bg/
│   ├── inventory.ini
│   ├── playbook-deploy-bg.yaml
│   ├── playbook-switch.yaml
│   └── roles/
│       ├── k8s_deploy/
│       │   ├── tasks/main.yml
│       │   └── templates/
│       │       ├── deployment-blue.yaml.j2
│       │       ├── deployment-green.yaml.j2
│       │       └── service.yaml.j2  # atualizado com nodePort fixo
│       └── k8s_switch/
│           ├── tasks/main.yml
│           └── templates/service-switch.yaml.j2
├── app/
│   └── app.py
├── Dockerfile
└── kind-config.yaml  ✅

```

---

## ⚙️ Criação do Cluster Kind com mapeamento de porta

Crie o arquivo `kind-config.yaml` com o seguinte conteúdo:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
  - containerPort: 31252   # Mapeamento fixo para o NodePort do Service
    hostPort: 31252
    protocol: TCP
- role: worker
- role: worker
- role: worker
```

<aside>
📌

Esse mapeamento garante acesso via http://127.0.0.1:31252 sem precisar de kubectl port-forward.

</aside>

Crie o cluster:

```bash
kind create cluster --name kind-ansible --config kind-config.yaml
```

---

## 📦 Construir e carregar imagem Docker

```bash
docker build -t clecio/flask-app-ansible:latest .
kind load docker-image clecio/flask-app-ansible:latest --name kind-ansible
```

---

## 🚀 Executar Playbook de Deploy

```bash
cd ansible-k8s-bg
ansible-playbook -i inventory.ini playbook-deploy-bg.yaml
```

---

## 🔁 Alternar a versão ativa

```bash
ansible-playbook -i inventory.ini playbook-switch.yaml -e "active_version=blue"
# ou
ansible-playbook -i inventory.ini playbook-switch.yaml -e "active_version=green"
```

---

## 🧪 Verificar via `curl`

```bash
curl http://127.0.0.1:31252
```

A resposta indicará a versão ativa (`blue` ou `green`) com fundo colorido:

- `lightblue` → Blue
- `lightgreen` → Green

## 🛠️ Template de Service atualizado (`service.yaml.j2`)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: flask-app-ansible-svc
spec:
  type: NodePort
  selector:
    app: flask-app-ansible
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 31252  # Porta fixa usada no cluster Kind
```

---

## ✅ Requisitos

- Docker + Kind
- Kubernetes configurado localmente (`kubectl config current-context`)
- Ansible com collection `kubernetes.core` instalada:
    
    ```bash
    ansible-galaxy collection install kubernetes.core
    ```
    

---

## 🧭 Próximos Passos

- Implementar `canary deployment` em `deploy-k8s-canary/`
- Adicionar healthchecks (`livenessProbe`, `readinessProbe`)
- Automatizar testes com `ansible.builtin.uri`
- Adicionar rollback automatizado baseado em validação de resposta