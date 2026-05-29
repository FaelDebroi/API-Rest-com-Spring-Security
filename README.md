# API Rest com Spring Security + Microsserviços com RabbitMQ

Projeto acadêmico desenvolvido para a disciplina **Web 3 — IFSP**.  
Contém três microsserviços Spring Boot independentes organizados em um único repositório.

---

## Estrutura do Repositório

```
├── user-service/   → Autenticação e autorização com JWT (porta 8081)
├── ms-user/        → Cadastro de usuários com publicação no RabbitMQ (porta 8081)
└── ms-email/       → Consumidor RabbitMQ que envia e-mail e persiste no banco (porta 8082)
```

---

## user-service — Spring Security + JWT

Serviço de autenticação stateless com controle de acesso por roles.

### Tecnologias
- Spring Boot 3.2.4
- Spring Security 6
- JWT (jjwt 0.11.5)
- Spring Data JPA
- Spring AMQP
- MySQL

### Configuração (`application.properties`)

```properties
server.port=8081
spring.datasource.url=jdbc:mysql://localhost:3306/ms_user?useSSL=false&serverTimezone=UTC
spring.datasource.username=root
spring.datasource.password=SUA_SENHA
spring.jpa.hibernate.ddl-auto=update
jwt.secret=SEU_SECRET_256_BITS
jwt.expiration=86400000
```

### Entidades

| Classe | Descrição |
|--------|-----------|
| `User` | Usuário com email, senha (BCrypt) e roles |
| `Role` | Perfil de acesso |
| `RoleName` | Enum: `ROLE_ADMINISTRATOR`, `ROLE_CUSTOMER` |

### Endpoints

| Método | URL | Auth | Descrição |
|--------|-----|------|-----------|
| `POST` | `/users` | Público | Criar usuário |
| `POST` | `/users/login` | Público | Autenticar e obter token JWT |
| `GET` | `/users/test/customer` | ROLE_CUSTOMER | Rota protegida |
| `GET` | `/users/test/administrator` | ROLE_ADMINISTRATOR | Rota protegida |

### Exemplos de uso

**Criar usuário:**
```json
POST /users
{
  "email": "joao@email.com",
  "password": "123456",
  "role": "ROLE_CUSTOMER"
}
```

**Login:**
```json
POST /users/login
{
  "email": "joao@email.com",
  "password": "123456"
}
```
Resposta:
```json
{ "token": "eyJhbGciOiJIUzI1NiJ9..." }
```

**Acessar rota protegida:**
```
GET /users/test/customer
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
```

---

## ms-user + ms-email — Microsserviços com RabbitMQ

Arquitetura baseada em eventos: o `ms-user` cadastra um usuário e publica uma mensagem na fila do RabbitMQ. O `ms-email` consome a mensagem, envia o e-mail e persiste o registro no banco.

```
POST /users
     │
  ms-user (8081)         ──►  RabbitMQ: default.email  ──►  ms-email (8082)
  MySQL: ms_user                                             MySQL: ms_email
  TB_USERS                                                   TB_EMAILS
                                                              │
                                                         Gmail SMTP
```

### ms-user

#### Tecnologias
- Spring Boot 3.2.4
- Spring Data JPA
- Spring AMQP (RabbitMQ)
- MySQL

#### Configuração (`application.properties`)

```properties
server.port=8081
spring.datasource.url=jdbc:mysql://localhost:3306/ms_user?useSSL=false&serverTimezone=UTC
spring.datasource.username=root
spring.datasource.password=SUA_SENHA
spring.jpa.hibernate.ddl-auto=update
spring.rabbitmq.addresses=amqps://USUARIO:SENHA@HOST/VHOST
broker.queue.email.name=default.email
```

#### Endpoint

| Método | URL | Descrição |
|--------|-----|-----------|
| `POST` | `/users` | Cadastra usuário e dispara evento de e-mail |

**Corpo da requisição:**
```json
{
  "name": "João Silva",
  "email": "joao@email.com"
}
```

---

### ms-email

#### Tecnologias
- Spring Boot 3.2.4
- Spring Data JPA
- Spring AMQP (RabbitMQ)
- Spring Mail (JavaMailSender)
- MySQL

#### Configuração (`application.properties`)

```properties
server.port=8082
spring.datasource.url=jdbc:mysql://localhost:3306/ms_email?useSSL=false&serverTimezone=UTC
spring.datasource.username=root
spring.datasource.password=SUA_SENHA
spring.jpa.hibernate.ddl-auto=update
spring.rabbitmq.addresses=amqps://USUARIO:SENHA@HOST/VHOST
broker.queue.email.name=default.email

spring.mail.host=smtp.gmail.com
spring.mail.port=587
spring.mail.username=SEU_EMAIL@gmail.com
spring.mail.password=SUA_SENHA_DE_APP
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.starttls.enable=true
```

#### Modelo de E-mail

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `emailId` | UUID | Identificador único |
| `userId` | UUID | ID do usuário origem |
| `emailFrom` | String | Remetente |
| `emailTo` | String | Destinatário |
| `subject` | String | Assunto |
| `text` | String | Corpo |
| `sendDateEmail` | LocalDateTime | Data/hora do envio |
| `statusEmail` | Enum | `SENT` ou `ERROR` |

---

## Pré-requisitos

- Java 17+
- Maven 3.8+
- MySQL 8+
- Conta CloudAMQP (ou RabbitMQ local)
- Conta Gmail com **senha de app** ativada

## Como executar

### 1. Criar os bancos de dados

```sql
CREATE DATABASE ms_user;
CREATE DATABASE ms_email;
```

### 2. Ajustar as senhas nos `application.properties`

Substitua os placeholders `SUA_SENHA`, `SEU_EMAIL@gmail.com` e `SUA_SENHA_DE_APP` nos três serviços.

### 3. Iniciar cada serviço

```bash
# user-service
cd user-service && mvn spring-boot:run

# ms-user
cd ms-user && mvn spring-boot:run

# ms-email
cd ms-email && mvn spring-boot:run
```

### 4. Senha de app do Gmail

Acesse: **Conta Google → Segurança → Verificação em 2 etapas → Senhas de app**  
Gere uma senha de 16 caracteres e use no campo `spring.mail.password`.

---

## Autor

**Rafael Debroi** — CP3025861  
IFSP — Curso de Análise e Desenvolvimento de Sistemas  
Disciplina: Web 3
