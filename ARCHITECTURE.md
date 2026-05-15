# 🏗️ Arquitetura e Design

## Visão Geral da Arquitetura

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Frontend / Cliente REST                          │
└───────────────────────────┬─────────────────────────────────────────────┘
                            │ HTTP POST /users
                            │
        ┌─────────────────────────────────────────────────────┐
        │                                                      │
┌───────▼──────────────────────────────────────────────────────▼───────────┐
│                        USER MICROSERVICES (8081)                         │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  ┌─────────────────────────────────────────────────────────┐            │
│  │              UserController (REST API)                  │            │
│  │  - POST   /users          (Create User)                │            │
│  │  - GET    /list/users     (List All Users)             │            │
│  └────────────────────┬────────────────────────────────────┘            │
│                       │                                                   │
│  ┌────────────────────▼────────────────────────────────────┐            │
│  │              UserService (Business Logic)               │            │
│  │  - saveAndPublish(UserModel)                           │            │
│  │  - getAllUsers()                                       │            │
│  └────────────────────┬────────────────────────────────────┘            │
│                       │                                                   │
│           ┌───────────┴───────────┐                                     │
│           │                       │                                      │
│  ┌────────▼──────────┐  ┌────────▼────────────────────────┐            │
│  │ UserRepository    │  │  UserProducer (RabbitMQ)        │            │
│  │ (JPA/Database)    │  │  - publishEvent()               │            │
│  └───────────────────┘  └────────┬──────────────────────┬─┘            │
│                                   │                      │              │
│        ┌──────────────────────────┘                      │              │
│        │                                                  │              │
│  ┌─────▼──────────────────────────┐                   │              │
│  │   PostgreSQL Database           │                   │              │
│  │   - tb_users                    │                   │              │
│  │   - Migrations (Flyway)         │                   │              │
│  └────────────────────────────────┘                   │              │
│                                                         │              │
└─────────────────────────────────────────────────────────┼──────────────┘
                                                          │
                              ┌───────────────────────────┘
                              │
                              │ RabbitMQ Message
                              │ (email-queue)
                              │
        ┌─────────────────────▼──────────────────────────────┐
        │                                                     │
┌───────▼────────────────────────────────────────────────────▼──────────┐
│                    EMAIL MICROSERVICES (8080)                         │
├────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────────────────────────────────────────────┐             │
│  │      EmailConsumer (RabbitMQ Listener)               │             │
│  │  - @RabbitListener(queues = \"email-queue\")       │             │
│  │  - Consome EmailDto da fila                          │             │
│  └──────────────────┬─────────────────────────────────┘             │
│                     │                                                │
│  ┌──────────────────▼─────────────────────────────────┐             │
│  │       EmailService (Business Logic)                │             │
│  │  - sendEmail(EmailModel)                          │             │
│  │  - updateStatus(EmailStatus)                      │             │
│  │  - Valida e envia email via SMTP                  │             │
│  └──────────────────┬─────────────────────────────────┘             │
│                     │                                                │
│         ┌───────────┴───────────┐                                  │
│         │                       │                                   │
│  ┌──────▼─────────┐  ┌─────────▼───────────────────┐             │
│  │EmailRepository │  │   JavaMailSender (SMTP)      │             │
│  │(JPA/Database)  │  │  - host: smtp.gmail.com      │             │
│  └────────────────┘  │  - port: 587                 │             │
│                      └──────────┬────────────────────┘             │
│        ┌─────────────────────── │                                 │
│        │                        │                                  │
│  ┌─────▼──────────────────┐  │  RFC 5587 (SMTP + TLS)          │
│  │  PostgreSQL Database    │  │                                  │
│  │  - tb_email             │  └────────────────────┐             │
│  │  - Migrations (Flyway)  │                       │             │
│  └───────────────────────┘                        │             │
│                                                   │             │
│                                      ┌────────────▼─────┐       │
│                                      │   Gmail Servers   │       │
│                                      │ (Envio de emails) │       │
│                                      └──────────────────┘       │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
                                     ▲
                                     │
                        Recebe email / Notificação
                                     │
                        ┌────────────▼────────────┐
                        │  Caixa de Email do      │
                        │  Usuário                │
                        └─────────────────────────┘
```

## Fluxo de Dados Completo

### Passo 1: Criação de Usuário

```json
Cliente HTTP POST /users
{
  "name": "João Silva",
  "email": "joao@gmail.com"
}
```

### Passo 2: Salvamento no Banco

```sql
INSERT INTO tb_users (name, email) VALUES ('João Silva', 'joao@gmail.com');
```

### Passo 3: Publicação na Fila (RabbitMQ)

```json
{
  "id": 1,
  "name": "João Silva",
  "email": "joao@gmail.com",
  "created_at": "2026-05-15T10:30:00",
  "timestamp": 1715849400000
}
```

### Passo 4: Consumo da Fila (Email Service)

```
EmailConsumer recebe mensagem
→ Converte em EmailModel
→ Passa para EmailService
```

### Passo 5: Envio de Email

```
EmailService
→ Valida dados
→ Monta corpo do email
→ Conecta ao SMTP Gmail
→ Envia email
→ Atualiza status em BD (SENT/FAILED)
```

## Estrutura de Camadas

### User Service

```
┌───────────────────────────────────────────┐
│          PRESENTATION LAYER               │
│  - UserController                         │
│  - UserDto                                │
└──────────────────┬────────────────────────┘
                   │
┌──────────────────▼────────────────────────┐
│          BUSINESS LAYER                   │
│  - UserService                            │
│  - Validações de negócio                  │
│  - Orquestração                           │
└──────────────────┬────────────────────────┘
                   │
┌──────────────────▼────────────────────────┐
│          APPLICATION LAYER                │
│  - UserProducer (RabbitMQ)                │
│  - UserRepository                        │
└──────────────────┬────────────────────────┘
                   │
┌──────────────────▼────────────────────────┐
│          DATA LAYER                       │
│  - PostgreSQL                             │
│  - Migrations (Flyway)                    │
│  - tb_users                               │
└───────────────────────────────────────────┘
```

### Email Service

```
┌───────────────────────────────────────────┐
│          MESSAGE LAYER                    │
│  - EmailConsumer (RabbitMQ Listener)      │
└──────────────────┬────────────────────────┘
                   │
┌──────────────────▼────────────────────────┐
│          BUSINESS LAYER                   │
│  - EmailService                           │
│  - Validações de email                    │
│  - Tratamento de erros                    │
└──────────────────┬────────────────────────┘
                   │
┌──────────────────▼────────────────────────┐
│          INTEGRATION LAYER                │
│  - JavaMailSender (SMTP)                  │
│  - EmailRepository                        │
│  - Status Manager                         │
└──────────────────┬────────────────────────┘
                   │
┌──────────────────▼────────────────────────┐
│          DATA & EXTERNAL LAYER            │
│  - PostgreSQL                             │
│  - SMTP Gmail                             │
│  - Migrations (Flyway)                    │
│  - tb_email                               │
└───────────────────────────────────────────┘
```

## Padrões de Design

### 1. Producer-Consumer Pattern

```
User Service                    RabbitMQ                    Email Service
    │                              │                              │
    ├─ Criar usuário              │                              │
    ├─ Publicar evento ──────────→ [email-queue] ──────────────→ ├─ Consumir evento
    │                              └─────────┘                    ├─ Processar
    │                                                             ├─ Enviar email
```

### 2. Repository Pattern

```
Service
    │
    └─→ Repository Interface
            │
            ├─→ UserRepository (Spring Data JPA)
            │       │
            │       └─→ PostgreSQL
            │
            └─→ EmailRepository (Spring Data JPA)
                    │
                    └─→ PostgreSQL
```

### 3. Injeção de Dependência

```java
@Service
public class UserService {
    // ✅ Construtor (recomendado)
    private final UserRepository userRepository;
    private final UserProducer userProducer;
    
    public UserService(UserRepository repo, UserProducer producer) {
        this.userRepository = repo;
        this.userProducer = producer;
    }
}
```

### 4. DTO Pattern

```
JSON (Client)
    │
    └─→ UserDto (Transferência)
            │
            ├─→ Validação
            │
            └─→ UserModel (Entidade)
                    │
                    └─→ Banco de Dados
```

## Configurações RabbitMQ

### Exchange, Queue e Binding

```
Exchange: user-exchange (type: direct/topic)
    │
    └─→ Routing Key: user.email.send
            │
            └─→ Queue: email-queue
                    │
                    └─→ Listener: EmailConsumer
```

### Message Format

```json
{
  "id": 1,
  "name": "João Silva",
  "email": "joao@gmail.com",
  "timestamp": 1715849400000,
  "headers": {
    "contentType": "application/json"
  }
}
```

## Transações e Consistência

### User Service Transação

```sql
BEGIN TRANSACTION
  INSERT INTO tb_users (...)
  PUBLISH MESSAGE TO RabbitMQ
COMMIT
```

### Email Service Transação

```sql
BEGIN TRANSACTION
  UPDATE tb_email SET status = 'SENDING'
  ENVIAR EMAIL VIA SMTP
  UPDATE tb_email SET status = 'SENT'
COMMIT
```

## Segurança

### Camadas de Segurança

1. **AMQP SSL/TLS**: Conexão criptografada com RabbitMQ
2. **SMTP TLS**: Conexão criptografada com Gmail
3. **Variáveis de Ambiente**: Credenciais não versionadas
4. **Validação de Entrada**: DTOs com @Valid e annotations

### Fluxo de Credenciais

```
.env (Local - não versionado)
  │
  ├─→ Application.yml (referencia ${VAR})
  │
  └─→ Spring Boot (carrega em tempo de execução)
        │
        ├─→ RabbitMQ Configuration
        │
        └─→ Mail Configuration
```

## Performance e Escalabilidade

### Pontos de Escalabilidade

1. **Múltiplas instâncias do User Service**
   - Compartilham mesmo banco de dados
   - Load Balancer na frente

2. **Múltiplas instâncias do Email Service**
   - Consomem da mesma fila
   - Processamento em paralelo
   - Sem duplicação (RabbitMQ acknowledgment)

3. **RabbitMQ Clustering**
   - Alta disponibilidade
   - Persistência de mensagens

### Throughput Esperado

```
User Service:
- Cria usuário: ~100 req/s
- Publica evento: ~1000 msg/s

Email Service:
- Consome eventos: ~100 msg/s
- Envia emails: ~50 emails/s (limitado por SMTP)
```

## Tratamento de Erros

### User Service

```
Erro de Validação
    │
    └─→ Status 400 (Bad Request)

Erro de BD
    │
    └─→ Status 500 (Internal Server Error)
        └─→ Log de erro
```

### Email Service

```
Erro ao Consumir Mensagem
    │
    └─→ Dead Letter Queue (DLQ)
        └─→ Manual análise

Erro ao Enviar Email
    │
    ├─→ Retry automático (3x)
    │
    └─→ Status FAILED no BD
```

## Observabilidade

### Logs

```
[User Service]
INFO: User created: user_id=1, email=joao@gmail.com
INFO: Event published: queue=email-queue, routing_key=user.email.send

[Email Service]  
INFO: Message consumed from queue
INFO: Sending email to: joao@gmail.com
INFO: Email sent successfully, status=SENT
```

### Métricas (Futuro)

```
- Requisições HTTP por endpoint
- Tempo médio de resposta
- Taxa de mensagens processadas
- Taxa de emails enviados com sucesso
- Taxa de erros
```

## Autoscaling (Futuro)

```
Kubernetes/Docker Swarm
    │
    ├─→ User Service: 1-5 replicas
    │       └─→ Escalável por CPU/Memória
    │
    └─→ Email Service: 2-10 replicas
            └─→ Escalável por tamanho da fila
```

---

Para mais detalhes, consulte o README.md e INSTALL.md

