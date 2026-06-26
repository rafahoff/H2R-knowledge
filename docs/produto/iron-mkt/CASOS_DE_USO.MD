# Casos de uso — Iron MKT

> **Produto:** Iron MKT · **Repo:** `iron-mkt` · **Glossário:** [[CONTEXT]]  
> **Leitura obrigatória** antes de alterar código no repositório `iron-mkt`.  
> **Actualizado:** 2026-06-26 — nomenclatura técnica EN **concluída** (API, web, worker, testes); narrativa de negócio em pt-BR.

Iron MKT é uma plataforma **SaaS única** (multi-tenant na mesma instância) para **disparo de campanhas por WhatsApp**, com integração **EvolutionAPI** (envio) e **Chatwoot** (contatos e atendimento). O operador configura integrações, monta listas a partir do espelho de contatos, dispara campanhas e acompanha métricas e logs — **sem editar contatos manualmente** no Iron MKT.

---

## Visão do produto

| O Iron MKT **faz** | O Iron MKT **não faz** |
|--------------------|-------------------------|
| Disparo em massa WhatsApp (Evolution) | Atendimento conversacional (fica no Chatwoot) |
| Espelho de contatos sincronizado do Chatwoot | CRUD manual de contatos na UI ou API pública |
| Listas, campanhas, métricas e logs por organização | Configuração de agentes ou modelos de IA (ADR-0005) |
| Multi-organização por operador (trocar contexto) | Uma instância global de Evolution/Chatwoot para todos os clientes |
| Isolamento de dados por `organizationId` (JWT) | Quotas/limites por tenant *(futuro)* |

**Princípio central:** contatos são **fonte de verdade no Chatwoot**; o Iron MKT mantém espelho local para listas, disparo e logs. Falha de sync Chatwoot **não bloqueia** disparo WhatsApp se Evolution estiver OK (ADR-0002).

---

## Arquitectura (visão operador)

```text
Operador → apps/web (Next.js)
              ↓ JWT
           apps/api (NestJS + PostgreSQL)
              ↓ filas BullMQ
           worker (disparo campanhas + sync Chatwoot)
              ↓                    ↓
         EvolutionAPI          Chatwoot API
```

| Componente | Responsabilidade |
|------------|------------------|
| **Web** | UI autenticada; stores Zustand chamam API; guard client-side (`localStorage` + JWT) |
| **API** | CRUD negócio, auth, sync contatos, enfileirar disparo; credenciais cifradas |
| **Worker** | Processar disparo (`campaign-dispatch`); sync periódico Chatwoot (`chatwoot-contact-sync`, cron default 30 min) |
| **PostgreSQL** | Persistência multi-tenant (`organizationId` em todas as entidades) |
| **Redis** | Filas BullMQ |

Deploy self-hosted (VPS + Docker Compose). Ver ADR-0001 em `iron-mkt/docs/adr/`.

---

## Atores

| Ator | Papel |
|------|--------|
| **Operador** | Usuário autenticado (JWT) que gere organizações, integrações, listas, campanhas; consulta contatos (leitura) e métricas/logs. |
| **EvolutionAPI** | Envia mensagens WhatsApp; expõe status de entrega quando disponível. |
| **Chatwoot** | Fonte de verdade dos contatos; inbox de atendimento e IA no canal de conversação. |
| **Sistema** | API + worker + filas; isolamento por `organizationId`; upsert interno de contatos no sync. |

---

## Entidades principais

- **Organização** — tenant (estabelecimento); operador pode pertencer a várias e trocar contexto activo no JWT (`organizationId`).
- **Utilizador** — e-mail + senha; registo bootstrap cria utilizador e primeira organização.
- **Contato** — espelho local do Chatwoot: `chatwootContactId` (chave de ligação), nome, telefone normalizado, e-mail, tags, status (`ativo`, `inativo`, `bloqueado`); **somente leitura** na UI; API pública sem `POST`/`PATCH`/`DELETE` (escrita só via sync interno).
- **Lista de disparo** — conjunto nomeado de contatos (`contactIds` do espelho local).
- **Campanha** — nome, listas alvo, `preferredIntegrationId` (integração Evolution preferida), `integrationId` (último disparo), template, status (**rascunho** (`draft`), **agendada** (`scheduled`), **aguardando fila** (`queue_waiting`), **enviando** (`sending`), **concluída** (`completed`), **erro** (`error`)), `scheduledAt` opcional, métricas.
- **Log** — evento por contato/campanha (`enviado`, `entregue`, `resposta`, `falha`) com timestamp e snapshot `integrationName`; persistido pelo worker no disparo.
- **Integração** — `chatwoot` ou `evolution`; URL base, API key (cifrada na BD), status (`conectado`, `desconectado`, `erro`). Várias integrações do mesmo provider por org; campanha referencia qual Evolution usar.

---

## Mapa de telas

| Rota | Função |
|------|--------|
| `/login`, `/register` | Autenticação e bootstrap (utilizador + org) |
| `/dashboard` | Resumo: campanhas, listas, integrações, contatos da org activa |
| `/organizations` | Listar, criar e activar organizações |
| `/integrations` | CRUD integrações Chatwoot/Evolution |
| `/contacts` | Listagem leitura + «Sincronizar com Chatwoot» + estado do sync |
| `/lists` | CRUD listas; associar/remover membros (`contactIds`) |
| `/campaigns` | CRUD campanhas; seleccionar Evolution; **Disparar**; polling de métricas |
| `/logs` | Consulta logs com filtro por evento e campanha |

Selector de organização no header; ao trocar org, stores recarregam dados da org activa.

## Contrato API (inglês)

Rotas e JSON da API em inglês (ADR-0007 concluído). UI mantém copy em português; valores técnicos exemplificados:

| Recurso | Rota base |
|---------|-----------|
| Auth | `/api/v1/auth/register`, `login`, `me`, `switch-organization` |
| Organizações | `/api/v1/organizations` |
| Contatos | `/api/v1/contacts`, `.../sync`, `.../sync-status` |
| Listas | `/api/v1/contact-lists` |
| Campanhas | `/api/v1/campaigns`, `.../dispatch`, `.../dispatch-preview` |
| Integrações | `/api/v1/integrations`, `.../test-connection` |

Exemplo de payload:

```json
{
  "status": "draft",
  "integrationStatus": "connected",
  "contactStatus": "active"
}
```

Equivalências de status: ver `iron-mkt/CONTEXT.md` (negócio pt-BR ↔ valor API).

---

## Casos de uso

### UC-01 — Registar-se e autenticar-se

**Actor:** Operador  
**Pré-condição:** Instância Iron MKT disponível.

1. Operador acede `/register`, informa nome, e-mail, senha (≥8), nome da org (slug opcional).
2. Sistema cria utilizador + organização e devolve JWT.
3. Alternativa: `/login` com e-mail e senha → JWT em `localStorage`.
4. Rotas `(app)/*` exigem token; sem token → redirect `/login`.

**Pós-condição:** Operador autenticado com `organizationId` activo no JWT.

---

### UC-02 — Gerir organizações

**Actor:** Operador autenticado.

1. Operador lista organizações via `GET /auth/me`.
2. Operador cria nova org em `/organizations` (`POST /organizations`).
3. Operador troca contexto no header (`POST /auth/switch-organization`).
4. UI recarrega integrações, contatos, listas, campanhas e logs da org activa.

**Notas:** `PATCH`/`DELETE` de organização não são requisito do MVP; listagem cobre o DoD actual.

---

### UC-03 — Configurar integrações

**Actor:** Operador autenticado.

1. Operador acede `/integrations`.
2. Cria integração **Chatwoot** (URL + token) — obrigatória para sync de contatos.
3. Cria integração **Evolution** (URL + API key) — obrigatória para disparo.
4. API key mascarada na listagem; envio de nova key só ao editar.
5. Status **erro** (`error`) bloqueia novos disparos nessa integração até correcção.

**Notas:** «Testar conexão» na UI pode ser simulado até integração real com health-check das APIs externas.

---

### UC-04 — Sincronizar contatos (Chatwoot → espelho local)

**Actor:** Operador ou Sistema (cron).

**Pré-condição:** Integração Chatwoot configurada (**conectado** (`connected`) ou **erro** (`error`) recuperável).

1. **Manual:** operador em `/contacts` clica «Sincronizar com Chatwoot» → `POST /contacts/sync`.
2. **Automático:** worker executa job `chatwoot-contact-sync` (cron configurável, default cada 30 min).
3. Sistema pagina contactos Chatwoot e faz upsert idempotente por `(organizationId, chatwootContactId)`; telefone normalizado antes de gravar.
4. UI mostra estado via `GET /contacts/sync-status` (`never` | `in_progress` | `ok` | `error`).

**Pós-condição:** Espelho local actualizado; UI **não** permite criar, editar ou importar CSV de contatos.

**Regra:** API pública não expõe escrita em contatos; operador edita contactos no Chatwoot.

---

### UC-05 — Gerir listas de disparo

**Actor:** Operador autenticado.

1. Operador cria lista em `/lists` com nome e descrição.
2. Associa contatos do espelho (`PATCH` com array completo de `contactIds`).
3. Pode editar nome/descrição ou remover lista.
4. Listas vazias ou sem contactos activos impedem disparo com mensagem clara.

---

### UC-06 — Criar e disparar campanha

**Actor:** Operador autenticado.

**Pré-condição:** Pelo menos uma integração Evolution com `instanceName`; listas com contactos; template ou anexo.

1. Operador cria campanha em `/campaigns` (wizard): **Básico** → **Mensagem** → **Audiência** → **Integração** (`preferredIntegrationId`). Com uma única integração pronta, o passo Integração é omitido e preenchido automaticamente.
2. Operador pode definir `scheduledAt` e status **agendada** (`scheduled`) para disparo automático.
3. Disparo manual: **Disparar** → `POST /campaigns/:id/dispatch` com `integrationId` (pré-selecção: último disparo ou preferida).
4. Disparo automático: worker `CampaignScheduleScheduler` (cron `CAMPAIGN_SCHEDULE_CRON`, padrão cada minuto) processa campanhas `scheduled` com `scheduledAt <= now`.
5. Worker enfileira fila `campaign-dispatch`; UI faz polling (~3s) enquanto `status === "sending"`.
6. Worker regista logs (`sent` / `failure`) com `integrationId` e `integrationName`.
7. Campanha passa a `completed` quando processamento termina; integração ocupada → `queue_waiting` (ADR-0006).

**Pós-condição:** Métricas actualizadas; logs com coluna integração em `/logs` e dashboard.

---

### UC-08 — Disparo agendado automático

**Actor:** Sistema (scheduler worker).

**Pré-condição:** Campanha com `status === scheduled`, `scheduledAt` preenchido e `preferredIntegrationId` válido.

1. A cada tick (`CAMPAIGN_SCHEDULE_CRON`), worker lista campanhas elegíveis (`scheduledAt <= now`).
2. Claim transacional (`UPDATE … WHERE status = scheduled`) evita duplo disparo.
3. Valida `preferredIntegrationId` (Evolution, `instanceName`, não `error`).
4. Reutiliza lógica de `dispatch` — enfileira `campaign-dispatch` ou `queue_waiting`.
5. Falha dura (integração inválida): `status = error`, `dispatchError` preenchido; **sem** retry automático.
6. Worker parado: campanhas atrasadas disparam no próximo tick após recuperação.

**Pós-condição:** `integrationId` actualizado; logs com integração usada.

---

### UC-07 — Consultar logs e métricas

**Actor:** Operador autenticado.

1. Operador acede `/logs` — lista eventos da org com filtro por `event` e `campaignId`.
2. Dashboard e `/campaigns` mostram métricas agregadas da campanha.
3. Dados isolados por `organizationId`; acesso cross-tenant retorna 403/404.

---

## Fluxo principal (caminho feliz)

1. Registo ou login → JWT.
2. Configurar Chatwoot + Evolution em `/integrations`.
3. Sincronizar contatos em `/contacts`.
4. Criar listas e associar contactos em `/lists`.
5. Criar campanha em `/campaigns` e disparar.
6. Acompanhar progresso, métricas e logs.
7. (Opcional) Criar segunda org e trocar contexto para operar outro estabelecimento.

---

## Backlog (próximas entregas)

Documentado em `iron-mkt/TODO.MD` e planos em `iron-mkt/docs/plans/`.

| Prioridade | Entrega | Notas |
|------------|---------|-------|
| Alta | **Evolution real** no worker | Substituir stub por envio WhatsApp efectivo |
| Média | **Testes T2–T5** | Worker, unit domain, Playwright E2E, smoke script — ver [[PLANO_TESTES_API_E2E]] |
| Média | Webhook Chatwoot | Sync em tempo real além do cron |
| Baixa | Health-check real em «Testar conexão» | Validar credenciais Evolution/Chatwoot |
| Futuro | Quotas / limites por tenant | Bloquear disparo acima do plano |

Planos concluídos (arquivo): [[PLANO_UI_API_INTEGRACAO]], [[PLANO_SYNC_CHATWOOT]], [[PLANO_MONOREPO_DEPLOY]], [[PLANO_NOMENCLATURA_INGLES]], [[PLANO_DISPARO_AGENDADO]].

---

## Fluxos de exceção

### Integrações

| Situação | Comportamento esperado | Notas |
|----------|------------------------|-------|
| EvolutionAPI offline ou timeout | Campanha em `enviando` **pausa**; alerta ao operador; retentativas com backoff *(a definir)* | Cancelar vs retomar manualmente — pendente |
| API key Evolution inválida | Integração `erro`; **bloqueia novos disparos** | — |
| Chatwoot sync falha | Disparo **pode continuar** se Evolution OK; retry em background | ADR-0002 |
| Chatwoot offline prolongado | Fila de sync; UI estado degradado em `/contacts` | `GET /contacts/sync-status` |
| Sync sem integração Chatwoot | `POST /contacts/sync` retorna **400** | Configurar Chatwoot primeiro |

### Campanha e contatos

| Situação | Comportamento esperado | Notas |
|----------|------------------------|-------|
| Telefone inválido ou contato **bloqueado** (`blocked`) | **Pula** contacto; log **falha** (`failure`) com motivo | Não conta em métricas `sent` |
| Rate limit WhatsApp | Fila **desacelera**; UI informa atraso | BullMQ |
| Campanha interrompida a meio | `enviando` ou `pausada` *(a definir)* | Retomar vs cancelar — pendente |
| Lista vazia / sem activos | Validação impede disparo | — |
| Template vazio | Validação no formulário | — |
| Contato removido no Chatwoot | Próximo sync actualiza espelho; IDs obsoletos em listas *(a tratar)* | — |

### Plataforma

| Situação | Comportamento esperado | Notas |
|----------|------------------------|-------|
| Sessão expirada / token inválido | Redirect `/login`; jobs em fila **continuam** no servidor | Guard client-side |
| Recurso de outra org | API **403/404** | `tenant-isolation.e2e-spec.ts` |
| Tenant sem quota | Bloqueia disparo; mensagem upgrade *(futuro)* | — |
| Erro 5xx | Toast genérico; incidente em `iron-mkt/docs/incidents/` | — |

---

## Ver também

- [[iron-mkt/docs/index|Índice do repo iron-mkt]]
- [[PLANO_UI_API_INTEGRACAO]] — integração web ↔ API *(concluído)*
- [[PLANO_SYNC_CHATWOOT]] — espelho de contatos S1–S4 *(concluído)*
- [[PLANO_TESTES_API_E2E]] — testes API e E2E *(activo: T0–T1 feitos)*
- [[PLANO_MONOREPO_DEPLOY]] — monorepo e deploy *(concluído)*
- [[PLANO_NOMENCLATURA_INGLES]] — nomenclatura técnica EN *(concluído)*
- [[PLANO_DISPARO_AGENDADO]] — disparo agendado e scheduler *(concluído)*
- `iron-mkt/TODO.MD` — backlog técnico
- `iron-mkt/CONTEXT.md` — glossário local
- `iron-mkt/docs/adr/0002-multi-tenant-organizacao.md` — isolamento e integrações
- `iron-mkt/docs/adr/0005-escopo-sem-agentes-ia.md` — IA no Chatwoot, não no Iron MKT
