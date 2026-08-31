# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

This repository currently contains no application code — only the skill set
under `.claude/skills/` and the PRDs under `docs/prd/` that define how the
"Prepara Portugal" ecosystem is meant to be built. There is no build tool,
package manager, linter, or test runner configured yet, and no `git`
repository has been initialized. When code starts landing here, update this
file with the real commands (install, run, lint, test — including how to run
a single test) instead of guessing at them.

## Product requirements (`docs/prd/`)

Start with `docs/prd/PRD-00-mestre.md` — it is the index: project goal
(~1,000 students across ~200 fictitious companies doing real administrative
work for each other), the layered architecture, the skill→PRD map with build
status, and the non-negotiable cross-cutting principles (strict dependency
order write→migrate to Supabase→test with real SQL→package; all filtering
server-side; money always `bigint`; Realtime tables need an explicit `alter
publication supabase_realtime add table ...`; `pgcrypto` lives in the
`extensions` schema so `digest()`-using functions need
`set search_path = public, extensions`; `id_resolver` returns a single
`jsonb` accessed via `v_resolve->'dados'`, never in `FROM`). Individual PRDs:

- `PRD-01-fundacao-identidade-assinatura.md` — `pp-base`, `pp-identidade`, `pp-assinatura`
- `PRD-02-criacao-emprego.md` — `pp-criar-empresa`, `pp-emprego`
- `PRD-03-estado-e-dinheiro.md` — `pp-banco`, `pp-orgaos`, `pp-utilities`
- `PRD-04-clientes-externos.md` — `pp-clientes`
- `PRD-05-comunicacao.md` — `pp-correio`, `pp-mensagens`
- `PRD-06-identidade-visual.md` — per-company visual identity (Figma skills)
- `PRD-07-validacao-ia.md` — AI-based document validation via GitHub Actions
- `PRD-08-portal-auditoria-cidade.md` — single portal, teacher/audit dashboard, clickable city map (not yet built)

## What this project is

Prepara Portugal is an ecosystem of fictitious companies used for training,
built on **Supabase (Postgres + Auth + Realtime) + GitHub Pages**, with a
plain **HTML + JS** frontend (no framework, no build step). Data and document
language is PT-PT/PT-BR.

Supabase project: `projetoempresaficticia` (ref `moxxbehwylcjaqjacmyh`, region
`eu-west-1`).

Almost everything about how to build in this repo lives in the skills, not in
code yet — always consult the relevant `pp-*` skill in `.claude/skills/`
before writing anything for this ecosystem; `pp-base` is the foundation every
other `pp-*` skill depends on and should be read first.

## Ecosystem architecture (from pp-base)

- **Single API port**: all business logic goes through Postgres RPC functions
  (`sb.rpc(nome, args)`), never scattered `INSERT`/`UPDATE` calls from the
  frontend. Every RPC returns a uniform shape: `{ ok: true, dados }` or
  `{ ok: false, erro }` — raw exceptions must never reach the browser. Simple
  reads may go through `sb.from(...).select()` guarded by RLS; writes with
  business rules always go through RPC.
- **RLS everywhere**: Row Level Security is enabled on every table in
  `public`, no exceptions. Hiding a button is not protection — authorization
  is enforced server-side. Run the Supabase security advisor after every
  schema change.
- **Auth**: login is Supabase Auth (email + password). The identity record
  (the "Carteirinha", from `pp-identidade`) is a profile row pointing at
  `auth.users.id` — login identity and business identity are separate layers.
- **Explicit state machines**: anything with states (submissions, requests,
  payments) declares a `TRANSICOES` map and rejects any transition not in it.
  State changes happen only via RPC, which validates the transition and
  writes an audit entry.
- **Audit trail from day one**: a single `auditoria` table records who
  (`auth.uid()`), when, which table/record/field, and old → new value for
  every change.
- **Data rules that don't bend**:
  - Text that only looks like a date (competência `"2026-08"`, NIF, IBAN,
    references) is stored as `text`, never `date`.
  - Idempotent writes: functions that create data either use a unique
    key/`on conflict` or refuse to run if the target already exists — no
    duplicate records from a double-click.
  - Money (Prepacoin, P$) is always stored as integer cents (`bigint`),
    never floating point; formatting to P$ happens only at display time.
  - IDs/keys are generated server-side (RPC), never by the client.
- **Frontend**: one `index.html` + `app.js` + `estilos.css` per app. Icons and
  UI components come from the `figma-icons` / `figma-ui-kits` skills rather
  than being designed from scratch. Real-time features (Correio, Mensagens,
  bank balance) use Supabase Realtime channels, not page reloads. The
  `service_role` key must never appear in hosted HTML — only the anon
  (publishable) key; real protection is RLS, not hiding the key.
- **Hosting**: each app is a folder in the repo, served statically by GitHub
  Pages; a root `index.html` acts as the portal linking every app. Real-time
  behavior depends on Supabase, not on Pages' publish latency.

## Skills ecosystem (`.claude/skills/`)

Every `pp-*` skill declares a `Depende de: ...` line at its top making its
dependencies explicit. The golden rule: every company created in the
ecosystem must be linked to another company's service — operational skills
must respect this interdependency.

Build/dependency order:

```
1. pp-base          — foundation (Supabase, RLS, audit, RPC, GitHub Pages)
2. pp-identidade    — Carteirinha: people & companies, cédula PP-/EP-
3. pp-assinatura    — digital signature with slots (party+role+affiliation)
4. pp-banco         — Prepacoin: accounts, IBAN PT50, transfers, bankruptcy
5. pp-orgaos        — AT, Segurança Social, Cartório, Diário da República
6. pp-correio       — internal mail between people (real-time)
7. pp-mensagens     — chat with fictitious phone number and groups
8. pp-emprego       — job portal (postings, applications, CV in Storage)
9. pp-utilities     — recurring billing (water/energy/internet/telecom/rent)
10. pp-criar-empresa — "day zero" orchestrator: assembles a full company + catalog
11. pp-clientes      — external customer generator (the revenue faucet)
```

Everything anchors on the **cédula** (`pp-identidade`). Money flows through
the bank (`pp-banco`) — no balance means bankruptcy. Valid documents require a
signature (`pp-assinatura`) and are delivered to the state bodies
(`pp-orgaos`), which validate automatically and fine late submissions.
Utilities bill per cycle and deliver invoices via `pp-correio`. The company
creator (`pp-criar-empresa`) assembles everything in one operation; the
customer generator (`pp-clientes`) injects revenue, releasing money only
against a validly signed invoice.

Design/support skills that feed the frontend and adjacent workflows:
`figma-ui-kits` and `figma-icons` (UI components/icons), `pp-engenharia-dados`
(data engineering conventions for this project), `corretor-pt-pt` (PT-PT
proofreading), `faithful-translation` (PT→EN document translation),
`prepara-deck-builder` (on-brand PowerPoint decks), `whatsapp-teachers`
(short WhatsApp messages to trainers), and `apps-script-portal` (patterns
from the previous Google Apps Script stack — principles already migrated
into `pp-base`).

Full skill listings and cross-references: `.claude/skills/LEIA-ME.md` (pp-*
ecosystem), `.claude/skills/LEIA-ME-figma.md` (Figma skills), and
`.claude/skills/LEIA-ME-outras.md` (other skills).
