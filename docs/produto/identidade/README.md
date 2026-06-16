# Identidade visual — H2R

Assets de marca H2R para produtos da empresa (Iron MKT e futuros). **Fonte canónica** neste repositório; não duplicar sem necessidade.

## Ficheiros

| Ficheiro | Uso sugerido |
|----------|----------------|
| `Logo Colorido Transparente.png` | Sidebar, header claro, fundos neutros |
| `LogoBrancoVertical_transparente.png` | Fundos escuros / dark mode |
| `Logo-2016-Horizontal-Colorido.jpg` | Materiais horizontais, e-mail |
| `Logo_Escuro_FundoBranco_426x104.jpg` | Documentos, fundo branco fixo |
| `Logo2015.jpg` | Legado — preferir versões mais recentes |
| `Logo-BaixaRes.jpg` | Pré-visualização / placeholders apenas |

## Regras para implementação (Iron MKT)

- Referência da skill: `iron-mkt/.cursor/skills/design-system-h2r/SKILL.md`
- Cores e tokens só em `iron-mkt/src/app/globals.css` (ou ficheiros de tema dedicados).
- **Proibido** hex/rgb/hsl literal em componentes `.tsx`.
- Copiar logos para `iron-mkt/public/brand/` apenas quando integrar na UI (manter nomes descritivos em minúsculas).

## Produtos

| Produto | Tema |
|---------|------|
| Iron MKT | Neutro shadcn até tokens H2R serem extraídos da marca |

Ver [[CASOS_DE_USO]] e `iron-mkt` para contexto do produto.
