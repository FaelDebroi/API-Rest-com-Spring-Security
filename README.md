# API Rest com Spring Security + Microsserviços com RabbitMQ

Projeto acadêmico — **Web 3 · IFSP · CP3025861**  
Etapa 1 entregue com tag `entrega1`.

---

## Estrutura do Repositório

```
├── user-service/   → Autenticação JWT + Spring Security (porta 8081)
├── ms-user/        → Cadastro de usuários com publicação no RabbitMQ (porta 8081)
└── ms-email/       → Consumidor RabbitMQ + envio de e-mail (porta 8082)
```

---

## Pré-requisitos

- Java 17+
- Maven 3.8+
- MySQL 8 (XAMPP ou standalone) na porta 3306
- Conta CloudAMQP (já configurada nos `application.properties`)
- Conta Gmail com **senha de app** (somente para ms-email)

### Criar os bancos

```sql
CREATE DATABASE ms_user;
CREATE DATABASE ms_email;
```

---

## Como executar

Abra um terminal para cada serviço. Todos os comandos partem da raiz do repositório.

### user-service (porta 8081)

```bash
cd user-service
mvn spring-boot:run
```

### ms-user (porta 8081)

> Pare o user-service antes — ambos usam a porta 8081.

```bash
cd ms-user
mvn spring-boot:run
```

### ms-email (porta 8082)

```bash
cd ms-email
mvn spring-boot:run
```

---

## user-service — Spring Security + JWT

Autenticação stateless com roles, senha BCrypt e token JWT.

### Tecnologias

- Spring Boot 3.2.4 · Spring Security 6 · jjwt 0.11.5
- Spring Data JPA · Spring AMQP · MySQL

### Entidades

| Classe | Descrição |
|--------|-----------|
| `User` | Email, senha (BCrypt), lista de roles |
| `Role` | Perfil de acesso |
| `RoleName` | Enum: `ROLE_ADMINISTRATOR`, `ROLE_CUSTOMER` |

### Endpoints

| Método | URL | Autenticação | Descrição |
|--------|-----|-------------|-----------|
| `POST` | `/users` | Público | Criar usuário com role |
| `POST` | `/users/login` | Público | Login — retorna token JWT |
| `GET` | `/users/test/customer` | ROLE_CUSTOMER | Rota protegida |
| `GET` | `/users/test/administrator` | ROLE_ADMINISTRATOR | Rota protegida |

### Exemplos

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

**Rota protegida:**
```
GET /users/test/customer
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
```
- Sem token → `403 Forbidden`
- Com token ROLE_CUSTOMER → `200 OK`

---

## ms-user + ms-email — Microsserviços com RabbitMQ

```
POST /users { name, email }
       │
   ms-user (8081)
   MySQL: ms_user · TB_USERS
       │
       └──► RabbitMQ: fila "default.email"
                   │
             ms-email (8082)
             MySQL: ms_email · TB_EMAILS
                   │
              Gmail SMTP
              Salva status: SENT | ERROR
```

### ms-user

**Endpoint:**

| Método | URL | Descrição |
|--------|-----|-----------|
| `POST` | `/users` | Salva usuário e publica mensagem na fila |

**Body:**
```json
{
  "name": "João Silva",
  "email": "joao@email.com"
}
```

### ms-email

Consome a fila `default.email`, envia e-mail via SMTP e persiste o registro.

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `emailId` | UUID | ID do registro |
| `userId` | UUID | ID do usuário origem |
| `emailFrom` | String | Remetente (configurado no SMTP) |
| `emailTo` | String | Destinatário |
| `subject` | String | Assunto |
| `text` | String | Corpo do e-mail |
| `sendDateEmail` | LocalDateTime | Data/hora do envio |
| `statusEmail` | Enum | `SENT` ou `ERROR` |

**Configurar Gmail (senha de app):**  
Conta Google → Segurança → Verificação em 2 etapas → Senhas de app.  
Use a senha gerada no campo `spring.mail.password` do `application.properties`.

---

## Entregas

| Tag | Conteúdo |
|-----|----------|
| `entrega1` | user-service JWT funcionando + estrutura base do ms-email |

---

## Autor

**Rafael Debroi** — CP3025861  
IFSP · Análise e Desenvolvimento de Sistemas · Web 3
