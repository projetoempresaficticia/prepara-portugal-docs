# PRD-02 — Criação de Empresa e Portal de Emprego

> **Skills responsáveis:** `pp-criar-empresa` · `pp-emprego`
> **Dependência:** PRD-01 (identidade, assinatura), PRD-03 (banco).

---

## 1. `pp-criar-empresa` — Criador de empresa ("dia zero")

**Objetivo:** orquestrador que monta uma empresa fictícia completa numa só operação, chamando as skills já existentes. *Prioridade máxima do roadmap: nada flui sem isto.*

**Escopo (uma empresa de cada vez):**
- Cria a **identidade** da empresa (`EP-…`) e dos **sócios** (`PP-…`) — via `pp-identidade`.
- **Vincula** as pessoas à empresa.
- Abre **conta bancária** à empresa (com **fundo inicial**) e a cada sócio — via `pp-banco`.
- Gera **número de telemóvel fictício** de cada sócio — para `pp-mensagens`.
- Cria o **catálogo de produtos** (preço de venda + custo de insumo) — alimenta `pp-clientes` e `pp-utilities`.
- **Inventa o conteúdo em falta** (nome, setor, produtos) coerente com o setor.

**Fora do escopo:** não encadeia contrato/registo no Estado automaticamente — isso é feito à mão depois.

## 2. `pp-emprego` — Portal de Emprego (ex-Recruta)

**Objetivo:** site de vagas do ecossistema; alimenta o ciclo de contratação.

**Escopo:**
- Empresas publicam **vagas**; pessoas **candidatam-se** com **CV em PDF** (Supabase Storage).
- Ligado às **cédulas** (empresa e candidato já existem na identidade).
- Fluxo: **vaga → candidatura → seleção**.

**Fora do escopo:** NÃO encadeia contrato/conta/registo automaticamente — passo manual posterior.

**Migração:** de Google Apps Script + Sheets → Supabase + GitHub. (Protótipo antigo "Recruta" ainda referenciado.)

## 3. Critérios de aceitação

- `pp-criar-empresa` produz, numa operação, empresa + sócios + contas + números + catálogo, tudo resolvível por cédula.
- Uma vaga publicada aparece no portal e aceita candidatura com CV em PDF armazenado.
