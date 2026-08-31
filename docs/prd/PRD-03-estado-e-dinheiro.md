# PRD-03 — Estado e Dinheiro

> **Skills responsáveis:** `pp-banco` · `pp-orgaos` · `pp-utilities`
> **Dependência:** PRD-01. É o motor que gera obrigação e circulação de dinheiro sem gestão humana.

---

## 1. `pp-banco` — Banco Prepacoin

**Objetivo:** base de tudo o que envolve dinheiro (salários, impostos, taxas dos órgãos, faturação das utilities).

**Escopo:**
- Cada empresa e cada pessoa tem conta ligada à sua **cédula**.
- **IBAN fictício PT50**; saldo em **Prepacoin (P$)**, sempre `bigint`.
- **Transferências atómicas** com regra **limite → aprovação** (acima do limite exige aprovação).
- **Comprovante** com código de autenticação.
- Marcação de **incumprimento / falência** quando falta saldo.

**Camadas:** Camada 1 (contas, saldo, transferências, extrato, comprovativo, auditoria, dashboard) construída. Camadas 2–3 (boletos, notificações, expiração automática, relatórios) planeadas.

## 2. `pp-orgaos` — Órgãos do Estado

**Objetivo:** onde o ecossistema se autofiscaliza. Órgãos: **AT (Finanças)**, **Segurança Social**, **Cartório/Notário**, **Diário da República**.

**Escopo:**
- Recebem os **documentos obrigatórios** que as empresas entregam (Modelo 22, IVA, TSU, registo, etc.).
- **Validação automática:** assinatura (via `pp-assinatura`) + taxa paga (via `pp-banco`) + campos + prazo.
- Emitem **número de protocolo** (ex.: `AT-2026-000147`) e comprovativo.
- **Carimbo imutável:** ao aprovar grava `aprovado_por`, `aprovado_em` e **hash**; alterar depois → "adulterado".
- **Multa/pendência automática** a quem falha o prazo.
- **Máquina de estados:** `submetido → em_análise → aprovado / rejeitado`.
- **Hook de revisão por IA** (GitHub Actions) para casos ambíguos — ver PRD-07.

**Consequências que dão sentido:** sem licença aprovada a empresa não vende; sem imposto no prazo o banco bloqueia/multa; documento aprovado desbloqueia algo.

## 3. `pp-utilities` — Cobranças recorrentes

**Objetivo:** gerar despesa recorrente sem gestão humana.

**Escopo:**
- Serviços: **Água, Energia, Internet, Telecom** e **Renda** (a parte "conta mensal" da imobiliária).
- A cada ciclo fiscal, cada serviço gera **fatura de valor aleatório (com mínimo)** para cada empresa.
- Entrega a fatura na **caixa de correio** da empresa (via `pp-correio`).
- Quem não paga no prazo leva **multa**.

**Fora do escopo:** o **mapa clicável da cidade** e a **compra de lotes** ficam para skill futura da imobiliária (ver PRD-08).

## 4. Critérios de aceitação

- Transferência acima do limite exige aprovação; abaixo, é atómica e gera comprovante.
- Saldo insuficiente marca incumprimento.
- Documento submetido sem taxa paga ou sem assinatura válida é rejeitado com motivo; aprovado recebe protocolo + hash imutável.
- Ao virar o ciclo, cada empresa recebe faturas de utilities na caixa e multa se não pagar.
