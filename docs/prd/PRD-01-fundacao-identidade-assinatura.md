# PRD-01 — Fundação, Identidade e Assinatura

> **Skills responsáveis:** `pp-base` · `pp-identidade` · `pp-assinatura`
> **Dependência:** base de tudo. Nenhuma outra camada funciona sem esta.

---

## 1. `pp-base` — Fundação técnica

**Objetivo:** definir como todas as apps se ligam ao Supabase e são hospedadas, para que cada skill `pp-*` siga o mesmo padrão.

**Escopo:**
- Ligação ao Supabase e padrão de **porta única de API** (um ponto de entrada por app).
- Autenticação por **Supabase Auth** (email/password).
- Segurança por **RLS** — toda a filtragem no backend.
- **Trilho de auditoria** (quem fez o quê, quando).
- **Máquina de estados** e **idempotência** como padrões reutilizáveis.
- Esqueleto **HTML + JS** alojado no **GitHub Pages**.

**Regras críticas (aprendidas):**
- `pgcrypto` no schema `extensions` → `set search_path = public, extensions` em funções com `digest()`.
- Realtime só funciona após `alter publication supabase_realtime add table ...`.
- Migrações: convenção `pp_[skill]_[proposito]`.

## 2. `pp-identidade` — Carteirinha / cédula

**Objetivo:** serviço de identidade central; âncora de todas as outras apps.

**Escopo:**
- Regista **pessoas** (`PP-ANO-SEQ`) e **empresas** (`EP-ANO-SEQ`) no Supabase.
- Emite a **cédula única** que liga a pessoa a conta bancária, assinatura, contratos e vínculo a empresa.
- **Registo apenas por admin.**
- RPC **`id_resolver`** — lookup de identidade cruzado entre apps.

**Regra crítica:** `id_resolver` devolve `jsonb` único → aceder por `v_resolve->'dados'`, nunca usar em `FROM` com aliases de coluna.

## 3. `pp-assinatura` — Assinatura digital

**Objetivo:** assinatura fictícia mas verificável que valida documentos e contratos.

**Escopo:**
- Modelo baseado em **slots**: cada tipo de documento exige N assinaturas com requisito de **parte + papel + vínculo à empresa certa**.
- Cada assinatura: **código legível `ASS-…`** + **hash SHA-256** do conteúdo.
- Feita por pessoa **autenticada**; valida legitimidade (a pessoa tem de poder assinar por aquela empresa/papel).
- Documento fica **válido** só quando todos os slots exigidos estão preenchidos por quem tem legitimidade.
- Suporta **contratos multi-parte** com validação de vínculo empresa↔pessoa.

**Consumido por:** `pp-orgaos` (confere a assinatura antes de aprovar), `pp-emprego`, `pp-criar-empresa`.

## 4. Critérios de aceitação

- Uma pessoa e uma empresa podem ser registadas e resolvidas por cédula via `id_resolver`.
- Um contrato de teste só fica válido quando todos os slots são assinados por partes legítimas; alterar o conteúdo depois quebra o hash.
- RLS impede um utilizador de ver/alterar dados de outra pessoa a partir do frontend.
