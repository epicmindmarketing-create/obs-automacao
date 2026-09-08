# OBS Cloud — SaaS de Automação para Transmissões

Automação de OBS Studio (decupagem ao vivo por palestrante + upload automático no Google Drive)
com backend SaaS: contas, **assinatura mensal recorrente** via Mercado Pago, DRM/anti-pirataria
e túnel público automático para rodar 24/7 mesmo sem domínio.

## Componentes

| Pasta | Papel | Porta |
|---|---|---|
| **`payment-server/`** | Backend SaaS: auth (JWT), assinatura recorrente MP, licenças/HWID, painel admin, túnel público | `3002` |
| **`obs-cloud-client/`** | App desktop Electron (login, automação OBS, upload Drive, trava de licença) | — |
| `obs-cloud-server/` | **Legado** — não é mais usado. Ignore. | ~~3000~~ |

## Arquitetura

```
EpicMind-Broadcast/
├── payment-server/
│   ├── server.js               # orquestrador: migrations → túnel → rotas → listen
│   ├── ecosystem.config.js     # PM2 (execução 24/7)
│   ├── database.sqlite         # users, licenses, transactions, promotions, settings, logs
│   ├── src/
│   │   ├── config.js           # env + PUBLIC_URL dinâmico
│   │   ├── db.js               # sqlite3 + helpers async
│   │   ├── migrations.js       # cria/atualiza schema no boot
│   │   ├── tunnel.js           # localtunnel → injeta PUBLIC_URL
│   │   ├── auth.js             # JWT, bcrypt, assinatura de HWID (HMAC)
│   │   ├── licenseService.js   # trial / assinatura / vínculo de máquina
│   │   ├── mercadopago.js      # /preapproval (assinatura), pagamentos
│   │   ├── google.js           # OAuth server-side (opcional)
│   │   ├── views.js            # páginas dark-mode (sucesso, termos, etc.)
│   │   └── routes/             # auth, license, subscription, webhook, google, admin, pages
│   └── public/
│       ├── index.html          # painel administrativo
│       └── checkout.html       # assinatura via navegador
│
└── obs-cloud-client/
    ├── main.js                 # janelas (login → app), IPC, OBS, upload
    ├── preload.js / login-preload.js / modal-preload.js
    ├── services/
    │   ├── machineId.js        # HWID ESTÁVEL (id local persistido + hostname/user → sha256)
    │   ├── apiClient.js        # HTTP p/ payment-server + descoberta da URL pública
    │   ├── sessionManager.js   # JWT + heartbeat + trava OFFLINE (DRM)
    │   └── googleAuth.js       # OAuth Google via LOOPBACK (sem copiar/colar)
    └── src/
        ├── login.html / login-renderer.js     # tela de login (email/senha + Google)
        ├── index.html / renderer.js           # app + overlay de bloqueio
        └── admin-panel.html / admin-renderer.js
```

## Requisitos

- Node.js 18+
- OBS Studio 28+ (WebSocket Server habilitado)
- Conta Mercado Pago (token de acesso)
- Projeto Google Cloud com OAuth (para o Google Drive / login Google) — o `credentials.json`
  do cliente precisa ter `http://localhost:3000` nos *redirect URIs* (já vem assim).

## Instalação

### 1. Servidor

```bash
cd payment-server
npm install
cp .env.example .env      # edite os segredos
npm start
```

`.env` mínimo:
```env
PORT=3002
MERCADO_PAGO_ACCESS_TOKEN=APP_USR-xxxx   # APP_USR = PRODUÇÃO (cobranças reais); TEST- = sandbox
MERCADO_PAGO_PUBLIC_KEY=APP_USR-xxxx
MERCADO_PAGO_CLIENT_ID=...
MERCADO_PAGO_CLIENT_SECRET=...
ENABLE_TUNNEL=true                       # sobe localtunnel automaticamente
TUNNEL_SUBDOMAIN=epicmind-obs-cloud      # subdomínio FIXO → URL estável entre restarts
JWT_SECRET=um-segredo-longo-e-fixo       # se preenchido, NUNCA é sobrescrito (sessões sobrevivem ao restart)
HWID_SECRET=outro-segredo-longo-e-fixo
ADMIN_EMAIL=voce@exemplo.com             # essa conta vira admin do painel no boot
PUBLIC_URL=                              # opcional: domínio https próprio (ignora o túnel)
```

> **Produção (APP_USR)**: os pagamentos são **reais**. Configure a `notification_url` do
> webhook no painel do Mercado Pago apontando para `https://<TUNNEL_SUBDOMAIN>.loca.lt/api/v1/webhook/mercadopago`.
> Você **não consegue assinar com o e-mail da própria conta vendedora** — o servidor avisa
> com "Payer and collector cannot be the same user" (use outro e-mail para testar).

No boot o servidor imprime a URL pública:
```
🌍 TÚNEL PÚBLICO ATIVO
https://epicmind-obs-cloud.loca.lt
✅ Subdomínio fixo "epicmind-obs-cloud" ativo — URL estável entre restarts.
```

**Subdomínio fixo**: com `TUNNEL_SUBDOMAIN` definido, a URL pública é sempre a mesma
(`https://<TUNNEL_SUBDOMAIN>.loca.lt`) — o webhook do Mercado Pago configurado no painel
deles não precisa ser reconfigurado a cada restart. Se o subdomínio estiver ocupado no
momento do boot (ex.: instância anterior ainda liberando), o servidor sobe com uma URL
temporária e **re-tenta o subdomínio fixo em segundo plano a cada 30s**. Para URL 100%
garantida, use `PUBLIC_URL` com um domínio próprio.

Essa URL é injetada em `process.env.PUBLIC_URL` e usada como prefixo de **todas as URLs
voltadas ao usuário externo**: `back_url`/`notification_url` do Mercado Pago, `/checkout`,
`/termos`, `/privacidade` e callbacks OAuth. `config.publicBase()` / `config.publicUrl(path)`
leem `process.env.PUBLIC_URL` dinamicamente; `GET /api/v1/info` expõe `publicBase` + `links`.

### Cliente (.exe distribuído) → `server-config.json`

O app precisa saber a URL do servidor. Ordem de resolução em `services/apiClient.js`:
`process.env.PAYMENT_SERVER_URL` → `obs-cloud-client/server-config.json` (`serverUrl`) →
fallback embutido. **Antes de gerar o .exe**, coloque em `server-config.json` a URL pública
do seu servidor (o subdomínio fixo do túnel ou seu domínio). Em dev local:
`PAYMENT_SERVER_URL=http://localhost:3002`.

O app abre Termos/Privacidade/checkout sempre pela **URL pública** descoberta em
`/api/v1/info` — funciona para qualquer usuário, não só para quem roda o servidor.

### Painel admin (RBAC)

O painel (`http://localhost:3002/admin`) exige **login** e a conta precisa ter `is_admin = 1`:

- Defina `ADMIN_EMAIL` no `.env` — a conta é promovida a admin no próximo boot (após ela
  existir). Vários e-mails: separe por vírgula.
- Ou pelo terminal: `node scripts/make-admin.js <email>` (`--revoke` remove, `--list` lista).

Toda rota `/api/v1/admin/*`, `/users`, `/transactions`, `/promotions` e as ações
(conceder dias 🎁, liberar máquina 🔓) verificam o JWT + `is_admin`:
`401` sem sessão, `403` "Acesso restrito ao administrador" para não-admin.
Rotas públicas continuam abertas: `/termos`, `/privacidade`, `/checkout`,
`/api/v1/settings` (preço), `/api/v1/health`, `/api/v1/versions`.

### Segredos persistentes

Se `JWT_SECRET` / `HWID_SECRET` estão no `.env`, o servidor **nunca** gera valores
aleatórios — as sessões e licenças ativas dos clientes continuam válidas depois de
`pm2 restart` / reboot. Só quando ausentes é que um segredo efêmero é gerado (com aviso no log).

- **Landing page** (vendas + verificação Google): `http://localhost:3002/`
- Painel admin: `http://localhost:3002/admin` (exige login com conta `is_admin = 1`)
- Assinatura (web, com login): `http://localhost:3002/checkout`
- `/termos` e `/privacidade` → redirecionam para `/#termos` e `/#privacidade` (seções da landing)
- Health: `http://localhost:3002/api/v1/health`

### Landing page — `public/index.html`

Arquivo único (Tailwind CDN, dark mode broadcast). Seções: Hero · Recursos · Como funciona ·
Planos + **checkout público** · Termos de Serviço · Política de Privacidade (LGPD + escopo
Google `drive.file` + disclosure de *Uso Limitado*).

- `POST /api/v1/checkout/subscribe` `{email, machineId?}` — cria a conta (sem senha) e a
  assinatura recorrente **sem login**. A máquina é vinculada no 1º acesso ao app.
- Preço vem de `settings.licensePrice`; link de download vem da última versão publicada em
  *Versões & Updates*.
- O painel antigo foi para `public/admin.html`, servido em `/admin`.
- **Verificação Google**: exige `PUBLIC_URL` com **domínio próprio** (URLs de localtunnel
  não são aceitas). Troque `contato@obsautomacao.com.br` e "EpicMind Broadcast" pelos seus dados.
- O escopo do Google Drive no cliente foi reduzido para `drive.file` (só o que o app cria).

### 2. Cliente

```bash
cd obs-cloud-client
npm install
npm start
```

Abre a **tela de login** (email/senha ou "Entrar com Google"). Após entrar, o app abre e
começa o *heartbeat* de licença.

### 3. Gerar o instalador `.exe` (Windows)

```bash
cd obs-cloud-client
# 1. edite server-config.json com a URL pública do seu payment-server
npm run build:win
```

Gera `dist/OBS-Automacao-Setup.exe` (~98 MB) — instalador NSIS (escolha de pasta, atalhos
Desktop + Menu Iniciar, ícone gerado em `build/icon.png`). **Suba esse arquivo no GitHub
Release com o nome exato `OBS-Automacao-Setup.exe`** — é o nome que a landing e o painel
esperam. `npm run icon` só (re)gera o ícone.

`server-config.json` define para qual servidor o `.exe` fala:
```json
{ "serverUrl": "https://SEU-DOMINIO-OU-TUNEL" }
```
Em dev: `PAYMENT_SERVER_URL=http://localhost:3002 npm start`.

## Autenticação

- **Email + senha** — `bcrypt`, cadastro e login em `/api/v1/auth/register` e `/login`.
- **Entrar com Google** — OAuth *loopback*: o app sobe um servidor temporário em
  `http://localhost:3000`, o Google redireciona pra lá com o `code`, o app troca por tokens
  **sem nenhum copiar/colar**. O mesmo fluxo já conecta o Google Drive.
  Fallback manual (colar código) continua no painel de configuração.
- Toda sessão retorna um **JWT assinado pelo servidor** (`JWT_SECRET`). O cliente não consegue
  forjar status de licença localmente.

## Assinatura recorrente (Mercado Pago)

1. Cliente logado → botão **"Assinar"** (menu Conta) → `POST /api/v1/subscription/create`.
2. Servidor cria um **preapproval** (`/preapproval`, `frequency: 1 months`, `back_url:
   ${PUBLIC_URL}/checkout-success`) e devolve o `init_point`.
3. O checkout do Mercado Pago abre no navegador; o app faz *polling* em
   `/api/v1/subscription/status?preapprovalId=...`.
4. Ao ficar `authorized`, o servidor marca a licença `active` e soma **+30 dias**
   em `valid_until`. O app libera no próximo heartbeat (ou na hora, via evento).
5. Toda fatura mensal paga chega pelo **webhook** (`/api/v1/webhook/mercadopago`,
   tópicos `subscription_preapproval` / `subscription_authorized_payment` / `payment`) e
   renova mais **+30 dias**.

> O webhook exige URL pública — por isso o túnel sobe junto com o servidor. Sem túnel
> (`ENABLE_TUNNEL=false` e sem `PUBLIC_URL`), a criação de assinatura responde `503 needsTunnel`.

> **Testar com Mercado Pago**: você NÃO pode assinar usando o e-mail da própria conta
> vendedora (dono do access token) — o MP recusa com "Payer and collector cannot be the
> same user". O servidor detecta isso e responde `409` com mensagem clara. Para testar,
> logue no app com OUTRO e-mail (em sandbox, um "usuário de teste" comprador).

Novas contas ganham **7 dias de teste** (`settings.trialDays`). Depois é necessário assinar.

## DRM / Anti-pirataria

Regra: **1 conta = 1 máquina**. O mesmo usuário na mesma máquina sempre funciona
(inclusive re-assinando). Contas diferentes na mesma máquina também funcionam. O que
trava é usar **a mesma conta em outra máquina**.

- **HWID estável**: o cliente gera um id local aleatório uma única vez e o persiste em
  disco (`<userData>/.machine-id`), combinado com hostname/usuário. Não muda quando o
  usuário troca de Wi-Fi para cabo, atualiza driver, etc.
- **Vínculo de máquina**: na 1ª ativação o servidor assina `HMAC(userId::machineId,
  HWID_SECRET)` e devolve a assinatura. O cliente reenvia em todo heartbeat; assinatura
  inválida → `block`. `machineId` diferente do vinculado → `409 machineMismatch`
  ("já ativa em outro computador"). O admin libera em **Painel → 🔓** (ou
  `/api/v1/admin/release-machine`).
- Quando a fórmula do HWID muda entre versões, uma migration limpa os vínculos antigos
  uma única vez (re-vinculam no próximo login) para ninguém ficar preso.
- **Trava OFFLINE**: heartbeat a cada `heartbeatIntervalMinutes` (padrão 5). Se o cliente
  ficar sem contato com o servidor por mais de `offlineGraceMinutes` (padrão 15) **ou** a
  licença expirar, a interface é coberta por um overlay de bloqueio — zero uso offline
  prolongado.
- **JWT**: a resposta de licença traz sempre um token novo, assinado, com TTL curto
  (`JWT_TTL_SECONDS`, padrão 30 min).

## Banco de dados (SQLite)

Migrations rodam sozinhas no `npm start` (idempotentes, preservam dados existentes):

| Tabela | Conteúdo |
|---|---|
| `users` | id, email, name, password_hash, google_id, auth_provider, is_admin, blocked, app_version |
| `licenses` | user_id, machine_id, machine_id_sig, status (trial/active/pending/cancelled/expired), plan, mp_preapproval_id, valid_until |
| `transactions` | pagamentos/faturas (idempotente por `txn_<paymentId>`) |
| `promotions` | cupons (uso só conta em pagamento aprovado) |
| `versions` | version, downloadUrl, changelog, `disabled` (kill-switch), `forceUpdate`, `isLatest`, `isObsolete` |
| `messages` / `message_reads` | avisos no início do app + quem já dispensou |
| `settings` | licensePrice, trialDays, heartbeatIntervalMinutes, offlineGraceMinutes… |
| `logs` | webhook / heartbeat / auth / subscription / system / error |
| `webhook_events` | dedupe das notificações do Mercado Pago |

## Gestão de contas (painel admin → Licenças & Máquinas)

Por linha de usuário: 🎁 conceder dias · 🔓 **apagar máquina** vinculada · 🚫 **apagar
assinatura** (cancela no Mercado Pago + expira agora) · 🗑️ **apagar licença** (reset total —
vira trial no próximo acesso) · 🔑 redefinir senha · 🚫 **bloquear / desbloquear usuário**
(login e heartbeat passam a recusar com o motivo que você digitar).

Endpoints: `POST /api/v1/admin/user/block` · `/user/unblock` · `/user/set-password` ·
`/subscription/cancel` · `/license/delete` · `/release-machine` · `/grant`.
O usuário troca a própria senha em `POST /api/v1/auth/change-password`.

## Controle de versões (painel admin → Versões & Updates)

O cliente manda `appVersion` em todo heartbeat. Por versão você controla:

- **Desligar** (`disabled`) — kill-switch: **todas as máquinas** rodando essa versão são
  bloqueadas no próximo heartbeat (~5 min) e veem a tela "Atualização necessária" com o
  botão de download. Religa quando quiser.
- **Obrigatória** (`forceUpdate`) — bloqueia até a pessoa baixar a nova.
- **Atual** (`isLatest`) — define qual URL de download o app oferece.
- Versão desconhecida (build local) → não bloqueia, só sugere atualizar.

Fluxo: gere o `.exe`, publique no GitHub Releases, e cadastre a versão no painel com a URL
do `.exe`. Para tirar uma versão do ar em todas as máquinas, clique em **Desligar**.

## Avisos no app (painel admin → Avisos no App)

Mensagens exibidas em modal quando o usuário abre o OBS Cloud: **novidade / promoção /
aviso / atualização**. Escolha o público-alvo (todos, em teste, assinantes, expirados),
um botão opcional (rótulo + URL) e se pode ser dispensado. O painel mostra quantos já viram.
O cliente busca em `GET /api/v1/messages` no start e marca lido em `POST /api/v1/messages/:id/read`.

## Execução 24/7 (PM2)

```bash
npm i -g pm2
cd payment-server
pm2 start ecosystem.config.js
pm2 save
pm2 startup          # Linux/VPS: auto-start no boot
```
Windows Server: `pm2 resurrect` no logon (Agendador de Tarefas) ou `pm2-installer`.
Logs: `pm2 logs obs-cloud-payment` · `pm2 monit`.

## Uso do app

### Conectar ao OBS
Configurações (Ctrl+Shift+L) → IP/porta/senha do WebSocket → "Testar Conexão OBS".

### Decupagem ao vivo por palestrante
1. Cole o **Link da Tabela** (Google Sheets publicado como CSV) → 🔄 carrega os participantes.
2. "Injeção Dinâmica" → mapeie os nomes das fontes de texto do OBS → "Testar & Confirmar".
3. Configurações → ative **Gravação Inteligente**.
4. Selecione o participante → **▶ Iniciar Pronunciamento** (injeta os textos + grava).
5. **⏹ Encerrar** → o vídeo vira `Nome/Nome_01.mp4` na pasta de destino.

### Uploads
- "Upload Automático" (Configurações) envia a cada 15 min o que ainda não foi enviado.
- Cada pessoa ganha uma subpasta em `EPICMINDMARKETING` no Drive. Resumable uploads.

## Endpoints principais

| Método | Rota | Descrição |
|---|---|---|
| POST | `/api/v1/auth/register` `/login` `/google` | contas / sessão |
| GET | `/api/v1/auth/me` | sessão atual (Bearer) |
| POST | `/api/v1/license/heartbeat` `/check` | DRM (Bearer + machineId + sig) |
| POST | `/api/v1/subscription/create` | cria preapproval |
| GET | `/api/v1/subscription/status?preapprovalId=` | polling |
| POST | `/api/v1/subscription/cancel` | cancela |
| POST/GET | `/api/v1/webhook/mercadopago` | notificações MP |
| GET | `/api/v1/google/callback` | OAuth server-side (domínio estável) |
| GET | `/api/v1/health` · `/api/v1/info` | status / URL pública |
| — | `/` `/checkout` `/checkout-success` `/termos` `/privacidade` | painel + páginas |

## Segurança

- Cliente: `contextIsolation: true`, `nodeIntegration: false`, sandbox, CSP em todas as telas.
- Servidor: Helmet, JWT, segredos via `.env`, `ADMIN_TOKEN` opcional no painel.
- **Produção**: sempre fixe `JWT_SECRET` e `HWID_SECRET`; use token `APP_USR-` do Mercado Pago.

## Licença

ISC
