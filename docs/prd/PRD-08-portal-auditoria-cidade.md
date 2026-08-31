# PRD-08 — Portal Único, Auditoria e Cidade

> **Skills responsáveis:** ainda **por criar** (dependem de `pp-base` e da identidade).
> **Estado:** a fazer — itens no horizonte do roadmap.

---

## 1. Portal Único

**Objetivo:** uma página que reúne os links de todos os serviços (banco, correio, emprego, órgãos, mensagens) para o aluno não se perder entre abas.
**Escopo:** HTML estático no GitHub Pages, com resolução de identidade para mostrar só o que é relevante à pessoa/empresa autenticada.

## 2. Painel do Docente / Auditoria

**Objetivo:** o docente atua como **auditor por amostragem**, não gerente.
**Escopo:**
- Agrega **saldos**, **pedidos cumpridos** e **rastro individual** por **região** (12 clusters).
- **Injetar moeda**, **disparar eventos**, monitorizar atividade.
- Avaliação: coletiva (empresa: saldo, pedidos, não-falência, prazos) + individual (cada documento assinado e registado por pessoa, evitando "boleia").

## 3. Cidade clicável / Imobiliária (lotes)

**Objetivo:** mapa clicável para **compra/arrendamento de lotes** com **diretório de empresas**.
**Escopo:** skill futura da imobiliária; a parte "renda mensal" já vive em `pp-utilities`. Falta a compra de lotes e o mapa.

## 4. "Dia zero" (onboarding global)

Script de arranque que faz nascer todas as empresas juntas (constituir → conta → inscrição no Estado → renda → utilities → seguro). O orquestrador por empresa já existe (`pp-criar-empresa`); falta a sequência global.

## 5. Critérios de aceitação

- Portal lista todos os serviços e respeita a identidade autenticada.
- Painel mostra saldos e rastro por região e permite injetar moeda / disparar evento.
- Mapa permite comprar/arrendar um lote e consultar o diretório de empresas.
