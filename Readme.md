# OIDC Authentication Microservice

A production-ready OIDC-compatible authentication microservice built with Node.js, Express, Prisma, and PostgreSQL.

## Features

- ✅ User Registration & Login with Argon2 password hashing
- ✅ JWT Token Issuance (ID Token & Access Token)
- ✅ JWKS Endpoint for public key discovery
- ✅ OAuth 2.0 Authorization Code Flow
- ✅ PKCE Support (S256 & plain)
- ✅ Refresh Tokens
- ✅ UserInfo Endpoint
- ✅ Security Headers (Helmet)
- ✅ CORS Support
- ✅ Structured Logging (Pino)

## Prerequisites

- Node.js 20+
- Docker & Docker Compose (for PostgreSQL)

## Setup

1. **Start PostgreSQL**:
   ```bash
   docker-compose up -d
   ```

2. **Install Dependencies**:
   ```bash
   npm install
   ```

3. **Run Migrations**:
   ```bash
   npx prisma migrate dev --name init
   ```

4. **Generate Prisma Client**:
   ```bash
   npx prisma generate
   ```

5. **Start Development Server**:
   ```bash
   npm run dev
   ```

The server will start on `http://localhost:3000`.

## API Endpoints

### Authentication
- `POST /auth/register` - Register a new user
- `POST /auth/login` - Login user

### OAuth 2.0 / OIDC
- `GET /oauth/authorize` - Authorization endpoint
- `POST /oauth/token` - Token endpoint
- `GET /oauth/userinfo` - UserInfo endpoint

### Discovery
- `GET /jwks` - JSON Web Key Set

### Health
- `GET /health` - Health check

## Example Flow

### 1. Register a User
```bash
curl -X POST http://localhost:3000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"password123"}'
```

### 2. Authorization Code Flow with PKCE

Generate code verifier and challenge:
```bash
# Code verifier (random string)
CODE_VERIFIER=$(openssl rand -base64 32 | tr -d "=+/" | cut -c1-43)

# Code challenge (SHA256 hash of verifier)
CODE_CHALLENGE=$(echo -n $CODE_VERIFIER | openssl dgst -sha256 -binary | base64 | tr -d "=+/" | tr "/+" "_-")
```

Get authorization code:
```bash
curl "http://localhost:3000/oauth/authorize?response_type=code&client_id=demo-client&redirect_uri=http://localhost:3000/callback&code_challenge=$CODE_CHALLENGE&code_challenge_method=S256"
```

Exchange code for tokens:
```bash
curl -X POST http://localhost:3000/oauth/token \
  -H "Content-Type: application/json" \
  -d "{\"grant_type\":\"authorization_code\",\"code\":\"YOUR_CODE\",\"redirect_uri\":\"http://localhost:3000/callback\",\"client_id\":\"demo-client\",\"code_verifier\":\"$CODE_VERIFIER\"}"
```

### 3. Access UserInfo
```bash
curl http://localhost:3000/oauth/userinfo \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

## Environment Variables

Create a `.env` file:
```
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/oidc_auth?schema=public"
PORT=3000
```

## Production Build

```bash
npm run build
npm start
```

## Tech Stack

- **Runtime**: Node.js 20+ with TypeScript
- **Framework**: Express.js
- **Database**: PostgreSQL
- **ORM**: Prisma 5
- **Auth**: jsonwebtoken, argon2
- **Validation**: Zod
- **Logging**: Pino
- **Security**: Helmet, CORS

## License

MIT
