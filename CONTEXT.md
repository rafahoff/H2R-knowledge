# Glossário — H2R

Linguagem ubíqua da empresa. Sem detalhes de implementação. Repositório canônico do glossário; cada repo de código pode espelhar termos locais ou linkar aqui.

## Empresa

| Termo | Definição |
|-------|-----------|
| **H2R** | Empresa; este vault agrupa conhecimento de todos os produtos H2R. |
| **SaaS único** | Uma instância da plataforma serve múltiplos clientes (tenants) com isolamento de dados — decisão transversal aos produtos web. |

## Produtos

| Termo | Definição |
|-------|-----------|
| **Iron MKT** | Produto de marketing: disparo de campanhas por WhatsApp, integração EvolutionAPI e sincronização com Chatwoot. Repo: `iron-mkt`. |

## Atores transversais (integrações comuns)

| Termo | Definição |
|-------|-----------|
| **Operador** | Usuário autenticado que configura contatos, listas, campanhas e integrações na plataforma. |
| **EvolutionAPI** | Gateway de envio de mensagens WhatsApp. |
| **Chatwoot** | Plataforma de atendimento; inbox e conversas sincronizadas com contatos do Iron MKT. |

## Conceitos Iron MKT (resumo)

Detalhes e fluxos em [[CASOS_DE_USO]] (`docs/produto/iron-mkt/CASOS_DE_USO.MD`).

| Termo | Definição |
|-------|-----------|
| **Contato** | Pessoa com telefone, e-mail, tags e status (`ativo`, `inativo`, `bloqueado`). |
| **Lista de disparo** | Agrupamento nomeado de contatos para campanhas. |
| **Campanha** | Disparo em massa com template de mensagem, listas alvo e métricas de envio. |
| **Log** | Registro de evento por contato/campanha (`enviado`, `entregue`, `resposta`, `falha`). |
| **Integração** | Conexão configurada com `chatwoot` ou `evolution` (URL, API key, status). IA no atendimento via Chatwoot. |
