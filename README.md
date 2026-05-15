# MS-Dispatcher

Arquitetura de microsserviços robusta desenvolvida em **Spring Boot 3** e **Java 17**, focada no desacoplamento de componentes através de mensageria assíncrona com **RabbitMQ**.

---

## 📋 Descrição

O **MS-Dispatcher** gerencia o fluxo de cadastro de usuários com disparo reativo e assíncrono de notificações de boas-vindas. O design isola os domínios de negócio para garantir tolerância a falhas e escalabilidade linear.

### Fluxo Arquitetural

```text
  [ Cliente REST ]
         │
         ▼
 ┌────────────────────────┐
 │   ms-dispatcher-user   │ (User Service)
 └───────────┬────────────┘
             │
             ▼ (Publica evento)
     ┌──────────────┐
     │  RabbitMQ    │ [Fila: email-queue]
     └───────┬──────┘
             │
             ▼ (Consome evento)
 ┌────────────────────────┐
 │  ms-dispatcher-email   │ (Email Service)
 └────────────────────────┘
```

---

## 🏗️ Estrutura do Projeto

```text
ms-dispatcher/
├── ms-dispatcher-user/            # Microsserviço de Usuários (Produtor)
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/dev/java10x/dispatcher/user/
│   │   │   │   ├── UserApplication.java
│   │   │   │   ├── controller/      # Endpoints HTTP da API
│   │   │   │   ├── service/         # Camada de regras de negócio
│   │   │   │   ├── domain/          # Entidades de banco de dados
│   │   │   │   ├── dto/             # Objetos de transferência de dados
│   │   │   │   ├── repository/      # Interfaces de persistência JPA
│   │   │   │   ├── producer/        # Publicadores de eventos RabbitMQ
│   │   │   │   └── config/          # Instanciação de filas e exchanges
│   │   │   └── resources/
│   │   │       ├── application.yml  # Configurações do ambiente
│   │   │       └── db/migration/    # Versionamento de schema Flyway
│   │   └── test/
│   ├── docker-compose.yml           # Infraestrutura local (DB e Broker)
│   └── pom.xml
│
└── ms-dispatcher-email/           # Microsserviço de E-mails (Consumidor)
    ├── src/
    │   ├── main/
    │   │   ├── java/dev/java10x/dispatcher/email/
    │   │   │   ├── EmailApplication.java
    │   │   │   ├── consumer/        # Listeners e consumidores RabbitMQ
    │   │   │   ├── service/         # Regras e integração com SMTP
    │   │   │   ├── domain/          # Entidades e histórico de envios
    │   │   │   ├── dto/             # DTOs de entrada e mapeamento
    │   │   │   ├── enums/           # Estados do envio (SENT, ERROR)
    │   │   │   ├── repository/      # Auditoria de e-mails disparados
    │   │   │   └── config/          # Configurações de conexão à fila
```

---

## 🛠️ Tecnologias Principais

*   **Java 17 & Spring Boot 3** (Core da Aplicação)
*   **Spring AMQP / RabbitMQ** (Message Broker Assíncrono)
*   **PostgreSQL / MySQL** (Bancos Relacionais Isolados)
*   **Flyway Migration** (Evolução de Banco de Dados Automatizada)
*   **Docker & Docker Compose** (Containerização de Infraestrutura)

---

## ⚡ Como Executar o Projeto

### 1. Subir a Infraestrutura (Banco de Dados e RabbitMQ)
Navegue até a pasta do microsserviço de usuários que contém o arquivo `docker-compose.yml` e execute:
```bash
docker compose up -d
```

### 2. Executar o Microsserviço de Usuários
Abra o terminal na pasta `ms-dispatcher-user/` e execute:
```bash
./mvnw spring-boot:run
```

### 3. Executar o Microsserviço de E-mail
Abra o terminal na pasta `ms-dispatcher-email/` e execute:
```bash
./mvnw spring-boot:run
```
