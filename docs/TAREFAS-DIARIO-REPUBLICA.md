# Tarefas — Diário da República

Decidido a **10 de setembro de 2026**, em conversa com o Germano. Identidade
visual já feita (`biblioteca.html`, publicada em
`projetoempresaficticia/diario-republica`) — o que falta é tudo o resto.

---

## O que a app faz (redefinido — não é o que a `pp-orgaos` previa)

A skill `pp-orgaos` descrevia o Diário como um sítio onde **empresas**
publicam editais e avisos, pelo motor genérico `orgao_submeter`. Foi
substituído por algo com mais valor pedagógico: **o Diário é o canal oficial
por onde a professora publica atividades obrigatórias**, e o app **confere
sozinho** se cada empresa cumpriu — sem a professora ter de visitar Talentos,
AT, Segurança Social e Prepacoin um a um.

O catálogo `orgao_tipos` já tinha uma linha `publicacao` para 'diario' — fica
para trás; não é o desenho que se segue.

### As duas peças que fazem isto funcionar

1. **Aviso automático por Correio.** Ao publicar uma atividade, o Diário
   escreve uma mensagem na caixa de cada empresa visada (mesmo mecanismo já
   usado no Talentos para avisar de uma candidatura decidida). A professora
   publica uma vez; ninguém precisa de abrir o Diário para saber que há
   trabalho novo — já está no Correio, que todos já verificam.

2. **Verificação automática, por tipo de atividade.** Todos os apps deste
   ecossistema partilham a mesma base Postgres — não há fronteira a atravessar.
   Uma função do Diário lê diretamente a tabela do app devido (`talentos.vagas`,
   `submissoes` da AT, `transacoes` do Prepacoin) e diz se a empresa já
   cumpriu, mostrando os campos reais que ela preencheu lá. **Não existe
   verificador genérico** — cada tipo de atividade sabe a sua própria tabela e
   os seus próprios campos, por isso a professora escolhe de um **catálogo
   fechado**, não escreve texto livre à espera que algo a verifique sozinho.

   Uma atividade **sem** tipo do catálogo (`tipo = null`) cai em
   autodeclaração + anexo — para o que não dá para verificar por código (ex.:
   "atender bem os clientes esta semana"). As duas formas convivem; a escolha
   é por atividade.

---

## Decisões tomadas

- **Substitui, não coexiste com, a ideia de empresas publicarem os seus
  próprios editais.** Mantém o âmbito pequeno; pode voltar-se a isso depois,
  se fizer sentido.
- **Alvo de uma atividade: "todas as empresas" ou uma lista de cédulas
  escolhida à mão.** Sem segmentação por setor/região na v1 — a professora
  consegue simular isso escolhendo a lista à mão, e evita depender de um
  campo `setor` que pode não estar consistente em todas as empresas.
- **A professora é identificada por `fn_e_professor()`**, já usado no resto
  do ecossistema (AT, SS, Cartório) — não se inventa um papel novo.
- **Uma atividade publicada não se edita** — mesma regra do
  `hash_carimbo` dos órgãos: se for preciso corrigir, publica-se uma
  retificação nova. (A confirmar com o Germano — pode ser preciso um botão
  "encerrar" para uma atividade deixar de contar sem apagar o histórico.)
- **Catálogo inicial de tipos com verificador automático — só quatro, só em
  apps que já existem a sério** (nada de `pp-clientes`/Pulso: ainda não têm
  tabela que sirva de prova):

  | tipo | app devido | tabela | confere |
  |---|---|---|---|
  | `publicar_vaga` | Talentos | `vagas` | existe linha da empresa com `estado = 'publicada'`, criada depois da atividade |
  | `enviar_guia_iva` | AT | `submissoes` | `tipo = 'guia_iva'`, `estado = 'aprovado'`, criada depois da atividade |
  | `registar_trabalhador` | Segurança Social | `submissoes` | `tipo = 'registo_trabalhador'`, `estado = 'aprovado'`, criada depois da atividade |
  | `fazer_transferencia` | Prepacoin | `transacoes` | `origem_iban` da empresa, `estado = 'concluida'`, criada depois da atividade |

  Cresce por catálogo, não por código genérico — cada tipo novo pede uma
  entrada nova nesta tabela e o `case` da função de verificação.

---

## Modelo de dados

### `atividade_tipos` — catálogo fechado (só isto tem verificador automático)
| coluna | tipo | notas |
|---|---|---|
| tipo | text pk | `'publicar_vaga'`, `'enviar_guia_iva'`, … |
| app_alvo | text | só informativo — 'talentos', 'AT', 'seg_social', 'prepacoin' |
| descricao | text | mostrado à professora ao escolher |
| link_sugerido | text | URL da página certa do app devido (ex.: `.../talentos/empresa.html`) |
| campos_mostrados | text[] | que campos da tabela de origem aparecem como prova |

Catálogo, não texto livre — a professora escolhe de uma lista; escrita só
por professor (mesma política do `orgao_tipos`).

### `atividades` — o que a professora publica
| coluna | tipo | notas |
|---|---|---|
| id | uuid pk | |
| titulo | text | |
| texto | text | HTML já limpo — mesmo editor/lista branca do Talentos (negrito, listas, ligações) |
| tipo | text fk → atividade_tipos, nullable | null = autodeclaração + anexo |
| alvo_todas | boolean | true = todas as empresas |
| alvo_cedulas | text[] | usado só quando `alvo_todas = false` |
| prazo | timestamptz | |
| criada_por | text | cédula da professora |
| criada_em | timestamptz | |
| encerrada | boolean default false | fecha sem apagar — decisão em aberto, ver acima |

### `atividade_declaracoes` — só para o caminho sem verificador automático
| coluna | tipo | notas |
|---|---|---|
| id | uuid pk | |
| atividade_id | uuid fk | |
| empresa_cedula | text | |
| anexo_caminho | text | Storage, mesmo padrão de path-based RLS já usado (CV do Talentos, SAF-T da AT) |
| declarada_em | timestamptz | |

Não existe tabela de "cumprimentos" para o caminho **com** verificador —
esse resultado calcula-se ao vivo (consulta à tabela de origem), nunca se
guarda uma cópia que possa desalinhar da verdade.

---

## RPCs

- **`dr_atividade_publicar(titulo, texto, tipo, alvo_todas, alvo_cedulas, prazo)`**
  `security definer`, exige `fn_e_professor()`. Grava a atividade e, na mesma
  transação, escreve uma linha em `correio` para cada empresa visada
  (remetente = cédula do próprio Diário, `EP-2026-00004` — mesmo padrão
  "honesto" já usado no resto do ecossistema, nunca uma cédula de sistema
  inventada).

- **`dr_minhas_atividades()`** — devolve as atividades da empresa da sessão,
  cada uma já com o estado calculado: se tem `tipo`, corre o `case` de
  verificação contra a tabela de origem e devolve os campos de prova; se não
  tem, devolve o que estiver em `atividade_declaracoes`.

- **`dr_atividade_declarar(atividade_id, anexo_caminho)`** — só para
  atividades sem `tipo`. Confirma que o anexo existe mesmo no Storage antes
  de aceitar (mesma regra "anexar é sempre conferir" do resto do projeto).

- **`dr_atividades_publicas()`** — montra pública (sem sessão) das atividades
  ativas, para quem quiser rever sem esperar pelo Correio.

---

## Fases

### Fase 0 — fundações
- [ ] `atividade_tipos` + `atividades` + `atividade_declaracoes`, RLS
      (empresa vê as suas, professor vê tudo, catálogo é leitura pública)
- [ ] Bucket de anexos para `atividade_declaracoes` (path
      `<empresaCedula>/<atividadeId>.pdf`, mesmo padrão do CV do Talentos)

### Fase 1 — motor
- [ ] `dr_atividade_publicar` + o fan-out para `correio`
- [ ] `dr_minhas_atividades` com os quatro verificadores do catálogo inicial
- [ ] `dr_atividade_declarar`
- [ ] `dr_atividades_publicas`

### Fase 2 — frontend
- [ ] Página da professora: criar atividade (escolhe tipo do catálogo ou
      deixa livre, título, texto formatado, prazo, alvo)
- [ ] "As minhas atividades" (empresa, com portão): texto da atividade + prova
      ao vivo ou formulário de autodeclaração
- [ ] Montra pública de atividades ativas
- [ ] Cache-busting, teste num Chrome a sério, publicar

---

## Decisão que continua tua

- **Encerrar uma atividade** — apagar, marcar `encerrada`, ou deixar viva até
  ao prazo passar sozinho? Proponho `encerrada` (nunca apagar histórico), mas
  confirma.
- **Quinto+ tipo do catálogo** — para além dos quatro iniciais, há mais
  algum cumprimento que valha a pena verificar automaticamente já na v1, ou
  ficam esses quatro e cresce-se depois?
