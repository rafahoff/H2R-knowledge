# Casos de uso — Iron MKT

> **Produto:** Iron MKT · **Repo:** `iron-mkt` · **Glossário:** [[CONTEXT]]  
> **Leitura obrigatória** antes de alterar código no repositório `iron-mkt`.

Iron MKT é uma plataforma **SaaS única** (multi-tenant na mesma instância) para disparo de campanhas por WhatsApp, com integração **EvolutionAPI** e sincronização de contatos com **Chatwoot**.

---

## Atores

| Ator | Papel |
|------|--------|
| **Operador** | Usuário autenticado que gere contatos, listas, campanhas, integrações e consulta métricas. |
| **EvolutionAPI** | Envia mensagens WhatsApp; expõe status de entrega quando disponível. |
| **Chatwoot** | Inbox de atendimento; fonte ou destino de sincronização de contatos e conversas. |
| **Sistema** | Orquestra filas de disparo, métricas, logs e isolamento entre tenants (fases futuras). |

---

## Entidades principais

- **Contato** — nome, telefone, e-mail, tags, status (`ativo`, `inativo`, `bloqueado`).
- **Lista de disparo** — conjunto nomeado de contatos (`contactIds`).
- **Campanha** — nome, listas alvo, template de mensagem, status (`rascunho`, `agendada`, `enviando`, `concluida`), métricas (`enviadas`, `entregues`, `respostas`, `total`).
- **Log** — evento por contato/campanha (`enviado`, `entregue`, `resposta`, `falha`) com timestamp.
- **Integração** — `chatwoot` ou `evolution`; URL base, API key, status (`conectado`, `desconectado`, `erro`).
- **Agente de IA** — nome, modelo, prompt, flag `ativo` (configuração futura de respostas automatizadas).

---

## Fluxo principal (caminho feliz) — MVP front-end

Estado actual: dados mockados em memória (Zustand); simulação de envio sem API real.

1. Operador autentica-se na plataforma *(login mock; auth real em fase futura)*.
2. Operador gere **contatos** (CRUD, tags, status).
3. Operador cria **listas de disparo** associando contatos.
4. Operador cria **campanha** em rascunho: selecciona listas, define template de mensagem.
5. Operador agenda ou dispara campanha → status passa a `enviando`.
6. Sistema simula envio progressivo; actualiza métricas e gera **logs** por contato.
7. Campanha conclui → status `concluida`; operador consulta **dashboard** e **logs**.

---

## Fluxos futuros (backlog)

Documentados em `iron-mkt/TODO.MD`; detalhar em planos em `iron-mkt/docs/plans/` quando aprovados.

1. Integração real com **EvolutionAPI** (envio WhatsApp).
2. **Sincronização Chatwoot** (inbox / conversas).
3. **Backend NestJS + PostgreSQL** (persistência e API).
4. **Fila BullMQ** para disparos escaláveis.
5. **Autenticação e multi-tenant** (isolamento por cliente na instância SaaS única).

---

## Fluxos de exceção

> **Estado:** rascunho inicial — comportamentos a validar com o time antes da implementação backend.

### Integrações

| Situação | Comportamento esperado (provisório) | Notas |
|----------|-------------------------------------|-------|
| EvolutionAPI offline ou timeout | Campanha em `enviando` **pausa**; operador vê alerta; retentativas automáticas com backoff *(a definir: N tentativas)* | Discutir se cancela ou retoma manualmente |
| API key Evolution inválida | Integração passa a `erro`; **bloqueia novos disparos** até correcção | Não enviar mensagens sem credencial válida |
| Chatwoot sync falha | Disparo **pode continuar** se Evolution estiver OK; sync em background com retry | Definir se sync é pré-requisito do envio |
| Chatwoot offline prolongado | Fila de sync; dashboard mostra estado degradado | — |

### Campanha e contatos

| Situação | Comportamento esperado (provisório) | Notas |
|----------|-------------------------------------|-------|
| Telefone inválido ou contato `bloqueado` | **Pula** contato; regista log `falha` com motivo | Não conta em `enviadas` |
| Rate limit WhatsApp | Fila **desacelera**; UI informa atraso; logs com código de erro | BullMQ na fase futura |
| Campanha interrompida a meio | Status permanece `enviando` ou novo estado `pausada` *(a definir)* | Retomar vs cancelar — pendente |
| Lista vazia ou sem contatos activos | Validação ao criar/disparar; **impede** envio com mensagem clara | — |
| Template vazio | Validação no formulário; não permite disparo | — |

### Plataforma (SaaS único)

| Situação | Comportamento esperado (provisório) | Notas |
|----------|-------------------------------------|-------|
| Sessão expirada | Redirecciona para login; operações em curso na fila **continuam** no servidor | — |
| Tenant sem quota / limite | Bloqueia novo disparo; mensagem de upgrade *(futuro)* | — |
| Erro interno (5xx) | Toast/alerta genérico; incidente registado em logs do sistema | Post-mortem em `iron-mkt/docs/incidents/` |

---

## Ver também

- [[iron-mkt/docs/index|Índice do repo iron-mkt]]
- `iron-mkt/TODO.MD` — backlog técnico
- `iron-mkt/CONTEXT.md` — glossário local do produto
