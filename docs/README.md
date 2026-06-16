# Vault de Conhecimento — H2R (empresa)

Documentação **compartilhada** entre repositórios dos produtos H2R. Versionada neste repo; integrada ao vault Obsidian na pasta pai `C:\Github\H2R\`.

**Ponto de entrada:** [[index]]

---

## Tipos de nota (camada empresa / produto)

| Tipo | Pasta | Quando usar |
|------|-------|-------------|
| **Índice (MOC)** | `docs/index.md` | Mapa global da empresa e produtos |
| **Produto** | `docs/produto/<nome>/` | Casos de uso, identidade visual por produto |
| **Brainstorm** | `docs/brainstorm/` | Discussão antes de virar plano |
| **Mock** | `docs/mock/` | Homologação e fixtures de teste (documentação) |
| **How-to** | `docs/howto/` | Onboarding, IDE, processo de time |

**Ciclo:** `brainstorm/` → plano no repo de implementação (`iron-mkt/docs/plans/`) → `plans/archive/`.

Planos de execução, how-tos de stack, incidentes e ADRs ficam no **repositório de código** que implementa.

---

## Regras para agentes

1. Ler `[[index]]` antes de varrer outras notas.
2. Wiki links `[[NomeDaNota]]` — nomes únicos em todo o vault pai (`H2R/`).
3. Não duplicar o mesmo nome de arquivo em `H2R-knowledge` e repos de código.

Skill global: `obsidian` · Manifesto por repo de código: `.cursor/knowledge.manifest.md`
