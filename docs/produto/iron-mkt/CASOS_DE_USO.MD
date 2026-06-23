# Casos de uso — Iron MKT

> **Produto:** Iron MKT · **Repo:** `iron-mkt` · **Glossário:** [[CONTEXT]]  
> **Leitura obrigatória** antes de alterar código no repositório `iron-mkt`.  
> **Actualizado:** 2026-06-23 — reflecte monorepo com API NestJS, auth JWT e sync Chatwoot.

Iron MKT é uma plataforma **SaaS única** (multi-tenant na mesma instância) para disparo de campanhas por WhatsApp, com integração **EvolutionAPI** e sincronização de contatos com **Chatwoot**.

---

## Atores

| Ator | Papel |
|------|--------|
| **Operador** | Usuário autenticado (JWT) que gere organizações, integrações, listas, campanhas e consulta contatos (leitura) e métricas. |
| **EvolutionAPI** | Envia mensagens WhatsApp; expõe status de entrega quando disponível. |
| **Chatwoot** | Fonte de verdade dos contatos; inbox de atendimento; sync para espelho local. |
| **Sistema** | API NestJS + PostgreSQL, fila BullMQ para disparos, worker de sync Chatwoot, isolamento por `organizationId`. |

---

## Entidades principais

- **Organização** — tenant; operador pode pertencer a várias e trocar contexto activo (JWT `organizationId`).
- **Contato** — espelho local do Chatwoot: nome, telefone, e-mail, tags, status (`ativo`, `inativo`, `bloqueado`); **não editável** na UI Iron MKT.
- **Lista de disparo** — conjunto nomeado de contatos (`contactIds` do espelho local).
- **Campanha** — nome, listas alvo, integração Evolution, template de mensagem, status (`rascunho`, `agendada`, `enviando`, `concluida`), métricas (`enviadas`, `entregues`, `respostas`, `total`).
- **Log** — evento por contato/campanha (`enviado`, `entregue`, `resposta`, `falha`) com timestamp; consultável em `/logs`.
- **Integração** — `chatwoot` ou `evolution`; URL base, API key (cifrada), status (`conectado`, `desconectado`, `erro`). Colaboração com agentes de IA é feita no **Chatwoot**, não no Iron MKT (ver ADR-0005 em `iron-mkt/docs/adr/`).

---

## Fluxo principal (caminho feliz) — produção actual

Estado actual: web Next.js (`apps/web`) ligada à API NestJS (`apps/api`); persistência PostgreSQL; disparo via fila BullMQ (worker com stub Evolution).

1. Operador **regista-se** em `/registro` ou **autentica-se** em `/login` → JWT em `localStorage`; guard client-side redirecciona para `/login` sem token.
2. Operador configura **integrações** em `/integrations`:
   - **Chatwoot** — necessária para sync de contatos.
   - **Evolution** — necessária para disparo de campanhas.
3. Operador **sincroniza contatos** em `/contacts` (`POST /contatos/sincronizar`) — espelho local a partir do Chatwoot; UI **somente leitura** (sem CRUD manual).
4. Operador cria **listas de disparo** em `/lists`, associando contatos do espelho (`PATCH` com `contactIds`).
5. Operador cria **campanha** em rascunho em `/campaigns`: selecciona listas, integração Evolution, define template.
6. Operador **dispara** campanha → `POST /campanhas/:id/disparar` enfileira job BullMQ → worker processa (stub ou Evolution real).
7. UI faz **polling** de métricas enquanto `status === "enviando"`; **logs** visíveis em `/logs` e dashboard.
8. Campanha conclui → status `concluida`; operador consulta métricas e logs.

**Multi-org:** operador pode criar segunda organização em `/organizacoes` e trocar contexto activo no header (`POST /auth/trocar-organizacao`); stores recarregam dados da org activa.

---

## Fluxos em backlog (próximas entregas)

Documentados em `iron-mkt/TODO.MD`; detalhar em planos em `iron-mkt/docs/plans/` quando aprovados.

1. **Integração real EvolutionAPI** — substituir stub do worker por envio WhatsApp efectivo.
2. **Testes E2E Playwright e smoke** — ver [[PLANO_TESTES_API_E2E]] (T2–T5 pendentes).
3. Quotas / limites por tenant *(futuro)*.

---

## Fluxos de exceção

### Integrações

| Situação | Comportamento esperado | Notas |
|----------|------------------------|-------|
| EvolutionAPI offline ou timeout | Campanha em `enviando` **pausa**; operador vê alerta; retentativas automáticas com backoff *(a definir: N tentativas)* | Discutir se cancela ou retoma manualmente |
| API key Evolution inválida | Integração passa a `erro`; **bloqueia novos disparos** até correcção | Não enviar mensagens sem credencial válida |
| Chatwoot sync falha | Disparo **pode continuar** se Evolution estiver OK; sync em background com retry | ADR-0002: sync não é pré-requisito do envio |
| Chatwoot offline prolongado | Fila de sync; dashboard mostra estado degradado | `GET /contatos/sync-estado` expõe último sync |
| Sync sem integração Chatwoot | `POST /contatos/sincronizar` retorna **400** | Configurar integração Chatwoot primeiro |

### Campanha e contatos

| Situação | Comportamento esperado | Notas |
|----------|------------------------|-------|
| Telefone inválido ou contato `bloqueado` | **Pula** contato; regista log `falha` com motivo | Não conta em `enviadas` |
| Rate limit WhatsApp | Fila **desacelera**; UI informa atraso; logs com código de erro | BullMQ activo |
| Campanha interrompida a meio | Status permanece `enviando` ou novo estado `pausada` *(a definir)* | Retomar vs cancelar — pendente |
| Lista vazia ou sem contatos activos | Validação ao criar/disparar; **impede** envio com mensagem clara | — |
| Template vazio | Validação no formulário; não permite disparo | — |
| Contato removido do Chatwoot | Próximo sync actualiza espelho; listas podem referenciar IDs obsoletos *(a tratar)* | — |

### Plataforma (SaaS único)

| Situação | Comportamento esperado | Notas |
|----------|------------------------|-------|
| Sessão expirada / token inválido | Redirecciona para `/login`; operações em curso na fila **continuam** no servidor | Guard client-side em `(app)/layout` |
| Acesso a recurso de outra org | API retorna **403/404**; testes em `tenant-isolation.e2e-spec.ts` | JWT `organizationId` |
| Tenant sem quota / limite | Bloqueia novo disparo; mensagem de upgrade *(futuro)* | — |
| Erro interno (5xx) | Toast/alerta genérico; incidente registado em logs do sistema | Post-mortem em `iron-mkt/docs/incidents/` |

---

## Ver também

- [[iron-mkt/docs/index|Índice do repo iron-mkt]]
- [[PLANO_UI_API_INTEGRACAO]] — integração web ↔ API (concluído)
- [[PLANO_SYNC_CHATWOOT]] — espelho de contatos (concluído)
- [[PLANO_TESTES_API_E2E]] — testes API e E2E (activo)
- `iron-mkt/TODO.MD` — backlog técnico
- `iron-mkt/CONTEXT.md` — glossário local do produto
