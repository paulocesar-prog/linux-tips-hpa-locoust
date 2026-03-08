# linux-tips-hpa-locust

Projeto de demonstração de **HPA (Horizontal Pod Autoscaler)** no Kubernetes, com carga gerada por **Locust**. O objetivo é permitir testar e observar o comportamento de auto-scaling de pods sob carga controlada.

---

## O que este projeto faz

1. **Aplicação alvo (giropops-senhas)**: API Flask de geração de senhas com backend Redis
2. **Load testing (Locust)**: Simula usuários acessando a API para provocar escala
3. **HPA**: Escala o Deployment da aplicação com base em CPU e memória

---

## Arquitetura

```
                    ┌─────────────────────────────────────────────────────────┐
                    │                    Kubernetes Cluster                     │
                    │                                                           │
  Ingress (OCI)     │  ┌─────────────┐      ┌──────────────────┐               │
  giropops-senhas   │  │   Ingress   │──────│  giropops-senhas │◄──┐            │
  *.collecc.com.br  │  │  (oci-nic)  │      │  Service :5000   │   │            │
                    │  └─────────────┘      └────────┬─────────┘   │            │
                    │                                │             │            │
                    │         ┌──────────────────────┼─────────────┼───┐        │
                    │         │      HPA             │             │   │        │
                    │         │  (CPU 50% / Mem 50%) │             │   │        │
                    │         └──────────────────────┼─────────────┘   │        │
                    │                                │                 │        │
                    │         ┌──────────────────────▼─────────────────▼───┐    │
                    │         │  giropops-senhas Deployment (1→40 pods)    │    │
                    │         │  - Flask + Prometheus metrics (:8088)      │    │
                    │         └──────────────────────┬────────────────────┘    │
                    │                                │                         │
                    │                                ▼                         │
                    │         ┌──────────────────────────────────────────┐     │
                    │         │  redis-service :6379                      │     │
                    │         │  (armazena senhas geradas)                │     │
                    │         └──────────────────────────────────────────┘     │
                    │                                                           │
  Ingress (OCI)     │  ┌─────────────┐      ┌──────────────────┐               │
  locust-giropops   │  │   Ingress   │──────│  locust-giropops │               │
  *.collecc.com.br  │  │  (oci-nic)  │      │  Service :8089   │               │
                    │  └─────────────┘      └────────┬─────────┘               │
                    │                                │                         │
                    │                                ▼                         │
                    │         ┌──────────────────────────────────────────┐     │
                    │         │  locust-giropops Deployment               │     │
                    │         │  - Load testing → http://giropops-senhas  │     │
                    │         └──────────────────────────────────────────┘     │
                    │                                                           │
                    └─────────────────────────────────────────────────────────┘
```

---

## Componentes

| Componente      | Descrição                                                         |
|-----------------|-------------------------------------------------------------------|
| **giropops-senhas** | App Flask: gera senhas, armazena em Redis, expõe métricas Prometheus |
| **redis**       | Armazena senhas em memória (lista `senhas`)                       |
| **locust-giropops** | Locust: simula usuários chamando `/api/gerar-senha` e `/api/senhas` |
| **HPA**         | Escala giropops-senhas de 1 a 40 replicas (CPU e memória 50%)     |

### Endpoints da aplicação

| Endpoint              | Método | Descrição                        |
|-----------------------|--------|----------------------------------|
| `/`                   | GET/POST | Interface web para gerar senhas |
| `/api/gerar-senha`    | POST   | API para gerar senha (JSON)      |
| `/api/senhas`         | GET    | Lista as 10 senhas mais recentes |
| `/metrics`            | GET    | Métricas Prometheus              |

---

## Pré-requisitos

- Cluster Kubernetes (OKE/OCI ou outro)
- `kubectl` configurado
- **metrics-server** instalado (necessário para HPA baseado em recursos)
- Ingress Controller compatível (o projeto usa `oci-nic` para OCI Native Ingress)

---

## Deploy

### Ordem recomendada

1. Redis (base de dados)
2. Aplicação giropops-senhas (deployment, service)
3. HPA
4. Locust (configmap, deployment, service)
5. Ingress (se usar OCI — ajuste `certificate-ocid` e `subnet-id`)

### Comandos

```bash
# 1. Redis
kubectl apply -f giropops-senhas/redis-deployment.yaml
kubectl apply -f giropops-senhas/redis-service.yaml

# 2. Aplicação
kubectl apply -f giropops-senhas/app-deployment.yaml
kubectl apply -f giropops-senhas/app-service.yaml

# 3. HPA
kubectl apply -f giropops-senhas/app-hpa.yaml

# 4. Locust
kubectl apply -f locust/locust-configmap.yaml
kubectl apply -f locust/locust-deployment.yaml
kubectl apply -f locust/locust-service.yaml

# 5. Ingress (ajuste OCIDs conforme seu ambiente)
kubectl apply -f giropops-senhas/ingress.yaml
kubectl apply -f locust/ingress.yaml
```

### Verificar metrics-server

```bash
kubectl top nodes
kubectl top pods
```

Se não houver dados, o HPA não conseguirá escalar. Instale o metrics-server se necessário.

---

## HPA – Comportamento

O HPA está configurado com:

- **Target**: Deployment `giropops-senhas`
- **Min replicas**: 1
- **Max replicas**: 40
- **Métricas**: CPU e memória com `averageUtilization: 50%`
- **Scale Up**: janela de 5s, política 100% a cada 10s
- **Scale Down**: janela de 60s, política 100% a cada 10s

### Observar o HPA em ação

```bash
# Status do HPA
kubectl get hpa -w

# Número de pods
kubectl get pods -l app=giropops-senhas -w
```

Com o Locust injetando carga, o HPA deve aumentar os pods quando CPU/memória ultrapassam ~50%.

---

## Load Testing com Locust

1. Acesse a UI do Locust (via Ingress ou port-forward):
   ```bash
   kubectl port-forward svc/locust-giropops 8089:8089
   # Acesse http://localhost:8089
   ```

2. Na interface web:
   - **Number of users**: ex.: 50–200
   - **Spawn rate**: ex.: 5–10
   - Clique em **Start swarming**

3. Monitore o HPA e os pods em outro terminal:
   ```bash
   kubectl get hpa -w
   kubectl get pods -l app=giropops-senhas
   ```

### Perfil de carga (locustfile.py)

- `gerar_senha`: POST `/api/gerar-senha` (peso 1)
- `listar_senha`: GET `/api/senhas` (peso 2)
- `wait_time`: entre 1 e 2 segundos entre requisições

---

## Estrutura do repositório

```
linux-tips-hpa-locust/
├── giropops-senhas/
│   ├── app.py                 # App Flask
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── app-deployment.yaml
│   ├── app-service.yaml
│   ├── app-hpa.yaml           # HPA (CPU + memória)
│   ├── redis-deployment.yaml
│   ├── redis-service.yaml
│   ├── ingress.yaml
│   ├── static/
│   └── templates/
├── locust/
│   ├── locustfile.py          # Cenários de carga
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── locust-configmap.yaml
│   ├── locust-deployment.yaml
│   ├── locust-service.yaml
│   └── ingress.yaml
└── README.md
```

---

## Observabilidade

- **Prometheus**: a app expõe `/metrics` na porta 8088 (via sidecar ou ServiceMonitor, se configurado)
- **kubectl top**: uso de CPU e memória dos pods
- **kubectl get hpa**: estado atual do HPA e métricas observadas

---

## FinOps e armadilhas comuns

| Ponto                  | Observação |
|------------------------|------------|
| **maxReplicas: 40**    | Em produção, defina um limite adequado ao custo e capacidade do cluster |
| **Redis single-replica** | Ponto único de falha; para produção, considere Redis HA ou managed |
| **Requests/limits**    | Valores baixos (128Mi, 50m CPU) podem fazer o HPA escalar com pouca carga |
| **Scale down agressivo** | `stabilizationWindowSeconds: 60` reduz oscilação; aumente se notar flapping |

---

## Referências

- [Kubernetes HPA - Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Locust - Load testing](https://locust.io/)
- [OCI Native Ingress Controller](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengconfiguringingresscontroller.htm)
