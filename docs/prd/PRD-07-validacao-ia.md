# PRD-07 — Validação de Documentos por IA (CI/CD)

> **Skill responsável:** `pp-orgaos` (camada de aprovação) + **GitHub Actions** + **Claude API**
> **Estado:** esqueleto — `.github/workflows/verificacao.yml` e `scripts/verificar.mjs` existem.

---

## 1. Objetivo

Empresas "entregam" documentos aos órgãos do Estado para aprovação; a validação corre em CI/CD e usa IA apenas onde as regras não bastam.

## 2. Arquitetura híbrida (confirmada, para conter custos)

1. **Camada de regras + cruzamento** (primeira, barata): campos obrigatórios, formato, empresa existe, **taxa paga** (via `pp-banco`), **assinatura válida** (via `pp-assinatura`), prazo cumprido, cruzamento com registos Supabase.
2. **Camada IA** (última, cara): só o que passa nas regras vai à **Claude API** via GitHub Actions, para apanhar conteúdo sem sentido / ambíguo.

## 3. Máquina de estados e carimbo

- Estados: `submetido → em_análise → aprovado / rejeitado`.
- **Protocolo** único por documento.
- Ao aprovar: grava `aprovado_por`, `aprovado_em` e **hash** imutável; alteração posterior → "adulterado".

## 4. Consequências que dão sentido

Sem licença aprovada a empresa não vende; imposto fora do prazo → banco bloqueia/multa; documento aprovado desbloqueia algo (alvará, conta empresarial).

## 5. A construir

- Completar o **pipeline** (`verificacao.yml` + `verificar.mjs`) a partir do esqueleto.
- Ligar o resultado da IA de volta à máquina de estados de `pp-orgaos`.

## 6. Critério de aceitação

- Um documento com regras cumpridas mas conteúdo incoerente é rejeitado pela camada IA; um coerente é aprovado e carimbado.
