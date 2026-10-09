# 🐾 Sistema de Gestão Veterinária — Infraestrutura & Orquestração

Este repositório contém os scripts de orquestração, variáveis de ambiente e arquivos do Docker Compose responsáveis por provisionar a infraestrutura completa do **Sistema de Gestão Veterinária**.

A arquitetura é baseada em microsserviços desacoplados, utilizando o padrão **Database-per-Service** (PostgreSQL isolado por serviço), comunicação assíncrona por mensageria via **RabbitMQ** e ponto único de entrada através do **API Gateway (Nginx)**.

## 📐 Visão Geral da Arquitetura (C4 — Nível 2)

```text
                               +-------------------+
                               |   Interface Web   |
                               +---------+---------+
                                         | (HTTPS / JSON)
                                         v
                               +-------------------+
                               |    API Gateway    | (Nginx)
                               +----+----+----+----+
                                    |    |    |
        +---------------------------+    |    +---------------------------+
        | /identidade                    | /clinico                       | /agendamentos & /faturamento
        v                                v                                v
+---------------+                +---------------+                +-----------------------------+
| Identidade    |                | Serviços      |                | Agendamento & Faturamento   |
| Service       |                | Clínicos      |                | Services                    |
+-------+-------+                +-------+-------+                +--------------+--------------+
        |                                |                                       ^
        | SQL                            | AMQP (AtendimentoRealizado)           | AMQP (Consumo)
        v                                v                                       |
+---------------+                +---------------+                       +-------+-------+
| BD Identidade |                | Message Broker|---------------------->| RabbitMQ      |
| (PostgreSQL)  |                | (RabbitMQ)    |                       +---------------+
+---------------+                +---------------+
```

### Componentes mapeados

- **API Gateway (Nginx):** roteia todas as chamadas HTTP externas para os microsserviços internos.
- **Serviço de Identidade e Cadastro:** gerencia autenticação, usuários, perfis, tutores e animais.
- **Serviço Clínico:** gerencia prontuários, exames, atendimentos e catálogo de serviços.
- **Serviço de Agendamento:** gerencia agendas, disponibilidades e check-in.
- **Serviço de Faturamento e Pagamentos:** gerencia cobranças, faturas e pagamentos.
- **Message Broker (RabbitMQ):** permite comunicação assíncrona orientada a eventos, como `AtendimentoRealizado`.
- **Bancos de dados (PostgreSQL):** bancos independentes e isolados por microsserviço.

## 📂 Estrutura de diretórios recomendada

Para executar o ambiente local, os repositórios dos microsserviços devem estar clonados no mesmo diretório pai (`workspace-vet/`):

```text
workspace-vet/
├── vet-infrastructure/        # Repositório atual (orquestração Docker)
├── vet-frontend-web/           # Interface gráfica (HTML/CSS/JS)
├── vet-api-gateway/            # Arquivos do Nginx
├── vet-identidade-service/     # Microsserviço Spring Boot
├── vet-clinico-service/        # Microsserviço Spring Boot
├── vet-agendamento-service/    # Microsserviço Spring Boot
└── vet-faturamento-service/    # Microsserviço Spring Boot
```

## 🛠️ Pré-requisitos

Certifique-se de ter instalado em sua máquina:

- Docker Desktop (v20.10+)
- Docker Compose (v2.0+)
- Git
- Beekeeper Studio (opcional, para gestão gráfica dos bancos SQL)

## 🚀 Como executar o ambiente

### 1. Configurar as variáveis de ambiente

Copie o arquivo de exemplo `.env.example` para criar seu `.env` local:

```bash
cd vet-infrastructure
cp .env.example .env
```

Opcionalmente, edite o arquivo `.env` para alterar senhas ou portas padrão.

### 2. Subir todos os containers

Execute o comando do Docker Compose para construir as imagens dos repositórios e iniciar os serviços em segundo plano:

```bash
docker compose up -d --build
```

### 3. Verificar o status dos containers

Acompanhe a inicialização e o estado dos *healthchecks*:

```bash
docker compose ps
```

Para visualizar os logs unificados de todos os serviços:

```bash
docker compose logs -f
```

## 🌐 Mapeamento de endereços e portas

### Serviços principais

| Serviço | URL de acesso local | Descrição |
|---|---|---|
| Aplicação Web (Frontend) | `http://localhost` | Interface de usuário para recepção, veterinários e administradores |
| API Gateway | `http://localhost/api` | Ponto único de roteamento HTTP |
| RabbitMQ Management | `http://localhost:15672` | Interface web para monitoramento de filas e exchanges |

### Conexões de banco de dados

Os bancos utilizam portas mapeadas pelo `docker-compose.override.yml`. Para conectar seu cliente SQL (Beekeeper Studio ou DBeaver), use `localhost` como host:

| Banco de dados | Database | Porta no host | Usuário padrão |
|---|---|---:|---|
| BD Identidade | `db_identidade` | `5431` | `vet_user` |
| BD Clínico | `db_clinico` | `5432` | `vet_user` |
| BD Agendamento | `db_agendamento` | `5433` | `vet_user` |
| BD Faturamento | `db_faturamento` | `5434` | `vet_user` |

## 🛠️ Comandos úteis de manutenção

**Parar a execução mantendo os dados intactos:**

```bash
docker compose stop
```

**Derrubar containers e liberar redes:**

```bash
docker compose down
```

**Derrubar tudo e limpar todos os volumes/bancos de dados (reset total):**

> ⚠️ Este comando remove os volumes associados e apaga os dados persistidos dos bancos.

```bash
docker compose down -v
```

**Reconstruir apenas um microsserviço específico após alteração no código:**

```bash
docker compose up -d --build servico-clinico
```

## 🛡️ Segurança

- O arquivo `.env` contém senhas de acesso local e **nunca deve ser commitado** no controle de versão.
- O arquivo `.env.example` serve apenas como referência das chaves necessárias e deve ser mantido atualizado com dados genéricos.
