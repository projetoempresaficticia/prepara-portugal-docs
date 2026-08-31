# PRD-04 — Gerador de Clientes Externos

> **Skill responsável:** `pp-clientes` (plugin)
> **Dependência:** PRD-01, PRD-02 (catálogo), PRD-03 (banco), PRD-05 (correio).

---

## 1. Objetivo

A "torneira" que injeta **receita nova** no ecossistema, mantendo atividade **todos os dias** (batida pequena) enquanto o ciclo fiscal é a batida grande (2 semanas).

## 2. Escopo

- A cada ciclo, cria **pedidos** de clientes fictícios que escolhem produtos do **catálogo** de cada empresa.
- Entrega o pedido na **caixa de correio** da empresa (via `pp-correio`).
- Liberta o dinheiro **só quando** a empresa cumpre o pedido com **fatura assinada válida** (verificação por documentos).
- **Distribuição:** **piso proporcional ao custo** de cada empresa **+ bónus por mérito** (agilidade, qualidade, boa gestão rendem procura extra).
- **Estado do pedido:** `enviado → recebido → pago → comprovativo → concluído`.

## 3. Regras de justiça (aprendidas)

- Garantir o **piso mínimo a todas** antes de aplicar o bónus — senão as ricas afogam as pobres.
- A coluna de **estado** é o que distingue "empresa ignorou" de "empresa a processar".
- **Falência** só por má gestão real, nunca por azar do sorteio.

## 4. Cursos presenciais como clientes

Os cursos com forte componente presencial (barbearia, estética, gestão de restauração) **não** sustentam empresas em operação contínua — são reposicionados como **clientes externos** que geram procura, não como empresas ativas.

## 5. Critérios de aceitação

- Todas as empresas recebem pelo menos o piso de pedidos por ciclo.
- Um pedido só paga após fatura assinada válida; o estado transita corretamente.
