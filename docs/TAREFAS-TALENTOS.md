# Tarefas — Talentos (pp-emprego)

> O portal de emprego do ecossistema. Empresas publicam vagas, pessoas
> candidatam-se com CV em PDF, a empresa vê e avalia. A app está na
> `talentos/` (foi renomeada de `pp-emprego`).

**Estado medido em 6 de setembro de 2026.** Identidade visual, camada de
dados e frontend completos e testados. Falta testar num telemóvel a
sério e resolver a integração com `pp-criar-empresa`.

---

## O que está pronto

- [x] Identidade visual (`biblioteca.html`) com a paleta de 8 cores
  corrigida (terracota `#B9433F` em vez de coral como texto de botão;
  borda de campo `#9C867F` em vez de `#E7DEDC`) e 19 ícones.
- [x] `index.html` (entrada) + `web/marca/` + `web/atualizar.js` +
  `ferramentas/versoes.py` — a parte estática que anda de app em app.
- [x] Skill `pp-emprego/SKILL.md` descreve o modelo (2 tabelas, 8 RPCs,
  2 máquinas de estados, RLS por cédula) e tem as duas referências
  (`storage-cv.md` e `rls-emprego.md`).
- [x] `PRD-02` (secção 2) define o âmbito mínimo.
- [x] **Camada de dados completa** — `sql/001` a `004`, aplicada e testada
  com 30 passos de SQL real (as três pessoas, empresa e dois candidatos) +
  security advisor sem erros. Detalhe abaixo.

## Camada de dados — o que ficou

- [x] `sql/001_vagas_e_candidaturas.sql` — CHECK de estado (sem acento:
  `em_analise`, não `em_análise` — segue o precedente já usado pela
  Segurança Social), FK `candidaturas.vaga_id → vagas.id`, colunas
  `justificativa` e `notificacao_id` (aponta para a linha de `correio`
  que a mudança de estado gerou), e as políticas RLS. `vagas` e
  `candidaturas` já existiam na base desde a fundação — só faltava abrir
  a porta pela medida certa.
- [x] `sql/002_storage_curriculos.sql` — bucket `curriculos` (privado,
  10 MB, só PDF), criado por SQL (`insert into storage.buckets`, como em
  todos os outros apps — **não precisa de passo manual na consola**).
  Políticas por caminho (`<candidatoCedula>/<vagaId>.pdf`), sem tabela
  espelho. **Isto substitui e simplifica um rascunho anterior**, que
  criava uma tabela `curriculos_meta` só para a RLS ter onde perguntar
  "de quem é este ficheiro" — desnecessário, porque a política já
  pergunta diretamente ao caminho, tal como o AeroMail
  (`fn_anexo_correio_visivel`) e a AT (`fn_fisco_visivel`) já fazem.
- [x] `sql/003_rpc_vagas.sql` — `vaga_criar`, `vaga_publicar`,
  `vaga_arquivar`, `minhas_vagas` (empresa), `vagas_publicas` (pública,
  chamável por `anon` — é a montra do portal, como a consulta pública de
  um protocolo do Estado).
- [x] `sql/004_rpc_candidaturas.sql` — `emprego_candidatar`,
  `emprego_candidaturas_da_vaga`, `emprego_estado_candidatura`,
  `emprego_minhas_candidaturas`, e a notificação (secção 4, agora
  fechada e implementada).

### Três correções ao desenho original

1. **`emprego_candidatar` não recebe o caminho do CV como parâmetro.** A
   skill previa `emprego_candidatar(vaga_id, cv_url)`. Isso abre uma
   falha: uma função `security definer` corre sem RLS, e se aceitasse
   qualquer caminho, um candidato podia apontar para o CV de **outra
   pessoa** que já tivesse concorrido à mesma vaga, e a função confirmava
   a existência dele na mesma — a candidatura ficava com o CV de outro.
   A função passa a calcular o caminho sozinha, a partir da cédula
   resolvida no servidor e do `vaga_id`.
2. **Sem tabela `curriculos_meta`** — ver acima.
3. **`vaga_publica(uuid)`, RPC nova que a skill não previa** (acrescentada
   já na fase de frontend, `sql/005`). `vagas_publicas()` lista, mas não
   há como o PostgREST embeber o nome da empresa ao ler uma vaga isolada
   por `.from()`: `vagas.empresa_cedula` é texto, sem FK para
   `empresas.cedula` (decisão de propósito de toda a base — cédulas
   seguem-se por convenção, não por FK). Uma RPC resolve num pedido só o
   que senão seriam dois.

### Testado (30 passos, três pessoas: Germano/Padaria Central como
empresa, Rita e Tiago como candidatos)

Cobertos: criar/publicar/arquivar vaga e as transições proibidas; a
montra pública (`vagas_publicas`) visível a `anon`, sem vazar rascunhos;
candidatar-se sem CV enviado (recusado) e depois com CV (aceite);
idempotência (candidatar duas vezes devolve a mesma candidatura);
recusar candidatar-se à própria empresa; isolamento entre candidatos (um
não vê a candidatura do outro, nem o CV do outro, na tabela em bruto);
a máquina de estados completa — salto proibido (`novo→aprovado`
direto), `em_analise` a recusar justificativa, `reprovado` a exigir
justificativa, `aprovado` com justificativa opcional e com texto por
omissão, e `reprovado` como estado terminal (não aceita mais
transições); a notificação a chegar à caixa do candidato com o
remetente e o corpo certos, incluindo o candidato ver a mesma
justificativa via `emprego_minhas_candidaturas`.

Security advisor: 0 avisos de nível ERROR; os avisos WARN que aparecem
são os genéricos "esta função é chamável por `authenticated`/`anon`" —
esperados, porque é essa a porta que as RPC são supostas ser.

---

## O que falta

### 1. Frontend (HTML + JS, sem framework) — CONCLUÍDO em 6 de setembro

> Uma diferença desta app para as anteriores: ficheiros à raiz do
> repositório (`vaga.html`, `empresa.html`, …), não em subpastas
> `web/publico/`, `web/empresa/`, `web/candidato/` como o plano original
> desta lista previa. `ferramentas/versoes.py` só varre `*.html` à raiz
> (`RAIZ.glob('*.html')`, sem recursão) — um layout em subpastas
> quebraria o carimbo de versão. Segue o mesmo padrão da Segurança
> Social.

- [x] `web/biblioteca/ta.css` e `ta.js` — variáveis da paleta corrigida,
      botão terracota, campo com borda `#9C867F`, `esc()`,
      `mostrarMsg()`, `comVersao()`/`versionarLinks()`, `montarTopo()`
      (cabeçalho único partilhado pelas 5 páginas), avatar por cédula,
      selos de estado, `ligarFormularioLogin()`/`perguntar()` reutilizáveis.
- [x] `index.html` — montra pública (`vagas_publicas`), sem login;
      "Entrar" abre uma janela contextual, não um portão de página.
- [x] `vaga.html` — detalhe de uma vaga (`vaga_publica`, nova RPC — ver
      abaixo) + candidatura por CV (`sb.storage...upload` seguido de
      `emprego_candidatar`), com a mesma janela de login contextual: só
      pede sessão no momento de candidatar, nunca para ler a vaga.
- [x] `empresa.html` — `minhas_vagas()`, criar/publicar/arquivar, com
      portão de página inteira (só quem tem sessão E empresa vê o painel).
- [x] `gerir-vaga.html` — `emprego_candidaturas_da_vaga`, "ver CV" com
      `createSignedUrl` (60s), aprovar com justificação opcional,
      reprovar com justificação obrigatória.
- [x] `candidaturas.html` — `emprego_minhas_candidaturas()`, mostra a
      justificativa. Tempo real pela mesma subscrição do AeroMail
      (`correio-<minhaCedula>`): quando a notificação de aprovado/
      reprovado chega, a lista recarrega-se sozinha.
- [x] **Cache-busting** — `python ferramentas/versoes.py` corrido antes
      do commit; versão do site `d53ffbba6cf4`.
- [x] Testado num Chrome a sério (puppeteer-core, headless): as 5
      páginas sem erros de consola; `vaga_publica` confirmada a recusar
      vaga arquivada (limite de segurança); formulário de login exercido
      contra o Supabase real com senha errada; layout responsivo checado
      em emulação de telemóvel (390×844, iOS).
- [ ] **Testar num telemóvel a FÍSICO a sério** — só foi feito em
      emulação; nunca foi visto num aparelho real em nenhum dos oito
      apps anteriores.

### 2. Integração com o resto do ecossistema

- [ ] Confirmar que a `pp-criar-empresa` cria empresa com pelo menos
      um **membro autorizado** com `auth.users.id` (sem isto,
      `fn_minha_empresa_cedula()` não devolve nada e a empresa não
      consegue entrar no painel).

---

## Decisões tomadas nesta camada (para revalidares)

1. **Remetente da notificação: a cédula da empresa (`EP-…`), não uma
   cédula sintética de sistema.** É o único padrão que existe no
   ecossistema até aqui — a AT, a Segurança Social e o Cartório
   notificam sempre com a sua cédula real. Uma "TA-SISTEMA" quebraria o
   modelo em que todo o ator é uma entidade registada de verdade.
2. **O bucket `curriculos` não precisa de passo manual** — criado por
   SQL, como todos os outros.
3. **Texto por omissão sem justificativa** (aprovação sem texto): frase
   própria desta app ("A sua candidatura à vaga X foi aprovada."), sem
   link fixo para uma página que ainda não existe — links assim,
   gravados numa mensagem que depois não se reescreve, ficam partidos
   para sempre se a página mudar de sítio.

## Decisão que continua tua

- **A cor por empresa** — a secção 7 da skill diz que se mantém "agora
  vinda do brandbook/Figma". O brandbook existe? A biblioteca já tem a
  paleta do Talentos, mas a cor por empresa dentro do Talentos ainda
  não está mapeada. Isto só importa quando chegarmos ao frontend.

---

## O que eu faria a seguir

**`web/publico/vagas.html`** — a montra pública é o único ecrã que já
tem tudo o resto pronto atrás dele (`vagas_publicas` testada, paleta
corrigida, ícones prontos) e é o primeiro que qualquer pessoa vê. Depois
o painel da empresa, e o do candidato por último — é a ordem que a
skill também sugere (pública → fechada).
