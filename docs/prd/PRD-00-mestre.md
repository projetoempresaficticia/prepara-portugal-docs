# PRD-00 — Prepara Portugal · Documento-Mestre

> **Produto:** ecossistema de simulação pedagógica para o curso técnico de Auxiliar Administrativo e Secretariado.
> **Norma linguística:** dados e documentos em **PT-BR**; instituições retratadas são **portuguesas**.
> **Stack:** Supabase (Postgres + Auth + Realtime + RLS) para dados · GitHub Pages para HTML estático · GitHub Actions para CI/CD e revisão por IA.
> **Moeda:** Prepacoin (P$), inteiro em `bigint`. **Identidade:** `PP-ANO-SEQ` (pessoas) · `EP-ANO-SEQ` (empresas).
> **Estado:** consolidação de tudo o que já foi conversado e construído. Este PRD é o índice; cada camada tem o seu próprio PRD.

---

## 1. Objetivo

~1.000 alunos organizados em ~200 empresas fictícias executam trabalho administrativo autêntico uns para os outros dentro de uma economia simulada. O sucesso é medido por os alunos terem **sempre** tarefa administrativa real para fazer **sem o docente distribuir tarefas** — porque a saída de cada empresa é a entrada obrigatória de outra.

## 2. Números do projeto

| Item | Valor |
|---|---|
| Alunos | ~1.000 |
| Alunos por empresa | 5 |
| Empresas | ~200 (teto) |
| Infraestrutura | ~20 |
| Operacionais | ~180 |
| Regiões / clusters | ~12 |
| Ciclo fiscal | 2 semanas |
| Rotação de papéis | a cada ciclo |

## 3. Arquitetura em camadas

```
                    PORTAL ÚNICO (links)              ← PRD-08
                            │
   ┌────────────────────────┼────────────────────────┐
   ▼                        ▼                        ▼
FUNDAÇÃO + IDENTIDADE   INFRAESTRUTURA           COMUNICAÇÃO
  - pp-base              - Banco (P$)             - Correio Interno
  - pp-identidade        - AT/SegSoc/Cartório/DR  - App Mensagens
  - pp-assinatura        - Utilities + Renda      (PRD-05)
  (PRD-01)               (PRD-03)
                            ▲
                    ┌───────┴────────┐
   OPERAÇÃO         │  GERADOR DE     │  ← receita externa
   - pp-criar-empresa │  CLIENTES     │
   - pp-emprego       │  EXTERNOS     │
   (PRD-02, PRD-04)  └────────────────┘  (PRD-04)
```

## 4. Mapa de skills → PRD

| Camada | Skill(s) responsável(is) | PRD | Estado |
|---|---|---|---|
| Fundação técnica | `pp-base` | PRD-01 | Construído |
| Identidade / carteirinha | `pp-identidade` | PRD-01 | Construído |
| Assinatura digital | `pp-assinatura` | PRD-01 | Construído |
| Banco Prepacoin | `pp-banco` | PRD-03 | Construído |
| Órgãos do Estado | `pp-orgaos` | PRD-03 | Construído |
| Cobranças recorrentes | `pp-utilities` | PRD-03 | Construído |
| Criador de empresa (dia zero) | `pp-criar-empresa` | PRD-02 | Construído |
| Portal de emprego | `pp-emprego` | PRD-02 | Construído |
| Gerador de clientes externos | `pp-clientes` | PRD-04 | Construído (plugin) |
| Correio interno | `pp-correio` | PRD-05 | Construído |
| App de mensagens | `pp-mensagens` | PRD-05 | Construído |
| Padrões de dados | `pp-engenharia-dados` | transversal | Construído |
| Design / marca por empresa | `figma-ui-kits`, `figma-icons`, `prepara-deck-builder` | PRD-06 | Parcial |
| Revisão de docs por IA (CI/CD) | `pp-orgaos` + GitHub Actions | PRD-07 | Esqueleto |
| Portal único | — (a criar) | PRD-08 | A fazer |
| Painel do docente / auditoria | — (a criar) | PRD-08 | A fazer |
| Mapa clicável / imobiliária (lotes) | — (skill futura) | PRD-08 | A fazer |

## 5. Índice de PRD

- **PRD-01 — Fundação, Identidade e Assinatura** (`pp-base`, `pp-identidade`, `pp-assinatura`)
- **PRD-02 — Criação e Emprego** (`pp-criar-empresa`, `pp-emprego`)
- **PRD-03 — Estado e Dinheiro** (`pp-banco`, `pp-orgaos`, `pp-utilities`)
- **PRD-04 — Gerador de Clientes Externos** (`pp-clientes`)
- **PRD-05 — Comunicação** (`pp-correio`, `pp-mensagens`)
- **PRD-06 — Identidade Visual por Empresa** (Figma skills)
- **PRD-07 — Validação de Documentos por IA** (`pp-orgaos` + GitHub Actions)
- **PRD-08 — Portal, Auditoria e Cidade** (a construir)

## 6. Princípios transversais (não negociáveis)

- Ordem de dependência estrita: escrever → migrar para Supabase → testar com SQL real → empacotar.
- Toda a filtragem no **backend**; esconder no frontend não é segurança.
- Valores monetários sempre `bigint`; nunca `float`.
- Realtime exige `alter publication supabase_realtime add table ...` explícito.
- `pgcrypto` vive no schema `extensions` → funções com `digest()` declaram `set search_path = public, extensions`.
- `Supabase:execute_sql` multi-statement devolve só o último resultado → verificar intermédios em chamadas separadas.
- `id_resolver` devolve `jsonb` único → aceder por `v_resolve->'dados'`, nunca em `FROM`.
- Entrega em camadas testáveis (Camada 1 → 2 → 3), nunca dump único.
- Decisões fechadas por escolha múltipla estruturada **antes** de escrever código.
- Projeto Supabase: ID `moxxbehwylcjaqjacmyh`, org `projetoempresaficticia`, região `eu-west-1`, Postgres 17.

## 7. Decisões em aberto

- **`pp-mensagens`**: formato do número fictício (9XX XXX XXX), chat vs. mensagens avulsas, suporte a grupos — resolver e fechar.
- **Marca por empresa (Figma)**: falta o link do ficheiro Figma e a fonte de descrição das empresas.
- **Mapa clicável / lotes**: skill de imobiliária ainda por desenhar (só a parte "renda" está em `pp-utilities`).
- **Cursos presenciais** (barbearia, estética, gestão de restauração): reposicionados como **clientes externos**, não como empresas ativas.
