# QueueProcessor

Microserviço de **processamento assíncrono em filas**, escrito em **C#** e **.NET 10**, utilizando **RabbitMQ** para mensageria, **PostgreSQL** para persistência, **Entity Framework Core** como ORM, **Serilog** para logging estruturado e **xUnit** para testes automatizados.

O objetivo do `QueueProcessor` é desacoplar operações pesadas ou demoradas (como envio de e-mails, SMS, geração de relatórios, etc.) do fluxo síncrono das aplicações, garantindo **resiliência**, **escalabilidade horizontal** e **observabilidade**.

---

## Índice

- Visão Geral
- Principais Funcionalidades
- Arquitetura e Estrutura
- Tecnologias Utilizadas
- Pré-requisitos
- Como Executar em Desenvolvimento
- Execução com Docker Compose
- Testes Automatizados
- API HTTP (Jobs)
- Boas Práticas e Produção
- Roadmap
- Licença

---

## Visão Geral

O **QueueProcessor** centraliza o processamento de jobs assíncronos usando filas. Exemplos de uso:

- envio de **e-mails transacionais** (boas-vindas, recuperação de senha, notificações);
- envio de **SMS** (OTP, confirmações, alertas);
- geração de **relatórios** (PDF/Excel/CSV) em background;
- integração com sistemas externos que não podem impactar o tempo de resposta da API principal.

As aplicações clientes apenas **publicam jobs** (via API ou diretamente na fila), e o `QueueProcessor` se encarrega de:

1. Persistir o job no banco;
2. Publicar a mensagem no RabbitMQ;
3. Consumir a mensagem em um ou mais workers;
4. Controlar status, retries, dead-letter e logging.

---

## Principais Funcionalidades

- **Publicação e consumo assíncrono** de mensagens com RabbitMQ;
- **Retry automático** com controle de tentativas e marcação de falhas definitivas;
- **Dead-Letter Queue (DLQ)** para mensagens que não conseguem ser processadas;
- **Persistência de jobs** usando PostgreSQL + EF Core;
- **Logging estruturado** com Serilog (console, arquivos, integração com ferramentas como Seq);
- **Testes unitários e de integração** com xUnit, Moq e EF Core InMemory;
- **Escalabilidade horizontal**: múltiplos workers consumindo da mesma fila;
- **Pronto para containerização** com Docker e orquestração via Docker Compose/Kubernetes.

---

## Arquitetura e Estrutura

O projeto segue princípios de **Clean Architecture** e separação em camadas.

Estrutura da solução:

```text
QueueProcessor/
  src/
    QueueProcessor.Application/    # Casos de uso, orquestração de aplicação
    QueueProcessor.Domain/         # Regras de negócio (entidades, interfaces, serviços de domínio)
    QueueProcessor.Infrastructure/ # Acesso a dados, implementação de mensageria, etc.
    QueueProcessor.Worker/         # Worker service que processa filas
  tests/
    QueueProcessor.Tests/          # Testes unitários e de integração
```

> Observação: quando você adicionar uma API HTTP (por exemplo, `QueueProcessor.API`), ela será o ponto de entrada REST para enfileirar jobs e consultar status.

### Domínio

Conceitos principais (inspirados no guia do FilaProcessor que originou este projeto):

- **Job**  
  Representa um trabalho enfileirado com:
  - `Id`, `Type`, `Payload` (JSON), `Status` (`Pending`, `Processing`, `Completed`, `Failed`, `Retrying`),
  - `RetryCount`, `MaxRetries`,
  - `ErrorMessage`,
  - `CreatedAt`, `ProcessedAt`,
  - `QueueName`.

- **Mensagens de Fila**  
  Tipos fortes para os payloads de cada fila:
  - `EmailMessage` (destinatário, assunto, corpo, se é HTML),
  - `SmsMessage` (telefone, texto),
  - `ReportMessage` (tipo de relatório, formato, parâmetros).

- **Interfaces de Domínio (Ports)**  
  - `IJobRepository`: abstração de persistência de jobs;
  - `IQueuePublisher`: abstração de publicação em filas (com e sem delay);
  - `IJobProcessor`: processamento específico de um tipo de job.

### Infraestrutura

- **Banco de Dados (PostgreSQL + EF Core)**  
  - `DbContext` com mapeamento de `Job` para tabela `jobs`;
  - índices em colunas de filtro (`status`, `type`, `created_at`);
  - migrações gerenciadas via `dotnet ef`.

- **Repositórios**  
  - implementação de `IJobRepository` usando EF Core.

- **RabbitMQ**  
  - configuração via `appsettings.json` (`HostName`, `Port`, `UserName`, `Password`, nomes de filas, DLX, política de retry);
  - `ConnectionFactory` com reconexão automática;
  - criação de filas principais + DLQ, TTL, prioridade, etc.;
  - publisher responsável por:
    - serializar mensagens para JSON;
    - definir propriedades (persistência, IDs, timestamp);
    - publicar em filas normais ou de “delay” (dead-letter trick);
  - consumer genérico com:
    - `BasicQos` (prefetch);
    - `BasicAck` / `BasicNack` (envio para DLQ em caso de falha);
    - desserialização e execução de handler assíncrono.

### Workers

Workers especializados (por tipo de mensagem), normalmente derivados de uma classe base que:

- cria o `Job` ao receber a mensagem da fila;
- chama `ProcessMessageAsync` (implementado pelo worker específico);
- controla `MarkAsProcessing`, `MarkAsCompleted`, `MarkAsFailed`;
- registra logs detalhados por job.

Exemplos de workers previstos:

- `EmailWorker`: integração com SMTP/SendGrid/SES;
- `SmsWorker`: integração com Twilio/SNS/etc.;
- `ReportWorker`: geração de relatórios e escrita em disco/armazenamento externo.

---

## Tecnologias Utilizadas

- **.NET 10** (Worker Service e futuro Web API);
- **C#**;
- **RabbitMQ** (mensageria);
- **PostgreSQL** (banco relacional);
- **Entity Framework Core** (ORM);
- **Serilog** (logging estruturado);
- **xUnit, Moq, FluentAssertions** (testes);
- **Docker & Docker Compose** (containerização e orquestração local).

---

## Pré-requisitos

- **.NET 10 SDK**;
- **Docker** e **Docker Compose** (para subir infraestrutura local rapidamente);
- opcional:
  - RabbitMQ e PostgreSQL instalados localmente;
  - ferramentas para consulta ao banco (psql, DBeaver, etc.).

Verifique seu .NET:

```bash
dotnet --version
```

---

## Como Executar em Desenvolvimento

> Ajuste os caminhos dos projetos conforme o layout real da sua solução.

### 1. Subir infraestrutura (RabbitMQ + PostgreSQL) com Docker

```bash
# RabbitMQ com Management UI
docker run -d \
  --hostname rabbitmq-dev \
  --name rabbitmq \
  -p 5672:5672 \
  -p 15672:15672 \
  -e RABBITMQ_DEFAULT_USER=admin \
  -e RABBITMQ_DEFAULT_PASS=admin123 \
  rabbitmq:3-management

# PostgreSQL
docker run -d \
  --name postgres-dev \
  -p 5432:5432 \
  -e POSTGRES_USER=queueprocessor \
  -e POSTGRES_PASSWORD=queue123 \
  -e POSTGRES_DB=queueprocessordb \
  postgres:15-alpine
```

### 2. Restaurar pacotes e compilar

Na raiz da solução:

```bash
dotnet restore
dotnet build
```

### 3. Aplicar migrações (quando o projeto de infraestrutura estiver configurado)

```bash
dotnet ef database update \
  --project src/QueueProcessor.Infrastructure \
  --startup-project src/QueueProcessor.Worker
```

### 4. Rodar Worker

```bash
dotnet run --project src/QueueProcessor.Worker
```

> Quando houver uma API HTTP (por exemplo `QueueProcessor.API`), você poderá executá-la em paralelo para expor endpoints REST de enfileiramento e consulta.

---

## Execução com Docker Compose

Se houver um `docker-compose.yml` no projeto, você poderá subir tudo de uma vez:

```bash
# Subir todo o stack
docker compose up -d

# Ver logs dos workers
docker compose logs -f queueprocessor-worker

# Escalar quantidade de workers
docker compose up -d --scale queueprocessor-worker=5

# Derrubar tudo
docker compose down
```

---

## Testes Automatizados

Na raiz da solução ou pasta de testes:

```bash
dotnet test --verbosity normal
```

Se houver configuração de cobertura:

```bash
dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=lcov
```

---

## API HTTP (Jobs)

Quando a API for adicionada (por exemplo `QueueProcessor.API`), os endpoints previstos são:

- `POST /api/jobs/email` — enfileira um job de e-mail;
- `POST /api/jobs/sms` — enfileira um job de SMS;
- `GET /api/jobs/{id}` — consulta um job específico;
- `GET /api/jobs/stats` — retorna estatísticas agregadas de jobs (pendentes, concluídos, falhos);
- `GET /api/jobs?status=Pending` — lista jobs filtrados por status.

Exemplo de chamada (HTTP):

```bash
curl -X POST http://localhost:5000/api/jobs/email \
  -H "Content-Type: application/json" \
  -d '{
    "to": "cliente@empresa.com",
    "subject": "Bem-vindo ao QueueProcessor!",
    "body": "<h1>Olá!</h1><p>Seu cadastro foi confirmado.</p>",
    "isHtml": true
  }'
```

---

## Boas Práticas e Produção

O `QueueProcessor` foi desenhado com foco em produção:

- **Escalabilidade horizontal** via múltiplos workers e ajuste de `prefetch` no RabbitMQ;
- **Resiliência**:
  - retry com controle de tentativas;
  - Dead-Letter Queue monitorável;
  - possível uso de **Polly** para circuit breaker em integrações externas;
- **Observabilidade**:
  - logging estruturado com Serilog (console, arquivos, Seq, etc.);
  - integração com Prometheus/Grafana para métricas;
- **Health Checks**:
  - endpoints de liveness/readiness para RabbitMQ e PostgreSQL;
- **Segurança**:
  - segredos em gerenciadores de secrets (Key Vault / Secrets Manager);
  - TLS/SSL entre serviços em produção;
- **Operação**:
  - backups agendados do banco;
  - alertas em cima de DLQ e falhas recorrentes.

---

## Roadmap

Possíveis evoluções:

- adicionar `QueueProcessor.API` expondo todos os endpoints REST;
- dashboard para monitoramento visual de jobs e filas;
- suporte a novos tipos de jobs (webhooks, integrações específicas, etc.);
- métricas expostas em `/metrics` para Prometheus;
- rate limiting e quotas por cliente na API;
- pipeline de CI/CD com build, testes, análise estática e deploy automatizado.

---

## Licença

Defina aqui a licença desejada (MIT, Apache 2.0, uso interno, etc.).

