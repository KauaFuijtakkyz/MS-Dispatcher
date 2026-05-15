# 📦 Guia de Instalação Detalhado

## Pré-requisitos

### Windows
1. **Docker Desktop** - [Download](https://www.docker.com/products/docker-desktop)
2. **Java Development Kit (JDK) 17+** - [Download](https://www.oracle.com/java/technologies/downloads/#java17)
3. **Maven 3.6+** ou usar o `mvnw` bundled
4. **Git** - [Download](https://git-scm.com/download/win)

### macOS
```bash
# Instalar Homebrew se não tiver
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Instalar dependências
brew install openjdk@17 docker maven git
```

### Linux (Ubuntu/Debian)
```bash
sudo apt-get update
sudo apt-get install -y openjdk-17-jdk docker.io docker-compose maven git

# Dar permissões ao Docker
sudo usermod -aG docker $USER
```

## Passo 1: Verificar Instalações

```bash
# Java
java -version
# Deve ser >= 17

# Maven
mvn -version
# Ou se usar bundled
./mvnw -version

# Docker
docker --version
docker-compose --version

# Git
git --version
```

## Passo 2: Clonar o Repositório

```bash
git clone https://github.com/seu-usuario/user-mail-sender-ms.git
cd user-mail-sender-ms-main
```

## Passo 3: Configurar Variáveis de Ambiente

### Windows (PowerShell)
```powershell
# Copiar arquivo de exemplo
Copy-Item .env.example .env

# Editar o arquivo com seu editor favorito
notepad .env
```

**Conteúdo do `.env`:**
```env
RABBITMQ_USERNAME=pofhowkh_seu_usuario
RABBITMQ_PASSWORD=sua_senha_secreta
EMAIL_USERNAME=seu_email@gmail.com
EMAIL_PASSWORD=sua_app_password_16_chars
```

### macOS/Linux
```bash
cp .env.example .env
nano .env  # ou vim .env
```

## Passo 4: Obter Credenciais

### RabbitMQ (CloudAMQP)

1. Acesse [CloudAMQP](https://www.cloudamqp.com/)
2. Crie uma conta gratuita
3. Crie uma instância (Tiger: FREE)
4. Copie:
   - **Host**: `duck.lmq.cloudamqp.com`
   - **Username**: Do painel
   - **Password**: Do painel
   - **Virtual Host**: `pofhowkh` (ou seu)

### Gmail SMTP

1. Acesse [Google Account Security](https://myaccount.google.com/security)
2. Ative verificação em 2 passos
3. Gere [App Password](https://support.google.com/accounts/answer/185833)
4. Use o email e a senha gerada no `.env`

## Passo 5: Iniciar Bancos de Dados com Docker

### User Service Database
```bash
cd user
docker-compose up -d

# Verificar se está rodando
docker ps | findstr ms-user-db
```

### Email Service Database
```bash
cd ../email
docker-compose up -d

# Verificar
docker ps | findstr ms-email-db
```

Portas:
- User DB: `5435`
- Email DB: `5433`

## Passo 6: Compilar e Executar

### Opção 1: Abrir em 2 Terminais

**Terminal 1 - User Service:**
```bash
cd user
./mvnw spring-boot:run
# Ou no Windows:
# mvnw.cmd spring-boot:run
```

**Terminal 2 - Email Service:**
```bash
cd email
./mvnw spring-boot:run
# Ou no Windows:
# mvnw.cmd spring-boot:run
```

### Opção 2: Iniciar em Background

#### Windows (PowerShell)
```powershell
# User Service
Start-Process powershell -ArgumentList "-NoExit", "-Command", "cd user; mvnw.cmd spring-boot:run"

# Email Service
Start-Process powershell -ArgumentList "-NoExit", "-Command", "cd ../email; mvnw.cmd spring-boot:run"
```

#### macOS/Linux
```bash
cd user && ./mvnw spring-boot:run &
cd ../email && ./mvnw spring-boot:run &
```

## Passo 7: Verificar Instalação

```bash
# User Service Health
curl http://localhost:8081/actuator/health

# Email Service Health
curl http://localhost:8080/actuator/health

# Swagger UI
# User: http://localhost:8081/swagger-ui.html
# Email: http://localhost:8080/swagger-ui.html
```

## Passo 8: Testar Fluxo Completo

### Criar um Usuário (REST)
```bash
curl -X POST http://localhost:8081/users \
  -H "Content-Type: application/json" \
  -d '{
    "name": "João Silva",
    "email": "joao@gmail.com"
  }'
```

### Verificar Logs
A resposta deve indicar que:
1. Usuário foi criado
2. Evento foi publicado na fila
3. Email foi enviado (verificar pasta de spam!)

## Troubleshooting

### Erro: "Port already in use"
```bash
# Encontrar processo usando porta
# Windows
netstat -ano | findstr :8081

# Matar processo
taskkill /PID <PID> /F

# macOS/Linux
lsof -i :8081
kill -9 <PID>
```

### Erro: "Connection refused" no RabbitMQ
```bash
# Verifique variáveis de ambiente
echo $RABBITMQ_USERNAME
echo $RABBITMQ_PASSWORD

# Teste conexão
# Use: https://www.rabbitmq.com/downloads.html
```

### Erro: PostgreSQL connection timeout
```bash
# Verifique se Docker está rodando
docker ps

# Se não estiver, reinicie:
docker-compose restart
```

### Erro: "App Password" inválido
1. Volte à página de App Passwords do Google
2. Delete a antiga e gere uma nova
3. Certifique-se de copiar corretamente (17 caracteres)

## Build e Deploy

### Construir JAR
```bash
cd user
./mvnw clean package
# Arquivo: target/user-0.0.1-SNAPSHOT.jar

cd ../email
./mvnw clean package
# Arquivo: target/email-0.0.1-SNAPSHOT.jar
```

### Executar JAR Diretamente
```bash
java -jar user/target/user-0.0.1-SNAPSHOT.jar
java -jar email/target/email-0.0.1-SNAPSHOT.jar
```

### Docker Image
```bash
# Build
docker build -t user-service user/
docker build -t email-service email/

# Run
docker run -p 8081:8081 -e RABBITMQ_USERNAME=$RABBITMQ_USERNAME user-service
docker run -p 8080:8080 -e RABBITMQ_USERNAME=$RABBITMQ_USERNAME email-service
```

## Desenvolvimento

### IDE Recomendado
- **IntelliJ IDEA** - [Download](https://www.jetbrains.com/idea/)
- **VS Code** com extensions Java - [Guide](https://code.visualstudio.com/docs/java/getting-started)

### Setup IntelliJ
1. File → New → Project from Version Control
2. Colar URL do repositório
3. Abrir `pom.xml` como projeto
4. Aguardar Maven indexar dependências
5. Run → Edit Configurations
6. Adicionar 2 Spring Boot configurations

## Verificação de Segurança

### Credenciais em Git
```bash
# Não fazer commit de .env
echo ".env" >> .gitignore
git add .gitignore
git commit -m "Add .env to gitignore"
```

### Dependências Vulneráveis
```bash
# Verificar vulnerabilidades
mvn org.owasp:dependency-check-maven:check
```

## Próximos Passos

1. ✅ Todos os serviços rodando
2. ✅ Criar usuário via API
3. ✅ Verificar email recebido
4. 📚 Ler documentação
5. 🧪 Executar testes
6. 💻 Começar desenvolvimentos

---

Para dúvidas, abra uma issue no repositório!

