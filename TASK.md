# TASK.md — Temporary Email Service

## Context

Build a read-only temporary email web service similar to `generator.email`.

Chosen access model:

- **Option A — Public by address**
- Anyone who knows an email address can open the inbox.
- No password, no login, no ownership claim for ordinary inboxes.
- Email is **receive-only**. No outbound sending feature.

Preferred stack:

- **Node.js + TypeScript**
- **Next.js** for web UI
- **Fastify** for HTTP API
- **smtp-server** for SMTP ingress
- **mailparser** for MIME parsing
- **PostgreSQL** for persistent metadata/messages
- **Redis** for cache, pub/sub, rate limiting, and short-lived runtime state
- **BullMQ** for background jobs
- **Docker Compose** for VPS deployment
- **aaPanel Nginx** as existing reverse proxy / SSL manager

---

## 0. Product Requirements

### Core User Flow

1. User opens the web app.
2. User receives a generated email address.
3. User can generate a new email address.
4. User can create a custom local-part using one of the available system domains.
5. User can reopen an existing inbox by:
   - entering email + domain in the UI, or
   - opening a direct URL:
     - `https://mail.rains.com/inbox/name@domain.tld`
6. Inbox is read-only.
7. User can add their own custom domain by configuring DNS records:
   - TXT verification
   - MX record pointing to this server

### Explicit Non-Goals for v1

Do **not** implement these in v1:

- Sending email
- User accounts
- Password-protected inboxes
- Long-term mailbox ownership
- IMAP/POP3
- Full attachment preview
- Advanced spam filtering UI
- Billing
- Admin user management

---

## 1. Target Architecture

```txt
                         Internet
                            │
                            │ HTTPS
                            ▼
                    ┌─────────────────┐
                    │ aaPanel Nginx   │
                    │ SSL + proxy     │
                    └────────┬────────┘
                             │
          ┌──────────────────┴──────────────────┐
          │                                     │
          ▼                                     ▼
┌───────────────────┐                 ┌───────────────────┐
│ Next.js Web        │                 │ Fastify API        │
│ localhost:3000     │                 │ localhost:4000     │
└───────────────────┘                 └─────────┬─────────┘
                                                  │
                                     ┌────────────┴────────────┐
                                     │                         │
                                     ▼                         ▼
                              ┌─────────────┐           ┌─────────────┐
                              │ PostgreSQL  │           │ Redis       │
                              └─────────────┘           └─────────────┘


Incoming email flow:

External mail server
        │
        │ SMTP port 25
        ▼
┌────────────────────┐
│ SMTP Receiver       │
│ Node.js smtp-server │
│ host:25 -> app:2525 │
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│ BullMQ Queue        │
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│ Worker              │
│ MIME parsing        │
│ DB insert           │
│ Redis publish       │
└────────────────────┘
```

---

## 2. Monorepo Structure

Create this structure:

```txt
temp-mail/
  apps/
    web/
      # Next.js frontend
    api/
      # Fastify API server
    smtp/
      # SMTP receiver service
    worker/
      # BullMQ workers: email parsing, cleanup, DNS checks

  packages/
    config/
      # shared env parsing
    database/
      # Prisma or Drizzle schema/client
    mail-core/
      # address validation, domain utils, MIME helpers
    shared/
      # shared DTOs/types

  infra/
    nginx/
      # sample aaPanel/Nginx config snippets
    sql/
      # optional raw SQL migrations
    scripts/
      # setup and maintenance scripts

  docker-compose.yml
  docker-compose.prod.yml
  .env.example
  TASK.md
  README.md
```

---

## 3. Framework Decision

### Use Fastify for API

Use Fastify because the API is mostly:

- high-volume small JSON endpoints,
- strict request validation,
- low-latency inbox reads,
- domain verification endpoints,
- SSE event streams,
- stateless HTTP handlers.

Fastify is preferred over:

#### Express

Fastify advantages:

- better performance profile,
- better native TypeScript ergonomics,
- JSON schema validation support,
- cleaner plugin encapsulation,
- more suitable for a new TypeScript service.

Express is acceptable but not recommended for this project because it requires more manual setup for validation, error handling, typing, and serialization.

#### NestJS

NestJS advantages:

- strong enterprise structure,
- decorators,
- modules,
- dependency injection.

Fastify is preferred for this project because:

- less boilerplate,
- easier to reason about for a small/medium service,
- faster iteration,
- simpler deployment,
- no need for heavy application framework patterns yet.

If this project later becomes a larger team-maintained SaaS, NestJS with Fastify adapter can be reconsidered.

#### Next.js Route Handlers

Use Next.js only for the frontend and lightweight frontend-facing logic.

Do not make Next.js the main backend because this project needs:

- long-running SMTP receiver,
- queue workers,
- DNS verification jobs,
- Redis pub/sub,
- raw TCP port handling,
- background cleanup.

---

## 4. Core Services

### 4.1 `apps/web`

Responsibility:

- user-facing website,
- generated inbox page,
- custom inbox form,
- domain selector,
- existing inbox opener,
- custom-domain setup UI,
- read-only message viewer.

Tech:

- Next.js App Router
- TypeScript
- Tailwind CSS
- TanStack Query or SWR

Routes:

```txt
/
  Landing page; auto-generate inbox on first visit.

/inbox/[address]
  Public inbox page.
  Example:
  /inbox/test@example.com

/domains/add
  Custom-domain setup page.

/domains/[domain]/status
  Custom-domain DNS verification status page.
```

### 4.2 `apps/api`

Responsibility:

- inbox creation,
- inbox resolution,
- message listing,
- message detail,
- domain listing,
- custom-domain verification,
- SSE stream,
- rate limiting,
- validation.

Tech:

- Fastify
- TypeScript
- Zod or TypeBox
- Prisma or Drizzle
- Redis client
- BullMQ producer

Base URL:

```txt
http://127.0.0.1:4000
```

### 4.3 `apps/smtp`

Responsibility:

- listen for inbound SMTP,
- validate recipient domain,
- validate recipient local-part,
- reject unknown domains early,
- reject large messages,
- enqueue accepted email raw content into BullMQ.

Tech:

- `smtp-server`
- Redis/BullMQ
- shared database read access for domain validation

Internal listen port:

```txt
2525
```

Host port:

```txt
25 -> 2525
```

### 4.4 `apps/worker`

Responsibility:

- consume inbound email queue,
- parse raw MIME,
- upsert/resolve inbox,
- insert message rows,
- publish Redis event for SSE,
- cleanup expired messages,
- periodically verify pending custom domains.

Tech:

- BullMQ
- mailparser
- PostgreSQL
- Redis
- Node.js DNS resolver

---

## 5. Database Choice

Use **PostgreSQL**.

Use **Redis** for:

- rate limiting,
- BullMQ,
- pub/sub,
- short-lived generated inbox cache,
- SSE fanout,
- temporary DNS verification lock.

PostgreSQL remains source of truth.

---

## 6. Data Model

Use Prisma or Drizzle. Prisma is easier to start with; Drizzle is lighter. For v1, choose **Prisma** unless you strongly prefer SQL-first.

### 6.1 Tables

#### `domains`

```sql
CREATE TABLE domains (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  domain TEXT UNIQUE NOT NULL,
  type TEXT NOT NULL CHECK (type IN ('system', 'custom')),
  status TEXT NOT NULL CHECK (status IN ('pending', 'active', 'failed', 'disabled')),
  verification_token TEXT,
  verification_txt_name TEXT,
  mx_target TEXT,
  owner_fingerprint TEXT,
  last_checked_at TIMESTAMPTZ,
  verified_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Notes:

- `system` domains are yours.
- `custom` domains are user-provided.
- `owner_fingerprint` can be derived from user IP + browser local session ID for lightweight grouping, but no login is required.

#### `inboxes`

```sql
CREATE TABLE inboxes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  local_part TEXT NOT NULL,
  domain_id UUID NOT NULL REFERENCES domains(id),
  address TEXT UNIQUE NOT NULL,
  source TEXT NOT NULL CHECK (source IN ('generated', 'custom', 'url', 'smtp_auto')),
  expires_at TIMESTAMPTZ,
  last_accessed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inboxes_domain_id ON inboxes(domain_id);
CREATE INDEX idx_inboxes_expires_at ON inboxes(expires_at);
```

#### `messages`

```sql
CREATE TABLE messages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  inbox_id UUID NOT NULL REFERENCES inboxes(id) ON DELETE CASCADE,
  mail_from TEXT,
  rcpt_to TEXT NOT NULL,
  subject TEXT,
  from_header TEXT,
  to_header TEXT,
  text_body TEXT,
  html_body TEXT,
  raw_size INTEGER NOT NULL DEFAULT 0,
  spam_score NUMERIC,
  received_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at TIMESTAMPTZ
);

CREATE INDEX idx_messages_inbox_received
ON messages(inbox_id, received_at DESC);

CREATE INDEX idx_messages_deleted_at
ON messages(deleted_at);
```

#### `attachments`

```sql
CREATE TABLE attachments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  message_id UUID NOT NULL REFERENCES messages(id) ON DELETE CASCADE,
  filename TEXT,
  mime_type TEXT,
  size_bytes INTEGER NOT NULL,
  storage_key TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_attachments_message_id
ON attachments(message_id);
```

For v1, do not expose attachment download unless explicitly implemented with safety controls.

---

## 7. Address Policy

### 7.1 Local-Part Rules

Allowed:

```txt
a-z
0-9
dot .
dash -
underscore _
```

Regex:

```regex
^[a-z0-9][a-z0-9._-]{0,62}[a-z0-9]$
```

Additional rules:

- Convert local-part to lowercase.
- Reject consecutive dots.
- Reject leading/trailing dot/dash/underscore if desired.
- Max length: 64 chars.
- Min length: 3 chars for custom local-part.
- Generated local-part length: 8-12 chars.

### 7.2 Reserved Local-Parts

Reject:

```txt
admin
administrator
root
postmaster
abuse
hostmaster
webmaster
support
security
noreply
no-reply
mail
mx
smtp
imap
pop
api
www
```

### 7.3 Address Normalization

Implement a shared utility:

```ts
normalizeAddress(input: string): {
  localPart: string;
  domain: string;
  address: string;
}
```

Rules:

- trim whitespace,
- lowercase everything,
- validate exactly one `@`,
- validate domain using strict domain parser,
- reject invalid unicode/punycode until explicitly supported.

---

## 8. API Contract

### 8.1 List Public Domains

```http
GET /v1/domains/public
```

Response:

```json
{
  "domains": [
    {
      "domain": "rainsmail.com",
      "type": "system"
    }
  ]
}
```

### 8.2 Generate Inbox

```http
POST /v1/inboxes/generate
Content-Type: application/json
```

Body:

```json
{
  "domain": "rainsmail.com"
}
```

Response:

```json
{
  "address": "a9x4m2kd@rainsmail.com",
  "url": "https://mail.rains.com/inbox/a9x4m2kd@rainsmail.com",
  "expiresAt": "2026-04-27T00:00:00.000Z"
}
```

### 8.3 Create Custom Inbox

```http
POST /v1/inboxes/custom
Content-Type: application/json
```

Body:

```json
{
  "localPart": "testing",
  "domain": "rainsmail.com"
}
```

Behavior:

- if address does not exist, create it;
- if address exists, return it;
- never claim ownership;
- never expose private token because this model is public-by-address.

Response:

```json
{
  "address": "testing@rainsmail.com",
  "url": "https://mail.rains.com/inbox/testing@rainsmail.com"
}
```

### 8.4 Resolve Inbox

```http
GET /v1/inboxes/resolve?address=testing@rainsmail.com
```

Behavior:

- if inbox exists, return it;
- if inbox does not exist but domain is active and local-part valid, create inbox with `source = 'url'`;
- if domain is not active, return 404.

Response:

```json
{
  "address": "testing@rainsmail.com",
  "exists": true,
  "createdAt": "2026-04-26T12:00:00.000Z"
}
```

### 8.5 List Messages

```http
GET /v1/inboxes/:address/messages?limit=50&cursor=...
```

Response:

```json
{
  "messages": [
    {
      "id": "uuid",
      "from": "sender@example.com",
      "subject": "Verify your account",
      "receivedAt": "2026-04-26T12:01:00.000Z",
      "hasHtml": true,
      "hasText": true,
      "attachmentCount": 0
    }
  ],
  "nextCursor": null
}
```

### 8.6 Read Message

```http
GET /v1/messages/:id
```

Response:

```json
{
  "id": "uuid",
  "from": "sender@example.com",
  "to": "testing@rainsmail.com",
  "subject": "Verify your account",
  "textBody": "Your code is 123456",
  "htmlBody": "<p>Your code is <b>123456</b></p>",
  "receivedAt": "2026-04-26T12:01:00.000Z"
}
```

Security requirement:

- Sanitize HTML before rendering.
- Never render raw HTML directly in React without sanitization.
- Prefer iframe sandbox or sanitized HTML output.

### 8.7 SSE Inbox Events

```http
GET /v1/inboxes/:address/events
Accept: text/event-stream
```

Events:

```txt
event: message.created
data: {"messageId":"uuid","address":"testing@rainsmail.com"}
```

Implementation:

- API subscribes to Redis channel:
  - `inbox:{address}`
- Worker publishes message events after DB insert.

### 8.8 Add Custom Domain

```http
POST /v1/domains/custom
Content-Type: application/json
```

Body:

```json
{
  "domain": "exampleuser.com"
}
```

Response:

```json
{
  "domain": "exampleuser.com",
  "status": "pending",
  "records": {
    "txt": {
      "name": "_rainsmail-verify.exampleuser.com",
      "value": "rainsmail-verify-abc123"
    },
    "mx": {
      "name": "exampleuser.com",
      "value": "10 mx.mail.rains.com"
    }
  }
}
```

### 8.9 Verify Custom Domain

```http
POST /v1/domains/custom/:domain/verify
```

Behavior:

- check TXT record;
- check MX record;
- if valid, mark `active`;
- else return exact missing/incorrect records.

Response success:

```json
{
  "domain": "exampleuser.com",
  "status": "active"
}
```

Response pending:

```json
{
  "domain": "exampleuser.com",
  "status": "pending",
  "checks": {
    "txt": false,
    "mx": true
  }
}
```

---

## 9. SMTP Receiver Requirements

Use the `smtp-server` package.

### 9.1 Behavior

SMTP service must:

- listen on internal port `2525`;
- be exposed as host port `25`;
- reject unauthenticated outbound relay;
- accept only inbound mail for active domains;
- reject unknown domains during RCPT TO;
- reject invalid local-parts;
- reject oversized email before processing;
- enqueue raw email for worker parsing.

### 9.2 SMTP Acceptance Policy

Accept recipient if:

```txt
domain exists in domains table
AND domain.status = active
AND local_part is valid
AND message size <= MAX_EMAIL_BYTES
```

Reject recipient if:

```txt
domain is unknown
domain is disabled
local_part is reserved
local_part is invalid
```

### 9.3 Important SMTP Server Options

Set:

```ts
{
  authOptional: true,
  disabledCommands: ['AUTH'],
  size: Number(process.env.MAX_EMAIL_BYTES || 10485760),
  hideSTARTTLS: false
}
```

Do not allow relay. Do not implement `onAuth`.

### 9.4 Pseudocode

```ts
import { SMTPServer } from 'smtp-server';
import { simpleParser } from 'mailparser';

const server = new SMTPServer({
  authOptional: true,
  disabledCommands: ['AUTH'],
  size: 10 * 1024 * 1024,

  async onRcptTo(address, session, callback) {
    try {
      const normalized = normalizeAddress(address.address);
      const domain = await db.domain.findUnique({
        where: { domain: normalized.domain }
      });

      if (!domain || domain.status !== 'active') {
        return callback(new Error('550 Domain not accepted'));
      }

      if (!isAllowedLocalPart(normalized.localPart)) {
        return callback(new Error('550 Invalid recipient'));
      }

      session.envelope.rcptToNormalized = normalized;
      return callback();
    } catch (err) {
      return callback(new Error('550 Invalid recipient'));
    }
  },

  async onData(stream, session, callback) {
    try {
      const chunks: Buffer[] = [];
      let size = 0;

      stream.on('data', (chunk) => {
        size += chunk.length;
        if (size > MAX_EMAIL_BYTES) {
          stream.destroy(new Error('Message too large'));
          return;
        }
        chunks.push(chunk);
      });

      stream.on('end', async () => {
        const raw = Buffer.concat(chunks);
        await emailQueue.add('inbound-email', {
          raw: raw.toString('base64'),
          envelope: session.envelope
        });
        callback();
      });

      stream.on('error', (err) => {
        callback(err);
      });
    } catch (err) {
      callback(err as Error);
    }
  }
});

server.listen(2525, '0.0.0.0');
```

---

## 10. Worker Requirements

### 10.1 Email Parsing Worker

BullMQ job name:

```txt
inbound-email
```

Steps:

1. Decode raw base64.
2. Parse MIME with `mailparser`.
3. Normalize `rcpt_to`.
4. Find active domain.
5. Find or create inbox:
   - if existing: use it;
   - if absent: create with `source = 'smtp_auto'`.
6. Insert message.
7. Insert attachment metadata if enabled.
8. Publish Redis event:
   - channel: `inbox:{address}`
   - payload: message id and metadata.
9. Delete or archive raw email depending on config.

### 10.2 Cleanup Worker

Run every 5-15 minutes.

Rules:

- delete generated inboxes past `expires_at`;
- delete messages older than retention policy;
- delete orphan messages;
- delete attachment objects if attachment storage is enabled.

Default retention:

```env
GENERATED_INBOX_TTL_MINUTES=1440
MESSAGE_RETENTION_HOURS=24
CUSTOM_INBOX_RETENTION_HOURS=72
```

### 10.3 DNS Verification Worker

Run every 10 minutes.

For pending custom domains:

- check TXT verification,
- check MX target,
- update `status`,
- update `last_checked_at`.

---

## 11. Frontend Requirements

### 11.1 Home Page

On first visit:

1. Fetch public domains.
2. Generate inbox with default domain.
3. Redirect to `/inbox/{address}` or display inbox inline.

Controls:

- Copy email button
- Generate new button
- Custom local-part input
- Domain dropdown
- Open existing inbox input

### 11.2 Inbox Page

Display:

- current email address,
- copy button,
- message list,
- empty state,
- auto-update indicator,
- refresh button,
- message detail panel.

Behavior:

- on load, resolve inbox via API;
- open SSE connection;
- refetch messages on `message.created`;
- allow direct URL access.

### 11.3 Domain Add Page

Display:

- domain input,
- generated TXT record,
- generated MX record,
- verification status,
- verify button,
- troubleshooting notes.

---

## 12. Security Requirements

### 12.1 Public-by-Address Warning

Because inboxes are public by address, show a clear notice:

```txt
Anyone who knows this email address can view this inbox. Do not use it for sensitive accounts.
```

### 12.2 HTML Sanitization

Never render raw email HTML unsanitized.

Options:

- sanitize with DOMPurify on frontend,
- sanitize on backend,
- render in sandboxed iframe,
- strip scripts, inline event handlers, external JS.

Minimum:

```html
<iframe sandbox="" srcdoc="sanitizedHtml"></iframe>
```

Avoid allowing:

```txt
script
iframe
object
embed
form
on* attributes
javascript: URLs
```

### 12.3 Rate Limiting

Add rate limits:

```txt
Generate inbox:
  20 per IP per 10 minutes

Custom inbox:
  20 per IP per 10 minutes

Resolve inbox:
  120 per IP per minute

List messages:
  120 per IP per minute

Read message:
  120 per IP per minute

Custom domain add:
  5 per IP per hour

SMTP RCPT:
  configurable per remote IP

SMTP DATA:
  configurable per remote IP
```

### 12.4 Abuse Controls

Implement:

- max email size,
- max recipients per message,
- max messages per inbox per hour,
- block reserved local-parts,
- block unknown domains,
- block disabled domains,
- log rejected SMTP attempts,
- optional IP blocklist.

### 12.5 Attachment Policy

For v1:

```txt
Do not expose attachments publicly.
```

Still parse attachment metadata if needed, but do not provide download links.

Later:

- scan attachments,
- limit MIME types,
- store in S3-compatible storage,
- signed URLs,
- strict expiration.

---

## 13. Environment Variables

Create `.env.example`:

```env
# App
NODE_ENV=production
PUBLIC_WEB_URL=https://mail.rains.com
API_URL=http://api:4000

# API
API_HOST=0.0.0.0
API_PORT=4000
CORS_ORIGIN=https://mail.rains.com

# Web
NEXT_PUBLIC_API_BASE_URL=https://mail.rains.com/api

# SMTP
SMTP_HOST=0.0.0.0
SMTP_PORT=2525
SMTP_PUBLIC_HOSTNAME=mx.mail.rains.com
MAX_EMAIL_BYTES=10485760
MAX_RECIPIENTS_PER_MESSAGE=5

# Database
DATABASE_URL=postgresql://tempmail:change-me@postgres:5432/tempmail

# Redis
REDIS_URL=redis://redis:6379

# Retention
GENERATED_INBOX_TTL_MINUTES=1440
MESSAGE_RETENTION_HOURS=24
CUSTOM_INBOX_RETENTION_HOURS=72

# Rate Limit
RATE_LIMIT_REDIS_PREFIX=rl:tempmail

# DNS Verification
CUSTOM_DOMAIN_TXT_PREFIX=_rainsmail-verify
CUSTOM_DOMAIN_MX_TARGET=mx.mail.rains.com
```

---

## 14. Docker Setup

### 14.1 `docker-compose.yml`

Create:

```yaml
services:
  web:
    build:
      context: .
      dockerfile: apps/web/Dockerfile
    restart: unless-stopped
    env_file:
      - .env
    depends_on:
      - api
    ports:
      - "127.0.0.1:3000:3000"

  api:
    build:
      context: .
      dockerfile: apps/api/Dockerfile
    restart: unless-stopped
    env_file:
      - .env
    depends_on:
      - postgres
      - redis
    ports:
      - "127.0.0.1:4000:4000"

  smtp:
    build:
      context: .
      dockerfile: apps/smtp/Dockerfile
    restart: unless-stopped
    env_file:
      - .env
    depends_on:
      - postgres
      - redis
    ports:
      - "25:2525"
      - "2525:2525"

  worker:
    build:
      context: .
      dockerfile: apps/worker/Dockerfile
    restart: unless-stopped
    env_file:
      - .env
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: tempmail
      POSTGRES_USER: tempmail
      POSTGRES_PASSWORD: change-me
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "127.0.0.1:5432:5432"

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: ["redis-server", "--appendonly", "yes"]
    volumes:
      - redis_data:/data
    ports:
      - "127.0.0.1:6379:6379"

volumes:
  postgres_data:
  redis_data:
```

Notes:

- PostgreSQL and Redis are bound to `127.0.0.1`.
- Web/API are bound to localhost and proxied by aaPanel Nginx.
- SMTP must bind to public port `25`.

### 14.2 Production Override

Create `docker-compose.prod.yml` if needed:

```yaml
services:
  web:
    environment:
      NODE_ENV: production

  api:
    environment:
      NODE_ENV: production

  smtp:
    environment:
      NODE_ENV: production

  worker:
    environment:
      NODE_ENV: production
```

Run:

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build
```

---

## 15. VPS Ubuntu Setup

### 15.1 Update Server

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y ca-certificates curl gnupg lsb-release ufw git
```

### 15.2 Install Docker Engine

Use Docker's official apt repository method.

```bash
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update

sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo systemctl enable docker
sudo systemctl start docker

docker --version
docker compose version
```

Optional non-root Docker access:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

### 15.3 Firewall

Open:

```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 25/tcp
sudo ufw allow 2525/tcp
sudo ufw enable
sudo ufw status
```

If you expose only port 25 for SMTP, you may skip public `2525`. Keeping `2525` open is useful for testing, but not mandatory.

Recommended production rule:

```bash
sudo ufw delete allow 2525/tcp
```

### 15.4 Check Port 25 Availability

Run:

```bash
sudo ss -tulpn | grep ':25'
```

If another mail service is already using port 25, stop it.

Common services:

```bash
sudo systemctl stop postfix
sudo systemctl disable postfix
```

Only do this if Postfix exists and is not needed.

### 15.5 Create Project Directory

```bash
sudo mkdir -p /opt/temp-mail
sudo chown -R $USER:$USER /opt/temp-mail
cd /opt/temp-mail
```

Clone project:

```bash
git clone <YOUR_REPO_URL> .
```

Create `.env`:

```bash
cp .env.example .env
nano .env
```

Set secure values:

```env
POSTGRES_PASSWORD=use-a-strong-password
DATABASE_URL=postgresql://tempmail:use-a-strong-password@postgres:5432/tempmail
PUBLIC_WEB_URL=https://mail.rains.com
NEXT_PUBLIC_API_BASE_URL=https://mail.rains.com/api
CUSTOM_DOMAIN_MX_TARGET=mx.mail.rains.com
SMTP_PUBLIC_HOSTNAME=mx.mail.rains.com
```

### 15.6 Start Services

```bash
docker compose up -d --build
docker compose ps
docker compose logs -f api
```

Run migrations:

```bash
docker compose exec api npm run db:migrate
```

Seed system domains:

```bash
docker compose exec api npm run db:seed
```

---

## 16. aaPanel + Nginx Setup

Assumption:

- aaPanel is already installed.
- aaPanel Nginx is already managing website SSL.
- Docker services are bound locally:
  - Web: `127.0.0.1:3000`
  - API: `127.0.0.1:4000`

### 16.1 Create Website in aaPanel

In aaPanel:

1. Go to **Website**.
2. Add site:
   - Domain: `mail.rains.com`
   - Root path can be any placeholder path, e.g. `/www/wwwroot/mail.rains.com`
3. Enable SSL.
4. Use Let's Encrypt from aaPanel.
5. Open Nginx config for the site.

### 16.2 Nginx Reverse Proxy Config

Use this structure inside the server block for `mail.rains.com`.

```nginx
# API proxy
location /api/ {
    proxy_pass http://127.0.0.1:4000/;
    proxy_http_version 1.1;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    proxy_buffering off;
    proxy_read_timeout 3600;
    proxy_send_timeout 3600;
}

# SSE endpoint
location /api/v1/inboxes/ {
    proxy_pass http://127.0.0.1:4000/v1/inboxes/;
    proxy_http_version 1.1;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    proxy_buffering off;
    proxy_cache off;
    proxy_read_timeout 3600;
    proxy_send_timeout 3600;
}

# Next.js frontend
location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_http_version 1.1;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";

    proxy_read_timeout 3600;
    proxy_send_timeout 3600;
}
```

Important:

- The `/api/` path strips `/api/` because `proxy_pass` ends with `/`.
- API service should expose routes as `/v1/...`.
- Browser calls `https://mail.rains.com/api/v1/...`.

If aaPanel inserts config into a special include section, put this inside the custom Nginx config area.

Then test:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

If aaPanel uses its own Nginx binary path:

```bash
/www/server/nginx/sbin/nginx -t
/etc/init.d/nginx reload
```

Use whichever is valid on your aaPanel install.

### 16.3 Avoid Conflict With Docker Port 80/443

Do not expose Docker web/API directly to `0.0.0.0:80` or `0.0.0.0:443`.

Keep:

```yaml
ports:
  - "127.0.0.1:3000:3000"
  - "127.0.0.1:4000:4000"
```

aaPanel Nginx remains the only public HTTP/HTTPS entrypoint.

---

## 17. DNS Setup

### 17.1 Main Web Domain

For web:

```txt
mail.rains.com A <VPS_PUBLIC_IP>
```

### 17.2 MX Hostname

For SMTP receiver:

```txt
mx.mail.rains.com A <VPS_PUBLIC_IP>
```

### 17.3 System Mail Domains

For every system domain users can select:

```txt
rainsmail.com MX 10 mx.mail.rains.com
```

Optional:

```txt
rainsmail.com TXT "v=spf1 -all"
_dmarc.rainsmail.com TXT "v=DMARC1; p=reject; rua=mailto:abuse@rainsmail.com"
```

Because the service is receive-only, SPF can be restrictive.

### 17.4 Custom User Domain Instructions

For user domain `exampleuser.com`, show:

```txt
TXT:
_rainsmail-verify.exampleuser.com
rainsmail-verify-<token>

MX:
exampleuser.com
10 mx.mail.rains.com
```

Verification rules:

- TXT token must match exactly.
- At least one MX record must point to `mx.mail.rains.com`.
- Mark active only when both are valid.

---

## 18. Development Setup

### 18.1 Local Requirements

Install:

```bash
node --version
npm --version
docker --version
docker compose version
```

Use Node.js LTS.

### 18.2 Install Dependencies

Use pnpm or npm workspaces. Recommended: pnpm.

```bash
corepack enable
pnpm install
```

### 18.3 Local Docker

Run dependencies only:

```bash
docker compose up -d postgres redis
```

Run apps locally:

```bash
pnpm dev
```

Or run all via Docker:

```bash
docker compose up -d --build
```

### 18.4 Test SMTP Locally

Install swaks:

```bash
sudo apt install -y swaks
```

Send test email:

```bash
swaks \
  --server 127.0.0.1:2525 \
  --to test@rainsmail.com \
  --from sender@example.com \
  --header "Subject: Test email" \
  --body "Hello from swaks"
```

Then open:

```txt
https://mail.rains.com/inbox/test@rainsmail.com
```

For local frontend:

```txt
http://localhost:3000/inbox/test@rainsmail.com
```

---

## 19. Implementation Tasks for Codex

Execute tasks in order.

---

### Task 1 — Initialize Monorepo

Create a TypeScript monorepo.

Requirements:

- package manager: pnpm
- workspace apps:
  - `apps/web`
  - `apps/api`
  - `apps/smtp`
  - `apps/worker`
- workspace packages:
  - `packages/config`
  - `packages/database`
  - `packages/mail-core`
  - `packages/shared`

Root scripts:

```json
{
  "scripts": {
    "dev": "pnpm -r --parallel dev",
    "build": "pnpm -r build",
    "lint": "pnpm -r lint",
    "typecheck": "pnpm -r typecheck",
    "test": "pnpm -r test"
  }
}
```

Acceptance criteria:

- `pnpm install` works.
- `pnpm build` runs without missing package errors.
- Each app has a minimal `dev` script.

---

### Task 2 — Shared Config Package

Implement `packages/config`.

Requirements:

- parse environment variables;
- validate using Zod;
- export config objects for:
  - API
  - Web
  - SMTP
  - Worker
  - Database
  - Redis
  - Retention
  - Domain verification

Acceptance criteria:

- invalid env fails fast with useful error.
- no app reads `process.env` directly except config package.

---

### Task 3 — Database Package

Implement `packages/database`.

Use Prisma.

Requirements:

- define schema for:
  - Domain
  - Inbox
  - Message
  - Attachment
- expose Prisma client singleton;
- create migration;
- create seed script for system domains.

Seed script should read system domains from env or constant:

```txt
rainsmail.com
rainsbox.net
```

Acceptance criteria:

- `pnpm db:migrate` works.
- `pnpm db:seed` inserts system domains as active.
- unique constraints prevent duplicate domains and addresses.

---

### Task 4 — Mail Core Package

Implement `packages/mail-core`.

Functions:

```ts
normalizeAddress(input: string): NormalizedAddress
isAllowedLocalPart(localPart: string): boolean
isReservedLocalPart(localPart: string): boolean
generateRandomLocalPart(length?: number): string
normalizeDomain(input: string): string
buildInboxUrl(publicWebUrl: string, address: string): string
sanitizeEmailHtml(html: string): string
```

Types:

```ts
type NormalizedAddress = {
  localPart: string;
  domain: string;
  address: string;
};
```

Acceptance criteria:

- rejects invalid addresses,
- rejects reserved local-parts,
- handles uppercase input by normalizing lowercase,
- has unit tests for edge cases.

---

### Task 5 — Fastify API Skeleton

Implement `apps/api`.

Requirements:

- Fastify server;
- health route:
  - `GET /health`
- versioned API routes:
  - `/v1/...`
- structured error responses;
- CORS configured from env;
- request logging;
- graceful shutdown.

Acceptance criteria:

```bash
curl http://127.0.0.1:4000/health
```

returns:

```json
{
  "ok": true
}
```

---

### Task 6 — Domain API

Implement:

```txt
GET  /v1/domains/public
POST /v1/domains/custom
POST /v1/domains/custom/:domain/verify
GET  /v1/domains/custom/:domain/status
```

Requirements:

- list active domains;
- create pending custom domain;
- generate TXT verification token;
- return required DNS records;
- verify TXT and MX using Node.js DNS resolver;
- mark domain active only when TXT and MX are valid.

Acceptance criteria:

- system domains appear in `/v1/domains/public`.
- custom domain returns clear DNS setup instructions.
- verification returns partial status when only one record exists.

---

### Task 7 — Inbox API

Implement:

```txt
POST /v1/inboxes/generate
POST /v1/inboxes/custom
GET  /v1/inboxes/resolve
GET  /v1/inboxes/:address/messages
```

Requirements:

- generate random inbox;
- create custom local-part inbox;
- resolve existing inbox by address;
- auto-create inbox from direct URL if domain active;
- list messages sorted by newest first.

Acceptance criteria:

- generated inbox returns usable URL.
- custom local-part creates deterministic address.
- resolving unknown active-domain address creates inbox with source `url`.
- unknown domain returns 404.

---

### Task 8 — Message API

Implement:

```txt
GET /v1/messages/:id
```

Requirements:

- return message body;
- include text and sanitized HTML;
- do not expose raw headers unless needed;
- do not expose attachments in v1.

Acceptance criteria:

- message detail is readable by ID.
- HTML output is sanitized.

---

### Task 9 — Redis Rate Limiting

Add rate limiting to API.

Rules:

```txt
Generate inbox:
  20 per IP per 10 minutes

Custom inbox:
  20 per IP per 10 minutes

Resolve inbox:
  120 per IP per minute

List messages:
  120 per IP per minute

Read message:
  120 per IP per minute

Custom domain add:
  5 per IP per hour
```

Acceptance criteria:

- repeated requests eventually return HTTP 429.
- rate limit keys are stored in Redis.
- response includes clear error message.

---

### Task 10 — SSE Inbox Events

Implement:

```txt
GET /v1/inboxes/:address/events
```

Requirements:

- use `text/event-stream`;
- subscribe to Redis channel `inbox:{address}`;
- send keepalive comments every 20-30 seconds;
- close cleanly on client disconnect.

Acceptance criteria:

- browser receives event when worker publishes new message.
- no memory leak when client disconnects.

---

### Task 11 — SMTP Receiver

Implement `apps/smtp`.

Requirements:

- start SMTP server on `SMTP_PORT`;
- validate recipient domain during RCPT TO;
- validate local-part;
- reject unknown/disabled domains;
- reject oversized messages;
- enqueue accepted raw email to BullMQ queue `inbound-email`.

Acceptance criteria:

- `swaks` can send email to active domain.
- `swaks` to unknown domain is rejected.
- logs show accepted/rejected recipients.
- job appears in BullMQ/Redis.

---

### Task 12 — Email Worker

Implement `apps/worker`.

Requirements:

- consume `inbound-email` jobs;
- parse raw email using `mailparser`;
- find or create inbox;
- save message in PostgreSQL;
- publish Redis event;
- handle parser errors gracefully;
- implement retry policy.

Acceptance criteria:

- sent test email appears in inbox API.
- SSE receives new message event.
- parser errors are logged and do not crash worker.

---

### Task 13 — Cleanup Worker

Add scheduled cleanup.

Requirements:

- delete expired generated inboxes;
- soft-delete or hard-delete old messages based on env;
- delete attachment metadata for deleted messages;
- log cleanup counts.

Acceptance criteria:

- expired records are removed on schedule.
- cleanup can be run manually via script.

---

### Task 14 — Next.js Web App

Implement `apps/web`.

Pages:

```txt
/
 /inbox/[address]
 /domains/add
 /domains/[domain]/status
```

Components:

```txt
EmailAddressCard
DomainSelector
GenerateButton
CustomInboxForm
OpenInboxForm
InboxMessageList
MessageViewer
CopyButton
PublicInboxWarning
DomainDnsInstructions
```

Requirements:

- home page generates inbox on first visit;
- inbox page can open direct address URL;
- copy address button;
- generate new address button;
- custom local-part creation;
- open existing inbox form;
- SSE updates;
- message detail view;
- public-by-address warning.

Acceptance criteria:

- user can generate and use inbox end-to-end.
- user can open `/inbox/name@domain.tld`.
- incoming email appears without full page reload.

---

### Task 15 — HTML Email Rendering

Implement safe email rendering.

Requirements:

- sanitize HTML;
- strip scripts and unsafe attributes;
- render inside sandboxed iframe or safe container;
- fallback to text body if no HTML.

Acceptance criteria:

- `<script>` from email body never executes.
- `javascript:` links are removed or neutralized.
- text-only email renders correctly.

---

### Task 16 — Dockerfiles

Create Dockerfiles for all apps.

Minimum:

```txt
apps/web/Dockerfile
apps/api/Dockerfile
apps/smtp/Dockerfile
apps/worker/Dockerfile
```

Requirements:

- multi-stage builds;
- production dependencies only in final image;
- non-root user if practical;
- expose correct ports.

Acceptance criteria:

- `docker compose build` succeeds.
- `docker compose up -d` starts all services.

---

### Task 17 — Compose Deployment

Implement root `docker-compose.yml`.

Requirements:

- web on `127.0.0.1:3000`;
- api on `127.0.0.1:4000`;
- smtp public `25:2525`;
- postgres local only;
- redis local only;
- persistent volumes.

Acceptance criteria:

```bash
docker compose ps
```

shows all services healthy/running.

---

### Task 18 — aaPanel Nginx Integration

Add `infra/nginx/aapanel-mail.rains.com.conf`.

Include:

- `/api/` proxy to `127.0.0.1:4000`;
- `/` proxy to `127.0.0.1:3000`;
- SSE-friendly buffering disabled;
- forwarded headers;
- long read timeout.

Acceptance criteria:

- `https://mail.rains.com` loads Next.js.
- `https://mail.rains.com/api/health` reaches API.
- SSE stream stays open.

---

### Task 19 — DNS Documentation Page

Create internal docs page or markdown:

```txt
docs/dns-setup.md
```

Include:

- web A record;
- MX hostname A record;
- system domain MX record;
- custom domain TXT + MX instructions;
- DNS troubleshooting commands.

Commands:

```bash
dig A mail.rains.com
dig A mx.mail.rains.com
dig MX rainsmail.com
dig TXT _rainsmail-verify.exampleuser.com
dig MX exampleuser.com
```

Acceptance criteria:

- user can follow docs to configure a domain.

---

### Task 20 — Operational Scripts

Add scripts:

```txt
infra/scripts/deploy.sh
infra/scripts/logs.sh
infra/scripts/backup-postgres.sh
infra/scripts/restore-postgres.sh
infra/scripts/check-dns.sh
```

Requirements:

`deploy.sh`:

```bash
git pull
docker compose build
docker compose up -d
docker compose exec api npm run db:migrate
docker compose ps
```

`backup-postgres.sh`:

```bash
docker compose exec postgres pg_dump -U tempmail tempmail > backups/tempmail-$(date +%F-%H%M%S).sql
```

Acceptance criteria:

- deploy can be executed with one command.
- backup file is created successfully.

---

## 20. Manual VPS Deployment Checklist

Follow this checklist after code is ready.

### 20.1 Server

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y ca-certificates curl gnupg lsb-release ufw git swaks dnsutils
```

### 20.2 Docker

Install Docker Engine and Compose plugin.

Verify:

```bash
docker --version
docker compose version
docker run hello-world
```

### 20.3 Firewall

```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 25/tcp
sudo ufw enable
```

### 20.4 DNS

Create records:

```txt
mail.rains.com A <VPS_IP>
mx.mail.rains.com A <VPS_IP>
rainsmail.com MX 10 mx.mail.rains.com
```

Check:

```bash
dig A mail.rains.com
dig A mx.mail.rains.com
dig MX rainsmail.com
```

### 20.5 aaPanel

In aaPanel:

1. Add website `mail.rains.com`.
2. Enable SSL.
3. Add Nginx reverse proxy config.
4. Test config.
5. Reload Nginx.

### 20.6 Deploy App

```bash
cd /opt/temp-mail
git pull
cp .env.example .env
nano .env
docker compose up -d --build
docker compose exec api npm run db:migrate
docker compose exec api npm run db:seed
```

### 20.7 Test HTTP

```bash
curl https://mail.rains.com/api/health
```

Expected:

```json
{
  "ok": true
}
```

### 20.8 Test SMTP

```bash
swaks \
  --server mx.mail.rains.com \
  --to test@rainsmail.com \
  --from sender@example.com \
  --header "Subject: VPS SMTP Test" \
  --body "Hello from production SMTP test"
```

Open:

```txt
https://mail.rains.com/inbox/test@rainsmail.com
```

---

## 21. Monitoring and Logs

Useful commands:

```bash
docker compose ps
docker compose logs -f api
docker compose logs -f smtp
docker compose logs -f worker
docker compose logs -f web
```

Check ports:

```bash
sudo ss -tulpn | grep -E ':25|:80|:443|:3000|:4000'
```

Check Redis:

```bash
docker compose exec redis redis-cli ping
```

Check PostgreSQL:

```bash
docker compose exec postgres psql -U tempmail -d tempmail -c '\dt'
```

---

## 22. Production Hardening

After MVP works, implement:

- admin dashboard for domains/messages statistics,
- remote IP reputation checks,
- DNSBL checks for SMTP sender IP,
- better mail header inspection,
- optional ClamAV attachment scanning,
- per-domain message limits,
- domain abuse disable switch,
- message body size cap,
- Prometheus metrics,
- Sentry error reporting,
- Postgres automated backups,
- Redis memory limit,
- log rotation,
- fail2ban for abusive SMTP connections.

---

## 23. Important Edge Cases

Handle these:

- email arrives before user opens inbox;
- user opens inbox before email arrives;
- direct URL creates inbox lazily;
- multiple RCPT TO in one SMTP transaction;
- unknown domain recipient;
- uppercase address input;
- invalid `@` address format;
- custom domain MX exists but TXT missing;
- TXT exists but MX target wrong;
- domain disabled after messages exist;
- message with only HTML body;
- message with only text body;
- malformed MIME message;
- very large email;
- duplicate message delivery.

---

## 24. Acceptance Criteria for MVP

The MVP is complete when:

1. User can open `https://mail.rains.com`.
2. User automatically gets generated email.
3. User can generate a new email.
4. User can create `custom@selected-domain`.
5. User can open `https://mail.rains.com/inbox/custom@selected-domain`.
6. External email sent to that address is accepted by SMTP receiver.
7. Email appears in the web inbox.
8. Inbox updates without full page reload.
9. Unknown domains are rejected at SMTP RCPT stage.
10. User can add custom domain and verify TXT + MX.
11. Custom domain becomes selectable after verification.
12. All services run under Docker Compose.
13. aaPanel Nginx serves HTTPS and proxies to web/API.
14. PostgreSQL and Redis persist data via Docker volumes.
15. Cleanup worker removes expired data.

---

## 25. Recommended Execution Order

Use this exact order:

```txt
1. Monorepo setup
2. Config package
3. Database package + migrations
4. Mail core package
5. Fastify API skeleton
6. Domain API
7. Inbox API
8. Message API
9. Redis rate limiting
10. SSE events
11. SMTP receiver
12. Email worker
13. Cleanup worker
14. Next.js frontend
15. Safe HTML rendering
16. Dockerfiles
17. Docker Compose
18. aaPanel Nginx config
19. DNS docs
20. Deployment scripts
21. VPS deployment
22. End-to-end test
```

---

## 26. Notes for Codex

When implementing:

- Keep each service small and focused.
- Prefer explicit validation over permissive parsing.
- Put shared logic in `packages/mail-core`.
- Do not duplicate address parsing logic across apps.
- Do not implement outbound email.
- Do not implement authentication unless explicitly requested later.
- Keep inbox access public by address.
- Reject unknown SMTP domains early.
- Prioritize working end-to-end flow before UI polish.
- Add tests for address normalization and domain verification logic first.
