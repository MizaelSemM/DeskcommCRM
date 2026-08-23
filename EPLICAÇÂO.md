# EPLICAÇÃO — DeskcommCRM explicado para quem nunca viu um sistema desse porte

> Documento gerado por leitura real do código-fonte deste repositório (estado de ago/2026).
> Tudo aqui foi confirmado em arquivo e linha; o que for inferência está marcado.
> Conceitos técnicos novos são explicados em uma frase antes do primeiro uso.

## Glossário rápido

Os conceitos que vão aparecer o tempo todo:

| Conceito | Em uma frase |
|---|---|
| **Container / imagem** | Imagem é um "molde" congelado do seu programa + dependências; container é esse molde rodando, isolado da sua máquina. |
| **Volume (Docker)** | Pasta que sobrevive à morte do container — onde dados persistentes ficam. |
| **Webhook** | Quando um sistema chama **o seu** servidor via HTTP para avisar "aconteceu algo" (em vez de você ficar perguntando). |
| **HMAC** | Assinatura criptográfica de uma mensagem: quem recebe confere a assinatura com um segredo compartilhado, para provar que a mensagem não foi forjada nem alterada. |
| **Multi-tenant** | Uma única instalação serve vários clientes ("tenants"/organizações), cada um enxergando só os próprios dados. |
| **RLS (Row Level Security)** | Recurso do Postgres em que **o próprio banco** filtra linhas por usuário — mesmo se a query sair errada, o banco não devolve linha de outro tenant. |
| **Service role key** | Chave "mestra" do Supabase que **ignora RLS** — usar errado é vazar dados de todos os tenants. |
| **Idempotente** | Operação que pode repetir sem causar efeito duplicado (receber o mesmo webhook 2× grava 1 mensagem, não 2). |
| **Fila** | Lista durável de trabalhos pendentes: quem produz só escreve na lista, quem consome tira no seu ritmo. |
| **Worker** | Processo que fica rodando tirando trabalhos de uma fila e executando. |
| **Cron** | Agendador clássico de Linux: roda um comando a cada X minutos/horas. |
| **Embedding / RAG** | Embedding = texto virado num vetor de números parecido por "significado"; RAG = buscar trechos relevantes por similaridade e colar no prompt da IA antes de responder. |
| **MCP** | Protocolo aberto (Model Context Protocol) que expõe funções de um sistema como "ferramentas" que um agente de IA pode chamar. |
| **Pooler** | Serviço intermediário que reaproveita conexões de banco (Postgres aguenta poucas conexões diretas; muitos clientes precisam de muitas). |
| **BYOK** | Bring Your Own Key — cada organização cadastra a própria chave de IA em vez de usar a chave da plataforma. |

---

## 1. Visão geral da arquitetura

### As peças

| Peça | Onde vive | O que faz |
|---|---|---|
| **App Next.js 16** | `app/`, `components/`, `lib/` | A UI (inbox, kanban, configurações) **e** a API REST (`app/api/v1/` — ~166 route handlers). Um único processo Node faz os dois. |
| **Supabase (nuvem)** | serviço externo | Postgres + Auth (login) + Realtime (push pro browser) + Storage (arquivos de mídia). O schema do banco está em `supabase/baseline.sql`. |
| **WAHA Plus** | container Docker (`docker-compose.yml`, serviço `waha`) | Ponte com o WhatsApp: software de terceiros (devlikeapro) que mantém a sessão logada (via QR code) e fala HTTP. |
| **Worker do agent-engine** | `workers/agent-worker/main.ts` (container `worker`) | Processo 24/7 separado que consome a fila de turnos do agente de IA, follow-ups e watchdog de sessão. |
| **Scheduler** | `docker/scheduler/entrypoint.sh` (container `scheduler` em prod) | Um crond dentro de container que dá `curl` nos endpoints `/api/v1/cron/*` do app a cada minuto/5min/etc. |
| **Redis (+ SRH)** | containers em prod | Upstash Redis (ou Redis local atrás de `serverless-redis-http`) para rate limit e debounce. |
| **Caddy ou Traefik** | containers em prod | Proxy reverso que termina HTTPS e aponta seu domínio para o container do app. |

### Como se conectam

```
                 ┌──────────────────── SEU SERVIDOR (VPS) ────────────────────┐
                 │                                                             │
Cliente WhatsApp │ [WAHA] ──webhook HTTP──▶ [App Next.js :3000] ◀──curl──      │
     ◀─QR/mens.─▶│   ▲                          ▲    ▲          [scheduler]    │
                 │   │ resposta                 │    │            (cron)        │
                 │   └────── [worker] ──────────┘    │                          │
                 │            lê/escreve direto      │                          │
                 │            no Postgres            │                          │
                 └───────────────────────────────────┼──────────────────────────┘
                                                     │
                              ┌──────────────────────┴──────────────────────┐
                              │ SUPABASE (nuvem): Postgres+RLS, Auth,       │
                              │ Realtime, Storage                           │
                              └──────────────────────┬──────────────────────┘
                                                     │ postgres_changes (push)
                                             [Browser do atendente]
```

Em cadeias:

- **Receber mensagem**: WhatsApp → WAHA → `POST /api/v1/webhooks/waha[/token]` (app Next) → tabelas `messages`/`conversations` (Supabase) → trigger de banco grava em `event_log`/`job_queue` → worker roda a IA → IA chama a API de envio → WAHA → WhatsApp.
- **Ver na tela**: insert em `messages` → Supabase Realtime empurra `postgres_changes` pro browser → hook `hooks/inbox/useMessagesRealtime.ts` invalida o cache do React Query → tela atualiza.
- **Atendente responde**: browser → `POST /api/v1/messages` (`app/api/v1/messages/_handler.ts`) → adapter WAHA (`lib/channels/adapters/waha.ts`) → WAHA → WhatsApp.

Ponto de doutrina repetido no repo inteiro: **trigger de Postgres NUNCA faz chamada HTTP**. O banco só grava linhas nas filas (`event_log`/`job_queue`); quem fala com o mundo exterior é código TS (workers/crons).

---

## 2. O que é o Docker AQUI, especificamente

Existem **quatro** arquivos de compose e mais de um Dockerfile, com papéis diferentes.

### `docker-compose.yml` (DEV ONLY — o próprio cabeçalho diz isso)

Sobe exatamente **dois serviços**:

1. **`waha`** — imagem pronta `devlikeapro/waha:noweb`. Destaques (linhas 27–47):
   - `WHATSAPP_HOOK_URL`: para onde o WAHA avisa quando chega mensagem. Default `http://host.docker.internal:3003/api/v1/webhooks/waha`. **`host.docker.internal`** é o truque que faz o container enxergar seu computador (dentro do container, `localhost` seria ele mesmo).
   - `WHATSAPP_HOOK_HMAC`: segredo que **assina** cada webhook — o app confere com `crypto.timingSafeEqual`.
   - Volumes `waha-data` (`.sessions`) e `waha-media`: sessão pareada e mídias sobrevivem a restarts (sem isso, todo restart pediria QR de novo).
   - Porta mapeada `"3030:3000"`: o painel do WAHA fica em `http://localhost:3030/dashboard/` porque a 3000 é do app.
2. **`worker`** — construído do `Dockerfile.worker`; roda o agent-engine 24/7. Recebe `.env` + `.env.local` e sobrepõe `WAHA_API_BASE_URL=http://waha:3000` (nome do serviço na rede interna do compose — dentro da rede Docker, os serviços se acham pelo nome).

```bash
docker compose up -d   # sobe WAHA + worker
pnpm dev               # app Next roda FORA do Docker, no seu PC
```

**Como conversam:** o WAHA (container) entrega webhooks chamando `host.docker.internal:3003` (seu PC, onde o `pnpm dev` escuta); o app chama o WAHA de volta em `WAHA_API_BASE_URL` apontando pra `localhost:3030`. O banco **não** está no Docker: vem do Supabase na nuvem.

### `Dockerfile` (raiz) — a imagem de PRODUÇÃO do app

Build multi-estágio: `deps` (instala dependências pnpm) → `build` (roda `pnpm build`, gera `.next/standalone`) → `runner` (imagem slim alpine, usuário non-root, ffmpeg instalado pra derivar vídeo recebido).

Dois detalhes que revelam o design:

- **Imagem genérica**: os `NEXT_PUBLIC_*` recebem *placeholders* no build (linhas 24–27); os valores reais são injetados **em runtime** (no browser via `<PublicEnvScript/>`, no servidor via `lib/env.ts`). Uma imagem só serve qualquer instalação, sem rebuild — essencial porque o cliente self-host só faz `pull`.
- Quem builda é o **CI** (`publish-image.yml` publica 3 imagens: `app`, `worker`, `scheduler`). Buildar na VPS custa ~4 GB de heap e é escape de emergência (`docker-compose.build.yml`), nunca o caminho normal.

### `docker-compose.prod.yml` (+ `traefik.yml`, `build.yml`)

Sete serviços, todos com `image:` publicado (nenhum builda na máquina do cliente):

| Serviço | Imagem |
|---|---|
| `app` | `ghcr.io/melgarafael/deskcommcrm:stable` |
| `worker` | `ghcr.io/melgarafael/deskcomm-worker:stable` |
| `waha` | `devlikeapro/waha:latest-2026.7.2` (tag fixada por licença upstream) |
| `redis` | `redis:7-alpine` |
| `srh` | `hiett/serverless-redis-http` (pinado por digest) — adapta Redis local pra API estilo Upstash |
| `scheduler` | `ghcr.io/melgarafael/deskcomm-scheduler:stable` |
| `caddy` | HTTPS automático; desligável quando a VPS já tem proxy reverso próprio |

O `docker-compose.traefik.yml` adiciona as **labels** de roteamento ao serviço `app`; por isso o runbook (`docs/runbooks/deploy.md`) exige subir sempre com os dois `-f` — esquecer o segundo recria o contêiner sem labels e o domínio responde 404 com tudo "saudável".

### Resumindo: o que roda em container?

- No **dev**: só WAHA (+ worker, se você subir com `up -d`). O app roda no seu PC via `pnpm dev`. Banco é Supabase cloud.
- Em **produção self-host**: TUDO em container (app, worker, waha, redis, srh, scheduler, caddy). Banco continua sendo Supabase (cloud ou próprio).

---

## 3. Fluxo de uma mensagem de WhatsApp, do início ao fim

Cliente manda "oi" pro número da empresa:

**1. WhatsApp → WAHA.** O WAHA (sessão NOWEB pareada via QR) recebe o evento `message.any`.

**2. WAHA → webhook do app.** WAHA faz `POST` no `WHATSAPP_HOOK_URL`, assinando o corpo com HMAC-SHA512 (header `x-webhook-hmac`).
   - Rota por-tenant: `app/api/v1/webhooks/waha/[token]/route.ts` — resolve a organização pelo `webhook_path_token` único de cada linha de `channel_sessions` (linhas 86–96).
   - Rota global: `app/api/v1/webhooks/waha/route.ts` — resolve pela sessão (`body.session`) e responde `session_not_registered` se não conhece (pra o WAHA não retentar eternamente).
   - Validação: `lib/waha/webhook-auth.ts:50–73` compara a assinatura com `timingSafeEqual`; todo evento cru é arquivado em `webhook_events_log` (auditoria e debug).

**3. Ingestão** (`lib/waha/ingest.ts`, função `dispatchWahaEvent`, linhas 906–929):
   - Upsert (atualiza ou cria) **atômico** de contato e conversa via RPCs `fn_upsert_wa_contact` / `fn_upsert_wa_conversation`;
   - INSERT em `messages`, idempotente pela constraint única `(organization_id, external_id)` — se o mesmo webhook chegar 2×, a segunda tentativa cai no erro Postgres `23505` (violação de unique) e é tratada como dedup legítimo (`ingest.ts:526–547`);
   - `message.ack` atualiza status de entrega/leitura; `session.status` espelha saúde da conexão.

**4. Efeitos pós-entrada** (`lib/channels/pos-entrada.ts:214–239`): detecção de opt-out ("STOP") → nascimento do lead no CRM se for contato novo → `emit_event('ai_agent.dispatch_requested')` na tabela `event_log`. **É esse evento que acorda a IA.**

**5. Fila → job.** O drain do engine (`lib/agent-engine/edge/crm/drain.ts:52–118`) transforma o evento num job `inbound_turn` na tabela `job_queue`: agrupa rajadas (debounce — várias mensagens seguidas do mesmo contato viram um turno só), pula grupos e orgs sem agente publicado, e espera (até 45s) a derivação de mídia (áudio transcrito, frames de vídeo) quando a mensagem tem anexo.

**6. Turno do agente.** O worker (`workers/agent-worker/main.ts`) executa `runAgentTurn` (`lib/agent-engine/agent/inbound-turn.ts:949+`):
   - monta contexto: lead, memória da organização, histórico da conversa;
   - loop de ferramentas: `search_knowledge` (RAG), `update_lead_state`/`crm_move_lead_stage` (mover no funil), `schedule_followup`, e **`send_message`** (toda saída da IA é uma tool call — ela nunca "fala" direto);
   - tools extras do catálogo MCP habilitadas na tela (`lib/agent-engine/edge/crm/mcp-tools.ts`);
   - **guardrails before-send** (`lib/agent-engine/guardrails/before-send.ts:8–37`): cadeia de portões que toda mensagem candidata atravessa antes de sair — opt-out, LGPD, pacing anti-banimento, janela de 24h do WhatsApp, spinning (variação de texto pra não parecer robô), bloqueio de promessas proibidas, vazamento interno, disclosure de que é IA;
   - anti-banimento (`lib/agent-engine/pacing/engine.ts:56–109`): janela de horário (fuso do tenant, domingo evitado) → teto diário com warm-up conforme idade do número → throttle + jitter aleatório.

**7. Envio da resposta.** A tool `send_message` chama o MESMO handler HTTP que o atendente humano usa (`app/api/v1/messages/_handler.ts`), que grava em `messages` com ledger idempotente (`send_ledger`) e despacha pelo adapter WAHA → WAHA → WhatsApp.

**8. Tela do atendente, em paralelo:** desde o passo 3, cada insert em `messages` dispara Supabase Realtime `postgres_changes`; `hooks/inbox/useMessagesRealtime.ts:61–73` escuta filtrando por `conversation_id` e `organization_id` e invalida o React Query — o balão aparece sozinho, sem F5.

### E se for o atendente respondendo?

Browser → `POST /api/v1/messages` (`app/api/v1/messages/route.ts` → `_handler.ts:261`): valida conversa → grava mensagem com status `queued` → resolve o adapter da sessão → envia via WAHA → marca `sent` + audit log + `emit_event('message.sent')`. Resposta humana silencia a IA por janela deslizante de 5 min (`bot_silenced_until`). No caminho do agente há throttle/jitter; no caminho humano o envio sai imediato.

---

## 4. Autenticação e multi-tenancy

### Login (Supabase Auth)

Supabase Auth é um serviço de identidade gerenciado: ele guarda usuários/senhas (e MFA), emite JWTs e o seu app só confere. Fluxo real:

1. `app/(public)/login/page.tsx` renderiza `components/auth/LoginForm.tsx`, que chama a Server Action `app/actions/auth/signInWithPassword.ts`.
2. A action: valida com Zod → aplica rate limit por IP e por conta → chama `supabase.auth.signInWithPassword()` → se o usuário tem MFA TOTP verificado, devolve `mfa_required` e vai pra `/login/mfa` → audita sucesso/falha com e-mail hasheado.
3. A sessão vira cookie `sb-deskcomm-auth` (`lib/supabase/server.ts:34`): `HttpOnly` (JS não lê), `SameSite=Strict`, `Secure`.

Regra de ouro do repo: no servidor sempre `getUser()` (revalida o JWT com o Supabase), **nunca** `getSession()` (confia no cookie sem validar). O middleware `proxy.ts` (Next 16 renomeou `middleware.ts` → `proxy.ts`):
- injeta `X-Request-Id` em toda request (correlação com audit log);
- deixa passar os paths públicos (`lib/auth/public-paths.ts`: login, webhooks, `/api/v1/cron/*`, `/api/mcp` — estes têm guard próprio dentro da rota);
- para todo o resto, valida o JWT; sem usuário → API responde 401 JSON, UI redireciona pra `/login?next=...`.

### Organização (tenant) e RLS

Uma **organização** = um cliente (um e-commerce, uma clínica). Cada linha de `organizations` é um tenant; todo usuário pertence a organizações via `user_organizations` (com papel `viewer < agent < manager < admin`).

**RLS** (Row Level Security) é o banco filtrando linhas automaticamente por quem está perguntando. O mecanismo concreto deste repo:

- Função canônica `fn_user_org_ids()` (`baseline.sql:788–794`): retorna as orgs do usuário logado (`auth.uid()`) cuja membership não foi revogada. É `SECURITY DEFINER` (roda com privilégios do criador) justamente para não recursar na própria RLS de `user_organizations`.
- Toda tabela tenant-aware tem policy chamada `tenant_isolation_<tabela>_all`, exemplo real de `contacts` (`baseline.sql:4392`):

```sql
create policy tenant_isolation_contacts_all
  using (
    organization_id in (select fn_user_org_ids())
    or fn_is_platform_admin()
  )
  with check (...mesma coisa...);
```

Ou seja: o Postgres só devolve (e só deixa escrever) linhas cujo `organization_id` esteja na lista de orgs do usuário. Um usuário da org A consultando `messages` enxerga zero linhas da org B — mesmo com query errada no código.

**A exceção perigosa:** a service role key bypassa RLS. 89 dos ~169 handlers usam ela (para performance/administração); nesses, filtrar `organization_id` manualmente — resolvido de fonte confiável (cookie validado, JWT, segredo de webhook, token do path), **nunca do body da requisição** — é responsabilidade do autor do handler.

O guard canônico é `requireRole()` (`lib/auth/require-role.ts`): carrega o usuário via JWT → resolve a org ativa (`resolveActiveOrg()`, `lib/auth/server.ts:188–201`: cookie `active_org` **validado contra as memberships reais**, nunca confiado cego) → pergunta o papel ao banco via `fn_user_role_in_org()` → rank insuficiente = 403 com audit. Papéis de plataforma (super-admin transversal) passam por `platform_admins` + `requirePlatformAdmin`.

E existe prova: `tests/invariants/rls-isolation.test.ts` cria duas organizações, simula os claims JWT pelo mesmo caminho das policies de produção e demonstra que o usuário da org A lê **zero linhas** da org B em 12 tabelas (com caso de controle provando que as linhas da B existem).

### `scripts/bootstrap-owner.ts`, linha a linha (em termos simples)

O app **não tem tela de cadastro**. Este script cria o primeiro dono da instalação, de forma idempotente (rodar de novo não duplica nada):

1. **`loadEnv()` (24–35)**: junta as variáveis do ambiente com o que houver em `.env` / `.env.local`.
2. **Validação (39–50)**: exige `NEXT_PUBLIC_SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `OWNER_EMAIL` e `OWNER_PASSWORD` — sem isso, erro e para.
3. **`slugify()` (52–61)**: transforma "Padaria do Zé" em `padaria-do-ze` (minúsculo, sem acento, máx. 40 chars) porque a coluna `slug` só aceita esse formato.
4. **Client admin (63–65)**: cria o client Supabase com a service role key — aqui é legítimo, é script de bootstrap rodado pelo dono da máquina.
5. **`ensureOwnerUser()` (67–84)**: lista os usuários do Auth; se o e-mail já existe, apenas **redefine a senha**; senão cria o usuário já confirmado (`email_confirm: true` — sem e-mail de verificação).
6. **`ensureOrg()` (86–112)**: procura `organizations` pelo slug; se não existe, insere a organização. Um trigger de banco (`fn_seed_org_llm_defaults`) semeia defaults de IA nela.
7. **`aplicarProvedorEscolhido()` (130–160)**: se você passou `AI_PROVIDER=openrouter` no instalador, grava isso em `organizations.settings.llm.provider` (o trigger semeia `anthropic` fixo). Se falhar, não derruba a instalação — só avisa que dá pra trocar depois na tela Provedores.
8. **`ensureMembership()` (162–186)**: garante linha em `user_organizations` ligando dono↔organização com `role='admin'` (cria ou reativa).
9. **`ensurePlatformAdmin()` (188–220)**: insere em `platform_admins` (super-admin de plataforma, que enxerga todos os tenants), com `granted_by` = ele mesmo (auto-concessão de bootstrap) e `mfa_required: false` explícito — o comentário explica: o default da coluna era `true` e colocaria um bloqueador de tela cheia em TODA instalação nova logo após o onboarding.
10. **`main()` (222–228)**: executa os quatro passos em ordem e imprime o resumo ("faça login e conclua o onboarding").

É exatamente o passo 7 do `hostgator-setup-kit/install.sh` ("Bootstrap do 1º admin").

---

## 5. Banco de dados

### Migrations vs `baseline.sql` — por que os dois existem

- **Migration** (`supabase/migrations/*.sql`) = um arquivo versionado que descreve UMA mudança no schema ("crie a tabela X", "adicione a coluna Y"). É o diário histórico do banco.
- **Baseline** (`supabase/baseline.sql`, ~14 mil linhas) = o estado COMPLETO e atual do schema, como um dump. Cada migration nova também entra nele como **apêndice idempotente** (código escrito para poder rodar de novo sem quebrar — "se já existe, ignore").

O detalhe que explica tudo: **as migrations 0001–0009 e 0013 são stubs** — arquivos cujo corpo é só `SELECT 1;` (confirmado em `20260428195354_0001_platform_base.sql:10–13`, com comentário dizendo isso). A cadeia histórica **não sobe do zero**: aplicar as migrations numa ordem num banco vazio quebra na 0010, que altera tabelas que as stubs nunca criaram (medido: 21 aplicam, 80 falham — `supabase/migrations/MANIFEST.md:40`). O schema real de partida vive no baseline.

Por que importa tanto: quem instala self-host aplica **só o baseline** (`install.sh` passo 6; README §Desenvolvimento avisa "aplique o baseline, NÃO as migrations"). Então uma mudança de schema que não entrar no baseline **nunca chega em quem instalou**. Daí a regra da **tripla**, obrigatória pra qualquer mudança de schema:

1. migration versionada em `supabase/migrations/`,
2. apêndice idempotente em `supabase/baseline.sql`,
3. linha em `supabase/migrations/MANIFEST.md`.

Em dev você também usa o baseline (mesmo banco que o cliente tem); o CI (`invariants`) aplica o baseline duas vezes seguidas com `ON_ERROR_STOP=1` — a segunda passada é a prova de que o arquivo é mesmo idempotente.

### As tabelas mais importantes (posições reais no `baseline.sql`)

| Tabela | Linha | Pra que serve | Colunas/constraints-chave |
|---|---|---|---|
| `organizations` | 1746 | O tenant. Cada linha = 1 cliente da instalação | `slug` único; `status` (active/suspended/redacted/archived); `settings` jsonb (visibilidade, roteamento, LLM); `ai_budget_cents` |
| `user_organizations` | 1854 | Associação usuário↔organização com papel | `role` CHECK viewer/agent/manager/admin; hierarquia 1<2<3<4; `revoked_at` desliga |
| `platform_admins` | 1789 | Super-admins transversais (enxergam todos os tenants) | `scope` full/support_readonly; `mfa_required`; mudança exige DBA + dupla confirmação |
| `contacts` | 1345 | Pessoa física no escopo do tenant | `organization_id` NOT NULL; CPF cifrado (`cpf_encrypted` + hash p/ busca); anonimização irreversível |
| `conversations` | 1386 | Atendimento 1:1 por canal WhatsApp | status open/pending/resolved/**ai_handling**/closed…; `assigned_to_user_id`; única 1:1 por contato+sessão |
| `messages` | 1657 | Mensagens in/out | direction/type/status/sent_via; `sent_by_user_id` distingue humano de IA; unique `(organization_id, external_id)` = idempotência da ingestão |
| `crm_leads` (+ `crm_pipelines`/`crm_stages`) | 1462 | Negócios no funil kanban | `pipeline_id`/`stage_id`; `value_cents`+`currency`; motivo de perda obrigatória quando lost |
| `event_log` | 1543 | Barramento de eventos interno ("o que aconteceu") | `event_type` formato `a.b`; status pending/processing/done/dead; `consumed_by`, tentativas/backoff |
| `job_queue` | 6500 | Fila durável do agent-engine | kind inbound_turn/followup_turn/watchdog/flywheel; 1 job running por contato; dedup evento→job |
| `channel_sessions` | 1315 | Sessão WhatsApp pareada (número) | `webhook_path_token` + `webhook_secret_encrypted` (identidade do webhook); status STARTING→WORKING |
| `ai_chunks` / `ai_knowledge_sources` | 1070 | RAG: trechos indexados com embedding | `embedding vector(1536)` + índice ivfflat; org-scoped; versionamento da base de conhecimento |
| `ai_provider_credentials` | 1213 | BYOK: chave de IA de cada organização | `api_key_encrypted/iv/tag` (AES-256-GCM); provider anthropic/openai/google |

---

## 6. Workers e filas

### O que são as filas aqui

Duas filas, ambas **tabelas Postgres** (não é Redis/RabbitMQ):

- **`event_log`** — o barramento de "coisas que aconteceram". Triggers de banco e Server Actions inserem linhas via `emit_event()`. Exemplos: `message.received`, `media.persist_requested`, `knowledge_source.updated`.
- **`job_queue`** — a fila de trabalho do agent-engine (turnos do agente, follow-ups agendados, watchdog).

Por que tabela e não serviço de fila dedicado? Porque o banco já está lá, é transacional com o resto dos dados e sobrevive a restart. O truque clássico de claim seguro: `SELECT ... FOR UPDATE SKIP LOCKED` — cada consumidor trava a linha que pegou e pula as linhas que outro já travou, então vários workers podem drenar a mesma fila sem processar o mesmo job duas vezes.

### O que são os arquivos em `workers/`

Cada par `*-worker.ts` + `*.handler.ts` consome um tipo de evento do `event_log`:

| Worker | Consome | Faz |
|---|---|---|
| `ai-response-worker` | `message.received` | resposta IA legada (hoje auto-skipa quando o engine tem versão publicada) |
| `ai-sentiment-worker` | `message.received` | classifica sentimento; abaixo do limiar emite alerta/handoff |
| `rag-indexer` | `knowledge_source.updated`, produtos Nuvemshop | chunka → gera embeddings → grava em `ai_chunks` (com debounce 30s e versionamento da KB) |
| `media-persist` / `media-derive` | `media.persist_requested` | baixa mídia pro Storage; deriva áudio/frames de vídeo |
| `lgpd-export` / `lgpd-redact` | pedidos LGPD | monta export PDF / anonimiza titular |
| `storage-cleanup` | rotina | limpa órfãos do Storage |

E `workers/agent-worker/main.ts` é o worker novo (agent-engine): ele próprio faz o drain do `event_log`→`job_queue` e executa os handlers `inbound_turn`, `followup_turn`, `case_reply_turn`, `operator_turn` (`main.ts:498–505`). Além disso roda relógios internos: watchdog de sessão (reconcilia sessões × WAHA e reenvia mensagens presas), flywheel de auto-aprimoramento etc.

### Como ficam rodando "sempre" — quem precisa estar ligado?

Três mecanismos, dependendo do ambiente:

1. **Produção self-host:** containers `worker` (processo Node permanente) e `scheduler` (crond dentro de container cujo entrypoint gera um crontab e dá `curl` autenticado com `Authorization: Bearer INTERNAL_SECRET` nos endpoints `/api/v1/cron/*` — 18 rotas agendadas, ex.: `event-log-drain` e `agent-dispatcher` a cada minuto). Um teste unitário compara essa lista com o diretório de rotas: cron novo sem agendamento reprova o CI.
2. **Instalações mais antigas / host:** crons no crontab da VPS instalados pelo `install.sh` (`setup_event_log_drain_cron`: `* * * * * curl … /api/v1/cron/event-log-drain`; `setup_update_agent_cron` alimenta o botão "Atualizar agora" da tela).
3. **Dev local:** `docker compose up -d` sobe o container `worker`. Sem ele (ou com `EVENT_LOG_WORKER_ENABLED=false`), os eventos ficam na fila esperando — nada explode, só não anda até alguém drenar.

**O Next.js sozinho NÃO cuida disso** — o app é um processo request/response; os consumidores precisam de processo próprio ou do scheduler batendo nos endpoints de cron.

---

## 7. IA / Agente

### Qual modelo é usado? A decisão tem camadas

Para o **agente** (produção): a config é lida do banco **a cada chamada** (`lib/agent-engine/edge/llm/credentials.ts`):
- `organizations.settings.llm.provider` + `default_model` — escolhidos na instalação (`AI_PROVIDER`) ou na tela Agente de IA › Provedores;
- credencial BYOK mais recente ativa de `ai_provider_credentials` (decifrada AES-256-GCM, plaintext só em memória);
- sem BYOK, cai pra chave de plataforma do env conforme o provedor (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `OPENROUTER_API_KEY`);
- teto de gasto da org (`ai_budget_cents`) entra na mesma query — estourou, a IA para ou só avisa, conforme a org escolheu (e a alavanca global `AI_BUDGET_ENFORCEMENT` só afrouxa, nunca aperta).

Detalhe importante documentado no `.env.example`: preencher `OPENROUTER_API_KEY` não TROCA o provedor — só passa a pagar com ela depois que o provedor for OpenRouter. E o painel recusa salvar modelo sem suporte a ferramentas nos pontos principais, porque modelo sem tools não dá erro: responde texto plausível e nunca cria lead nem move card.

Para os **workers legados** (`lib/ai/gateway.ts`): ordem Gateway Vercel (`AI_GATEWAY_API_KEY`) → OpenRouter → provider direto, com bindings por finalidade (`lib/ai/gateway-binding.ts`: o modelo que conversa pode ser diferente do que indexa/embedda).

### RAG por tenant

Cada organização tem base de conhecimento própria: fontes cadastradas na tela Conhecimento viram documentos, que o `workers/rag-indexer.ts` fatia em pedaços (chunks), transforma cada pedaço em embedding (default `openai/text-embedding-3-small`, 1536 dimensões) e grava em `ai_chunks` com `organization_id`. Indexação é debounced (30s) e versionada: cria versão nova da KB e só ativa quando TODOS os chunks entraram — nunca ativa versão pela metade.

Na hora de responder, a tool `search_knowledge` chama a RPC `retrieve_top_k_chunks(p_organization_id, ...)` (`baseline.sql:12218–12238`): busca por similaridade vetorial **filtrando pela organização** — o conhecimento da clínica A nunca vaza na resposta da loja B. No engine, erro de busca nunca lança: vira telemetria e ensino ao modelo.

### Quando a IA não sabe / precisa passar pra humano (handoff)

`lib/agent-engine/agent/human-handoff.ts` — dois gatilhos:
1. **Determinístico**: regex conservadora em PT-BR ("falar com atendente"…) avaliada ANTES do modelo — se casa, nem gasta token;
2. **Pela própria IA**: tool `request_human_handoff`.

A ação `performHumanHandoff`: marca `contacts.force_human=true` (irreversível), transiciona a conversa `ai_handling → pending`, silencia o bot para sempre nela (`bot_silenced_until='infinity'`), cancela follow-ups pendentes e cria item na caixa de handoff com resumo. Turnos futuros desse lead são barrados pelo guard `isLeadInHandoff`. Há ainda handoff automático quando o orçamento de IA acaba, e a pilha antiga soma gatilhos de sentimento negativo (worker `ai-sentiment-worker`) e baixa confiança, avisando o time via Realtime (canal `org:<org>:queue`).

### MCP server interno

`app/api/mcp/route.ts` expõe o CRM como ferramentas MCP (autenticado por Bearer token de `api_tokens`). `lib/mcp/tools/` registra tools de contatos, conversas, leads, mensagens, funis, escalação etc., cada uma com validação Zod, checagem de papel, filtro manual de organização e auditoria obrigatória. É deste catálogo que vêm as tools extras do agente no turno — e é também a API pra agentes externos operarem o CRM.

---

## 8. Variáveis de ambiente, na prática

Fonte: `.env.example` (o template oficial). As mais importantes:

| Variável | O que controla | O que quebra se errada/vazia |
|---|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` | Endereço do seu projeto Supabase | App não conversa com banco/auth — nada sobe |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Chave pública do browser; RLS é o gatekeeper real | Front não autentica nem lê nada |
| `SUPABASE_SERVICE_ROLE_KEY` | Chave mestra que **ignora RLS** — só servidor | Sem ela, handlers admin falham. Vazada = todos os tenants expostos |
| `INTERNAL_SECRET` / `INTERNAL_CRON_SECRET` | Bearer dos endpoints `/api/v1/cron/*` e `/api/internal/*` | Scheduler recebe 401 → follow-ups, drain de fila e rotinas param |
| `WAHA_API_BASE_URL` | Onde o app acha o WAHA | Mensagens não saem; conexão WhatsApp "morta" |
| `WAHA_API_KEY` | Plaintext da chave do WAHA (o container guarda o hash SHA512) | App rejeitado ao chamar WAHA (401) |
| `WAHA_HMAC_SECRET` | Segredo da assinatura dos webhooks do WAHA | Se exigir assinatura, toda mensagem recebida é rejeitada; senão, fica sem proteção |
| `WHATSAPP_RESTART_ALL_SESSIONS` | Lido pelo WAHA: retomar sessões pareadas no restart | Default do WAHA (`False`) = número "conectado" no disco mas mudo na prática |
| `OPENROUTER_API_KEY` | Chave OpenRouter — paga o agente quando o provedor da org é openrouter (e workers legados) | Agente mudo com `LlmNotConfiguredError` (se for o único caminho) |
| `ANTHROPIC_API_KEY` | Fallback/chave direta Anthropic | Idem, quando provedor é anthropic sem BYOK |
| `OPENAI_API_KEY` | Embeddings do RAG (`text-embedding-3-small`) | Conhecimento não indexa; RAG vazio |
| `AI_BUDGET_ENFORCEMENT` | Alavanca de emergência do teto de gasto (`on`/`avisar`/`off`) | Typo cai em `on` de propósito; `off` deixa IA gastar sem freio |
| `UPSTASH_REDIS_REST_URL/TOKEN` | Rate limit + debounce | Degrada pra memória por processo (limite por instância, warn no log) |
| `CPF_ENCRYPTION_KEY` | Cifra o CPF dos contatos (LGPD) | CPFs novos não cifram/legíveis errado |
| `AI_CRED_AES_KEY` | Cifra as credenciais BYOK (`ai_provider_credentials`) | Não consegue salvar/decifrar chave de IA da org |
| `RESEND_API_KEY` / `RESEND_FROM_EMAIL` | E-mail transacional (convites, export LGPD) | Vazio é proposital: convite mostra link na tela; FROM inválido faz todo envio falhar lá na Resend |
| `LGPD_SIGNING_KEY` / `LGPD_DPO_EMAIL` / `LGPD_EXPORT_EXPIRES_HOURS` | Assinatura do export LGPD, DPO impresso no relatório, prazo | Export LGPD quebrado ou sem responsável identificado |
| `AGENT_DISPATCH_CONSUMER` | Dono ÚNICO dos eventos de despacho (`engine`=worker, `native`=app) | Dois consumidores ativos = turno duplicado ou perdido |
| `SUPABASE_DB_URL` | Connection string Postgres direta pro worker (fila SKIP LOCKED, FTS) | Worker do agente não roda |
| `APP_NAME` / `APP_LOGO_URL` / `APP_ACCENT_HEX` | Marca branca (white-label) da instalação | Vazio = marca padrão do produto (nunca edite código pra isso — some no update) |
| `META_*` (7 vars) | Canal oficial WhatsApp Cloud API | Vazio = canal Meta simplesmente não configurado; WAHA segue funcionando |

Regras do repo: variável nova entra em `.env.example` **e** em `lib/env.ts`; segredo nunca em query string; `.env*` nunca commitado.

---

## 9. O que falta entre "rodando no meu PC" e "produção pra clientes pagantes"

Baseado em `docs/DEPLOY-CHECKLIST.md`, `docs/runbooks/deploy.md`, `hostgator-setup-kit/install.sh` e `docker-compose.prod.yml`:

### Infra

1. **VPS** (4 GB+ RAM recomendados) com Docker — o instalador pergunta e instala se faltar.
2. **Domínio com registro A** apontando pro IP da VPS + HTTPS automático (Caddy do compose, ou Traefik próprio detectado pelo instalador).
3. **Banco Supabase** (conta própria) — com as extensões `vector`, `citext`, `pg_trgm` criadas ANTES do baseline.
4. **Stack inteira via containers**: `install.sh` sobe os 7 serviços com imagens publicadas, healthcheck real (`/api/v1/health` respondendo `"status":"ok"`) e mem_limit por serviço.

### Configuração e dados

5. **Segredos gerados pelo instalador** (HMAC, encryption keys) — você não inventa senha nenhuma; `.env` com chmod 600.
6. **Bootstrap do primeiro admin** (`bootstrap-owner.ts` embutido no install).
7. **Crons instalados**: drain do event_log a cada minuto + agente de atualização — sem eles, automações QUANDO/SE/ENTÃO nascem mas nunca rodam.
8. **WhatsApp conectado** por QR no onboarding (ou canal oficial Meta com templates aprovados).

### Operação contínua

9. **Backups**: plano grátis do Supabase NÃO faz backup sozinho — agendar `backup.sh` (banco + sessões WhatsApp) diariamente; `update.sh` já roda backup antes de cada atualização.
10. **Atualização versionada**: alvo é sempre tag (`v1.2.3`), nunca topo da main; rollback automático se versão nova subir quebrada; `restore.sh` para desastre.
11. **Provas pós-deploy** (do checklist): domínio respondendo **307** (não 404 — pegaria o caso do `-f` esquecido), `/health` reportando a versão certa, smoke test PELA TELA (curl não conta como prova de UX).
12. **LGPD**: quem hospeda é o **controlador** dos dados — preencher razão social/DPO da organização, saber que Sentry é opt-in e higienizado.
13. **Anti-banimento respeitado** (janela horária, teto diário, warm-up) — número novo disparando em massa = banimento.

### O que NÃO muda do dev pro prod

- Banco continua sendo Supabase (não há Postgres no compose).
- O código é o mesmo binário (imagem genérica, config injetada em runtime).
- A diferença essencial: no prod **tudo tem dono de processo permanente** (containers com restart policy + scheduler) e **prova de saúde automatizada**, enquanto no dev você segura o processo na mão.

---

*Documento gerado por exploração assistida do repositório; números de linha podem deslocar conforme o código evolui. Doutrina canônica: `CLAUDE.md`, `ARCHITECTURE.md`, `docs/specs/07-spec-events-workers.md`, `docs/threat-model.md`.*



