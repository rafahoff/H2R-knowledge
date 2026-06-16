# Onboarding — agentes de IA (Cursor)

> **Vault:** [[index]] · **Tipo:** how-to · **Data:** 2026-06-15  
> **Escopo:** alinhar o time H2R ao mesmo comportamento de agentes no Cursor.

---

## 1. Visão geral

O ecossistema H2R separa **conhecimento** de **código**:

| Camada | Onde | Versionado |
|--------|------|------------|
| **Empresa / produto** | `H2R-knowledge/docs/` | Repo `H2R-knowledge` |
| **Repo (iron-mkt, …)** | `iron-mkt/docs/`, etc. | Cada repo de código |
| **Instruções para agentes** | `.cursor/`, `.cursorrules`, manifestos | Dentro de cada repo |

**Objetivos:**

1. Todo colega abre os mesmos repositórios e o mesmo vault Obsidian.
2. Agentes **leem** documentação na ordem certa **antes** de alterar código.
3. Agentes **atualizam** documentação (planos, napkin, índices) no mesmo PR quando aplicável.

---

## 2. Estrutura de pastas (obrigatória)

Clonar todos os repos sob a **mesma pasta pai**:

```text
C:\Github\H2R\                    ← vault Obsidian (abrir ESTA pasta)
├── H2R-knowledge\                ← este repo
├── iron-mkt\                     ← Iron MKT (Next.js)
├── H2R.code-workspace            ← workspace multi-root (Cursor)
└── .obsidian\                    ← config do vault (opcional; sincronizar com o time)
```

**Clone exemplo (PowerShell):**

```powershell
mkdir C:\Github\H2R -Force
cd C:\Github\H2R
git clone <url-H2R-knowledge> H2R-knowledge
git clone <url-iron-mkt> iron-mkt
```

---

## 3. Artefactos do Cursor

| Artefacto | Ficheiro / pasta | Função |
|-----------|------------------|--------|
| **Manifesto** | `.cursor/knowledge.manifest.md` | Ordem de leitura: empresa → repo → notas linkadas |
| **Rules** | `.cursor/rules/*.mdc` | Regras always-on (idioma, casos de uso, planos) |
| **Skills** | `.cursor/skills/*/SKILL.md` | Procedimentos (napkin, testes, arquitectura) |
| **Skills globais** | `~/.cursor/skills/` | Partilhadas — ex.: `obsidian`, `bootstrap-template-projeto` |
| **Napkin** | `.cursor/napkin.md` | Runbook curado (comandos que funcionam) |
| **Commits** | `.cursorrules` | Mensagens em português (`feat(escopo): …`) |

---

## 4. Ordem de leitura ANTES de alterar código

```text
1. .cursor/knowledge.manifest.md     (repo em que está a trabalhar)
2. H2R-knowledge/docs/index.md       (se tarefa envolve produto/regra de negócio)
3. <repo>/docs/index.md              (planos, how-tos, incidentes locais)
4. Apenas notas [[linkadas]] relevantes — NUNCA varrer docs/ inteiro
5. H2R-knowledge/docs/produto/iron-mkt/CASOS_DE_USO.MD   (OBRIGATÓRIO no iron-mkt)
6. .cursor/napkin.md                 (runbook operacional)
```

### Teste de verificação (novo colega)

Abrir o repo `iron-mkt` e perguntar ao agente:

> *Qual a ordem de leitura de documentação antes de alterar código?*

Resposta esperada: manifesto → índice empresa (se aplicável) → `docs/index.md` → notas linkadas → **[[CASOS_DE_USO]]** → napkin.

---

## 5. Skills

### Globais (instalar uma vez por máquina)

```powershell
cd $env:USERPROFILE\.cursor
git clone https://github.com/rafahoff/skills.git skills
# Atualizações: cd skills && git pull
```

| Skill global | Uso |
|--------------|-----|
| **`obsidian`** | Wiki links, índices, taxonomia em `docs/` |
| `bootstrap-template-projeto` | Novos repos H2R |

### No repositório `iron-mkt`

| Skill | Invocação | Função |
|-------|-----------|--------|
| **`napkin`** | Automática | Curadoria de `.cursor/napkin.md` |
| **`nextjs-architecting-apps`** | Tarefas de arquitectura | DDD + App Router |
| **`nextjs-testing-apps`** | Testes | Vitest + Playwright |
| `webapp-testing` | **Explícita** (E2E) | Playwright em `e2e/` |
| `insecure-defaults` | **Explícita** | Auditoria de segredos e API keys |

### No repositório `H2R-knowledge`

Rule always-on: `.cursor/rules/h2r-knowledge.mdc` — sem skills de código.

---

## 6. O que o agente deve actualizar

| Evento | Ficheiros |
|--------|-----------|
| Plano aprovado a partir de brainstorm | `iron-mkt/docs/plans/PLANO_….md` + entrada em `docs/index.md` |
| Plano concluído | Mover para `docs/plans/archive/` + índice |
| Aprendizado operacional | `.cursor/napkin.md` |
| Nova nota produto / brainstorm | `H2R-knowledge/docs/` + [[index]] |
| Caso de uso alterado | [[CASOS_DE_USO]] |

**`TODO.MD`:** o agente **não** edita sem pedido explícito do usuário.

---

## 7. Configuração Cursor

1. Instalar [Cursor](https://cursor.com).
2. Clonar repos (§2) e skills globais (§5).
3. **File → Open Workspace from File** → `H2R.code-workspace`.
4. Confirmar **Rules** activas: Settings → Rules (`iron-mkt`, `h2r-knowledge`, etc.).
5. **Obsidian:** abrir `C:\Github\H2R` como vault.

---

## 8. Checklist — novo colega

- [ ] Clonar `H2R-knowledge` e `iron-mkt` sob `C:\Github\H2R\`
- [ ] Abrir `H2R.code-workspace`
- [ ] Skills globais em `%USERPROFILE%\.cursor\skills`
- [ ] Obsidian: vault pai `C:\Github\H2R`
- [ ] Ler [[CONTEXT]] e [[CASOS_DE_USO]]
- [ ] Ler `.cursor/napkin.md` do repo de trabalho
- [ ] Teste §4 com o agente

---

## 9. Referências

- Taxonomia: [[README]]
- Índice iron-mkt: `iron-mkt/docs/index.md`
- Manifesto exemplo: `iron-mkt/.cursor/knowledge.manifest.md`
