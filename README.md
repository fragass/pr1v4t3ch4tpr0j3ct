# Workday CNX Chat

Aplicação web de chat interno com autenticação simples, mensagens em tempo quase real (polling), presença online e upload de imagens por **Ctrl+V**, usando front-end puro (HTML/CSS/JS) e funções serverless integradas ao Supabase.

---

## 1) O que este projeto entrega hoje

- Tela de login com validação por usuário/senha.
- Sessão no navegador via `sessionStorage`.
- Página principal de chat com:
  - feed de mensagens,
  - envio de texto,
  - envio por Enter,
  - menções com destaque (`@usuario`),
  - painel de emojis,
  - som para novas mensagens/menções,
  - lista de usuários online,
  - upload de imagem por colar (`Ctrl+V`).

---

## 2) Arquitetura (visão rápida)

### Front-end

- `index.html` + `css/loginf.css` + `js/loginf.js`
  - Tela de login.
- `m3yxe8u27wpoovbz.html`
  - Tela do chat com CSS e JS embutidos.

### Back-end (serverless)

Diretório `api/`:

- `api/login.js` → autenticação com base em `LOGIN_USERS`.
- `api/messages.js` → leitura/escrita de mensagens no Supabase REST.
- `api/online.js` → heartbeat de presença e listagem de usuários online.
- `api/upload.js` → upload multipart para Supabase Storage (`chat-images`).

### Infra / dependências

- Supabase (PostgREST + Storage)
- Dependências npm:
  - `@supabase/supabase-js`
  - `formidable`

---

## 3) Estrutura de arquivos

```text
.
├── api/
│   ├── login.js
│   ├── messages.js
│   ├── online.js
│   └── upload.js
├── css/
│   └── loginf.css
├── js/
│   └── loginf.js
├── favicon.png
├── index.html
├── m3yxe8u27wpoovbz.html
├── message.mp3
├── package.json
└── package-lock.json
```

> Observação: o nome `m3yxe8u27wpoovbz.html` parece ofuscado/temporário; pode ser renomeado no futuro para algo mais claro (ex.: `chat.html`).

---

## 4) Fluxos de funcionamento

## 4.1 Login

1. Usuário preenche `username` e `password`.
2. Front envia `POST /api/login` com JSON.
3. API valida credenciais na env var `LOGIN_USERS` (formato `user:pass,user2:pass2`).
4. Se sucesso:
   - retorna `token` randômico e `user`;
   - front salva `token` e `loggedUser` no `sessionStorage`;
   - redireciona para `m3yxe8u27wpoovbz.html`.

## 4.2 Proteção de acesso da página de chat

Ao abrir a página de chat, um script inicial verifica `sessionStorage`:

- sem `token` ou sem `loggedUser` → volta para `index.html`.

## 4.3 Mensagens

- **Leitura**: `GET /api/messages`
  - API consulta `messages` no Supabase ordenando por `created_at` asc.
- **Envio**: `POST /api/messages`
  - payload: `{ name, content, image_url }`.

No front:

- polling de mensagens a cada **3 segundos**;
- renderização do histórico no container `#messages`;
- quando há `image_url`, renderiza link clicável da imagem;
- quando há texto, aplica destaque de menções (`@usuario`).

## 4.4 Presença online

- Front envia heartbeat em `POST /api/online` a cada **5 segundos**.
- Front consulta `GET /api/online` também a cada **5 segundos**.
- API considera online quem tem `last_seen` com menos de ~15s.

## 4.5 Upload de imagem por Ctrl+V

1. Listener de `paste` no campo de mensagem detecta item de imagem.
2. Front chama `uploadImage(file)` → `POST /api/upload` (`multipart/form-data`).
3. API usa `formidable` para parse, envia ao bucket `chat-images` com `SUPABASE_SERVICE_ROLE_KEY`.
4. API retorna URL pública.
5. Front guarda em `pendingImageUrl` para enviar junto da próxima mensagem.

---

## 5) Endpoints da API

## `POST /api/login`

**Body**

```json
{
  "username": "usuario",
  "password": "senha"
}
```

**200 (sucesso)**

```json
{
  "success": true,
  "token": "...",
  "user": "usuario"
}
```

**401 (inválido)**

```json
{ "success": false }
```

---

## `GET /api/messages`

Retorna a lista de mensagens em ordem crescente de criação.

## `POST /api/messages`

**Body**

```json
{
  "name": "usuario",
  "content": "texto da mensagem",
  "image_url": "https://..." 
}
```

`image_url` é opcional.

---

## `GET /api/online`

Retorna array com usuários considerados online pela janela de atividade.

## `POST /api/online`

**Body**

```json
{ "name": "usuario" }
```

Faz upsert do `last_seen`.

---

## `POST /api/upload`

- Espera `multipart/form-data`.
- Campo de arquivo: `file`.
- Campo opcional: `fileName`.
- Retorna `{ url: "https://..." }` da imagem pública.

---

## 6) Variáveis de ambiente

Configure no ambiente (ex.: Vercel):

- `LOGIN_USERS`
  - Exemplo: `alice:123,bob:456`
- `SUPABASE_URL`
- `SUPABASE_ANON_KEY`
- `SUPABASE_SERVICE_ROLE_KEY`

---

## 7) Modelo de dados esperado (Supabase)

## Tabela `messages`

Campos esperados pelo código:

- `id`
- `name`
- `content`
- `image_url` (opcional)
- `created_at`

## Tabela `online_users`

Campos esperados:

- `name` (idealmente único)
- `last_seen`

## Storage bucket

- Bucket: `chat-images`
- Arquivos públicos para link nas mensagens.

---

## 8) Como executar

## Deploy recomendado (Vercel)

1. Conectar repositório.
2. Definir variáveis de ambiente.
3. Deploy.

## Execução local (ambiente serverless compatível)

```bash
npm install
vercel dev
```

---

## 9) Segurança e limitações atuais (importante)

- O token de login é gerado, mas não há validação robusta de sessão no servidor para cada endpoint.
- Credenciais em `LOGIN_USERS` funcionam para uso simples/interno, mas não substituem um sistema de auth completo.
- `SUPABASE_SERVICE_ROLE_KEY` no upload exige cuidado no ambiente de deploy.
- Sem suíte de testes automatizados no momento.

---

## 10) Melhorias sugeridas

- Migrar autenticação para JWT assinado e validado no backend.
- Proteger endpoints com middleware de autenticação/autorização.
- Adicionar rate limit e validação de payload.
- Criar testes para APIs (`login`, `messages`, `online`, `upload`).
- Renomear a página principal do chat para nome sem ofuscação.
