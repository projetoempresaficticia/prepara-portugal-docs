# Checklist — Prepara Portugal

Estado apurado a **5 de setembro de 2026**, medido no Supabase e nos
repositórios, não de memória.

---

## Onde estamos

| | |
|---|---|
| Apps no ar | **5** de 12 |
| Funções na base | **65** |
| Tabelas | **28**, todas com RLS ligada |
| Empresas ativas | **6** |
| Pessoas ativas | **3** |

O ciclo básico já fecha de ponta a ponta: uma empresa pede a certidão ao
Cartório, o Cartório emite o boleto, o Prepacoin cobra, o pagamento marca
a situação como regularizada, e um salário em falta marca incumprimento
que aparece a quem consultar a certidão.

---

## Feito

- [x] **pp-base** — fundação: auditoria, `api()`, porta única, diretório público
- [x] **ClassCard** (pp-identidade) — carteirinha, admin, verificação · identidade azul
- [x] **Subsight** (pp-assinatura) — slots, documento, verificação · identidade laranja
- [x] **Prepacoin** (pp-banco) — 8 ecrãs, faturas, boletos com referência multibanco · identidade lima
- [x] **Cartório Notarial** — certidão permanente · identidade verde
- [x] **Portal das Finanças AT** — biblioteca de design e auditoria de paleta *(a app ainda não existe)*

---

## 1. Dívida técnica — antes de crescer

Cada app nova aumenta o custo destes. Por isso vêm primeiro.

- [ ] **Limpar a fuga de `sqlerrm`** — **26 de 65 funções** devolvem o erro
      cru do Postgres ao browser (`'Falha ao …: ' || sqlerrm`). Põe nomes
      de colunas e de constraints no ecrã de um formando e viola o R1 da
      pp-base. Passagem mecânica, atravessa todos os repos.
- [ ] **Rever quem pode chamar o quê** — o advisor do Supabase assinala
      **47 funções chamáveis por `anon`** e 52 por `authenticated`.
      Algumas são legítimas (verificação pública de comprovativo e de
      certidão); a maioria não devia estar aberta. Ver a nota do
      `pp-base/README.md`.
- [ ] **3 funções com `search_path` mutável** — risco de sequestro de
      esquema; basta `set search_path = public`.
- [ ] **Ligar a proteção de passwords vazadas** no Supabase Auth (um
      interruptor no painel).
- [ ] **Ver os ecrãs num telemóvel a sério** — as regras existem e
      nenhuma página transborda na horizontal, mas o `agent-browser`
      desta máquina não controla o viewport. Nunca foi visto.

> **Não é dívida:** 13 tabelas aparecem com "RLS sem política". São as
> dos apps por construir (correio, mensagens, vagas, pedidos, produtos)
> mais dois contadores internos. RLS ligada sem política nenhuma é
> negar tudo, que é o correto enquanto o app não existe.

---

## 2. Arrumação rápida

Cinco minutos cada, mas ficam a incomodar.

- [ ] **Renomear a pasta local `pp-banco` para `prepacoin`** — o
      repositório já se chama `prepacoin`; a pasta está presa por um
      handle do Windows. Resolve-se fechando o VS Code.
- [ ] **O favorito antigo está partido** —
      `github.io/pp-banco/` dá **404**. O GitHub redireciona o
      repositório mas **não** o Pages. O novo é
      `github.io/prepacoin/`. Em alternativa, criar um repo `pp-banco`
      com uma página de redirecionamento.
- [ ] **Revogar o token do Figma** partilhado no chat — os repos são
      públicos. Figma → Settings → Personal access tokens.
- [ ] **Limpar os dados de teste** no Prepacoin: `FT-2026-000009`
      ("Encomenda a cancelar (teste)") e `FT-2026-000010`
      ("Teste de emissao").

---

## 3. Os órgãos do Estado — 1 de 4 completo

Sem estes, uma empresa regista-se mas não tem onde entregar impostos nem
contribuições. É o que destrava o Utilities e o Criar-empresa.

- [x] **Cartório Notarial** — completo
- [ ] **Portal das Finanças (AT)** — biblioteca feita; falta:
  - [ ] Ecrãs: início, declarações, pagamentos, situação fiscal, e-fatura
  - [ ] SQL próprio (as RPC genéricas já vivem em `pp-orgaos`)
  - [ ] Ecrã de entrada com o fundo já preparado
  - [ ] Consultar os 9 kits do Figma para a anatomia que a maqueta não
        mostra — modais, paginação, tabela vazia *(o rate limit da API
        cortou; 2 de 9 vistos)*
- [ ] **Segurança Social Direta** — repo, identidade, biblioteca, app
- [ ] **Diário da República** — repo, identidade, biblioteca, app

---

## 4. Apps por construir

Pela ordem de dependência do `CLAUDE.md`. Todos têm repositório criado e
vazio.

- [ ] **pp-correio** — correio interno entre pessoas, tempo real
- [ ] **pp-mensagens** — chat com número fictício e grupos
- [ ] **pp-emprego** — vagas, candidaturas, CV no Storage
- [ ] **pp-utilities** — água, energia, internet, telecom, renda por ciclo
- [ ] **pp-criar-empresa** — o orquestrador do "dia zero"
- [ ] **pp-clientes** — o gerador de receita externa
- [ ] **PRD-08** — portal único, painel do professor, mapa da cidade

---

## 5. A escala

O projeto foi desenhado para **~1000 formandos em ~200 empresas**. Hoje
há **3 pessoas e 6 empresas**, quase todas fixtures de teste.

- [ ] **`pp-criar-empresa` é o que resolve isto.** Está em 10.º na ordem
      de dependência, mas é ele que monta uma empresa completa numa
      operação — identidade, sócios, contas, telemóveis e catálogo. Sem
      ele, povoar o ecossistema é trabalho manual vezes duzentos.
- [ ] Decidir se as empresas entram todas de uma vez ou por turmas.

---

## O que eu faria a seguir

Duas ordens defensáveis, conforme o que é mais urgente:

**Se o objetivo é usar com formandos em breve** → acabar o **Portal das
Finanças**, depois **Segurança Social** e **Diário da República**, e só
então `pp-criar-empresa` para povoar. O ciclo fiscal fica fechado e há
trabalho real para dar.

**Se o objetivo é a saúde do sistema** → a **limpeza do `sqlerrm` e das
permissões** primeiro. É a única coisa que fica mais cara quanto mais se
adiar: cada app nova acrescenta funções à lista.

A arrumação da secção 2 pode ir a qualquer momento — são minutos.
