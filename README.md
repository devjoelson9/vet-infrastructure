# 🐾 Sistema de Gestão Veterinária — Infraestrutura & Orquestração

Este repositório contém o Docker Compose, as variáveis de ambiente e a documentação de orquestração do **Sistema de Gestão Veterinária**.

A arquitetura usa microsserviços desacoplados com **Database-per-Service** (PostgreSQL isolado por serviço), comunicação assíncrona via **RabbitMQ** e ponto único de entrada com **API Gateway (Nginx)**.

## ✅ Status da integração

| Serviço | Situação | Observação |
|---|---|---|
| `vet-identidade-service` | ✅ Integrado | Autenticação/JWT, tutores, animais, usuários internos, e-mail (Mailpit). |
| `vet-clinico-service` | ✅ Integrado | Atendimentos, medicamentos, catálogo, prescrições e publicação de eventos. |
| `vet-agendamento-service` | ✅ Integrado | Agendamentos/disponibilidades e chamadas HTTP ao serviço de identidade. |
| `vet-frontend-web` | ✅ Integrado | Build Vite servido pelo Nginx e consumindo o gateway em `/api`. |
| `vet-api-gateway` | ✅ Integrado | Roteia `/` (frontend) e `/api/v1/**` (microsserviços). |
| `vet-faturamento-service` | ✅ Integrado | Container, banco próprio e rota de pagamentos no gateway. Consulta o atendimento/valor no clínico e conclui o agendamento via HTTP. |

## 🔗 Como os serviços conversam

```text
Navegador
   │  http://localhost:8000/
   v
┌──────────────────────┐
│  api-gateway (Nginx) │
│  :80                 │
└──────┬───────────────┘
       │ /api/v1/** (HTTP, JWT no header Authorization)
       ├──────────────────────────────► servico-identidade :8080 ──► bd-identidade
       ├──────────────────────────────► servico-clinico    :8080 ──► bd-clinico ──┐
       ├──────────────────────────────► servico-agendamento:8080 ──► bd-agendamento
       └──────────────────────────────► servico-faturamento:8080 ──► bd-faturamento
                                                │
                                                │ HTTP (RestClient, JWT repassado)
                                                ├──────────────► servico-clinico
                                                └──────────────► servico-agendamento
```

### 1. Frontend → Gateway
O frontend é compilado com `VITE_API_BASE_URL=/api` e faz chamadas relativas (`/api/v1/...`). Como ele é servido pelo mesmo Nginx do gateway, não há CORS no navegador.

### 2. Gateway → Microsserviços
O Nginx roteia pelo prefixo do recurso, preservando o path original:

| Prefixo | Destino |
|---|---|
| `/` | `interface-web:80` |
| `/api/v1/auth`, `/api/v1/tutores`, `/api/v1/admin`, `/api/v1/recepcionista`, `/api/v1/veterinario` | `servico-identidade:8080` |
| `/api/v1/atendimentos`, `/api/v1/medicamentos`, `/api/v1/catalogo-itens` | `servico-clinico:8080` |
| `/api/v1/atendimentos/{id}/pagamentos` | `servico-faturamento:8080` (regra de maior prioridade no Nginx) |
| `/api/v1/agendamentos`, `/api/v1/disponibilidades` | `servico-agendamento:8080` |
| `/api/actuator/health` | `servico-agendamento:8080/actuator/health` |

### 3. Agendamento → Identidade (síncrono)
O `IdentidadeClient` usa `RestClient` com `IDENTIDADE_BASE_URL=http://servico-identidade:8080` e repassa o header `Authorization` recebido para validar tutor/animal.

### 4. Clínico → RabbitMQ (assíncrono)
O serviço clínico publica no exchange `vet.events` (topic) as routing keys:
- `atendimento.iniciado`
- `atendimento.realizado`
- `atendimento.cancelado`

O serviço de agendamento consome esses eventos para atualizar o status do agendamento.

### 5. Faturamento → Clínico e Agendamento (síncrono)
O faturamento não mantém cópia dos atendimentos. Ao registrar um pagamento ele:
1. consulta `GET /api/v1/atendimentos/{id}` no clínico (`CLINICO_SERVICE_URL`) para validar o status `FINALIZADO` e obter o valor total calculado;
2. grava o pagamento no `db_faturamento`;
3. chama `PATCH /api/v1/agendamentos/{id}/status` (`AGENDAMENTO_SERVICE_URL`) para levar o agendamento a `CONCLUIDO`.

O header `Authorization` recebido do gateway é repassado nas duas chamadas.

> **JWT compartilhado:** identidade assina os tokens; clínico, agendamento e faturamento validam com o **mesmo** `JWT_SECRET`. Alterar o segredo em apenas um serviço quebra a autenticação entre eles.

## 📂 Estrutura de diretórios

```text
workspace-vet/
├── vet-infrastructure/              # Este repositório (orquestração Docker)
├── vet-frontend-web/                # Interface React/Vite
├── vet-api-gateway/                 # Nginx (proxy reverso)
├── vet-identidade-service/          # Spring Boot
├── vet-clinico-service/             # Spring Boot
├── vet-agendamento-service/         # Spring Boot
└── vet-faturamento-service/         # Spring Boot (faturamento e pagamentos)
```

## 🛠️ Pré-requisitos

- Docker Desktop 20.10+
- Docker Compose v2+
- Git

## 🚀 Como executar

```bash
cd vet-infrastructure
cp .env.example .env   # ajuste se necessário
docker compose up -d --build
```

Acompanhe a inicialização:

```bash
docker compose ps
docker compose logs -f
```

A primeira execução baixa dependências Maven/npm e pode levar alguns minutos. O `docker-compose.override.yml` já publica as portas de acesso local.

## 🌐 Endereços e portas

### Aplicação

| Serviço | URL local | Descrição |
|---|---|---|
| Frontend | http://localhost:8000 | Interface web (via gateway) |
| API Gateway | http://localhost:8000/api/v1 | Ponto único da API |
| RabbitMQ Management | http://localhost:15672 | Filas/exchanges (usuário/senha do `.env`) |
| Mailpit (e-mails) | http://localhost:8025 | Caixa de e-mail local (recuperação de senha) |

### Bancos (acesso com cliente SQL)

| Banco | Database | Porta no host |
|---|---|---:|
| Identidade | `db_identidade` | `5431` |
| Clínico | `db_clinico` | `5435` |
| Agendamento | `db_agendamento` | `5433` |
| Faturamento | `db_faturamento` | `5434` |

### Acesso direto aos microsserviços (debug)

| Serviço | URL local |
|---|---|
| Identidade | http://localhost:8081 |
| Clínico | http://localhost:8082 |
| Agendamento | http://localhost:8084 |
| Faturamento | http://localhost:8085 |

> Em uso normal, acesse sempre pelo gateway (`http://localhost:8000/api/v1/...`).

## 🧪 Verificando a integração

```bash
# Gateway + serviço de agendamento respondendo
curl http://localhost:8000/api/actuator/health

# Rota protegida do identidade: deve retornar 401 (prova que o gateway roteou)
curl -i http://localhost:8000/api/v1/tutores/me

# Rota protegida do clínico: deve retornar 401
curl -i http://localhost:8000/api/v1/atendimentos

# Rota protegida do agendamento: deve retornar 401
curl -i http://localhost:8000/api/v1/agendamentos

# Rota protegida do faturamento: deve retornar 401 (pagamento de um atendimento)
curl -i http://localhost:8000/api/v1/atendimentos/1/pagamentos

# Rota inexistente: 404 explícito do gateway
curl -i http://localhost:8000/api/v1/rota-inexistente
```

No RabbitMQ Management (`http://localhost:15672`), após iniciar/finalizar um atendimento, confira o exchange `vet.events`.

## 🛠️ Comandos úteis

```bash
# Parar mantendo dados
docker compose stop

# Derrubar containers e rede
docker compose down

# Reset total (apaga volumes/bancos)
docker compose down -v

# Rebuild de um serviço específico
docker compose up -d --build servico-clinico

# Rebuild do faturamento (inclui o banco na primeira subida)
docker compose up -d --build servico-faturamento

# Logs de um serviço
docker compose logs -f servico-agendamento
```

## 🩺 Problemas comuns

| Sintoma | Causa provável | Solução |
|---|---|---|
| `401` em todos os serviços | `JWT_SECRET` diferente entre containers | Use o mesmo valor no `.env` e recrie identidade, clínico, agendamento e faturamento. |
| Gateway retorna `502` | Microsserviço ainda subindo ou com erro | `docker compose logs -f servico-X`; aguarde o healthcheck. |
| Frontend chama `localhost:8080` | Build antigo com `.env` desatualizado | `docker compose build --no-cache interface-web` e suba de novo. |
| `404` em rota existente | Prefixo não mapeado no `nginx.conf` | Adicione o `location` correspondente e recrie o gateway. |
| Agendamento não acha tutor | `IDENTIDADE_BASE_URL` incorreto | Deve ser `http://servico-identidade:8080` dentro do Docker. |
| Faturamento retorna `404` ao registrar pagamento | Atendimento inexistente no serviço clínico | Confira o `id` do atendimento e o `CLINICO_SERVICE_URL`. |
| Faturamento recusa o pagamento | Atendimento ainda não está `FINALIZADO` | Finalize o atendimento no serviço clínico antes de faturar (RN14). |
| Faturamento não consegue concluir o agendamento | `AGENDAMENTO_SERVICE_URL` vazio/incorreto | Deve ser `http://servico-agendamento:8080` dentro do Docker. |

## 🛡️ Segurança

- O arquivo `.env` contém credenciais locais e **não deve ser commitado**.
- `.env.example` deve conter apenas valores genéricos.
- Em produção, remova o `docker-compose.override.yml` (ou não publique bancos/serviços internos) e troque todos os segredos.
