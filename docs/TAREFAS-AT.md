# Tarefas — Portal das Finanças (AT)

Decidido a **5 de setembro de 2026**, a partir do que já existe na base e nos
repositórios. A biblioteca de design, os ícones, a marca e o fundo já estão
feitos — o que falta é o que está aqui.

---

## O que a app faz (uma frase por peça)

A AT **não valida documentos** — regista factos fiscais e cobra imposto sobre
eles. Recebe declarações por formulário, recebe o SAF-T que o Prepacoin
exporta, confere-o contra as faturas que ela própria vê, e emite protocolo.
Quem quiser confirmar uma entrega usa o `orgao_verificar_protocolo`, que já
existe e serve todos os órgãos.

Duas obrigações no catálogo, já semeadas em `orgao_tipos`:

| tipo | quando | assinatura | taxa |
|---|---|---|---|
| `guia_iva` | por ciclo | não | P$ 20 |
| `modelo22` | por ciclo | sim, `declaracao_fiscal` | P$ 100 |

---

## Decisões tomadas

- **Sem "Contestar multa".** Fora do âmbito.
- **IVA derivado a 23%** do `valor_total` da fatura, tratado como valor com
  IVA: base = total ÷ 1,23, imposto = total − base. Evita mexer no Prepacoin
  agora; a alternativa correta a prazo é a fatura passar a guardar a taxa.
- **O SAF-T nasce no Prepacoin**, não na AT — é de lá que sai na vida real.
- **O Modelo 22 leva anexo**, com guarda de hash contra o original assinado.
- **O botão de anexo copia os bytes do original** em vez de abrir um seletor
  de ficheiros, para o anexo nunca poder ser outro documento.
- **As multas ficam desligadas** até haver empresas a sério. O mecanismo
  existe; só não se liga o agendador.

---

## Fase 0 — fundações partilhadas

Destravam tudo o resto e atravessam outros repos. Vêm primeiro.

- [x] **F0.1 — `fn_documento_visivel` vê o órgão de destino** ✅ 5 set
      `subsight/sql/0007_orgao_ve_o_que_lhe_e_entregue.sql`.
      A visibilidade **ancora na submissão, não no tipo de documento**: se
      ancorasse em `tipos_documento.orgao_destino`, o órgão veria todos os
      documentos daquele tipo, mesmo os que nunca lhe foram entregues.
      Novo helper `fn_orgao_cedula(orgao)` — o único elo que existe entre um
      órgão e a empresa que o opera é o IBAN das taxas.
      *Testado:* o Cartório vê o documento da submissão `registo_empresa`
      que recebeu; a AT, sobre o mesmo documento, dá `false`.

- [x] **F0.2 — hash do ficheiro** ✅ **já existia — tarefa cancelada**
      Enganei-me no âmbito: o `0004_upload_pdf.sql` do Subsight já tinha
      passado o documento de texto para PDF, e o `hash_conteudo` **já é o
      SHA-256 dos bytes do ficheiro**, calculado no browser com SubtleCrypto
      e travado por `ass_anexar_arquivo`. Não é preciso coluna nenhuma; a
      guarda do anexo do Modelo 22 compara contra o `hash_conteudo`.

- [x] **F0.3 — assinatura do contabilista certificado** ✅ 5 set
      `pp-orgaos/sql/0003_declaracao_fiscal.sql`.
      **Não** se acrescentou o slot ao tipo `declaracao`: a
      `ass_criar_documento` exige que os slots enviados batam *exatamente*
      com os do tipo, por isso isso partia os três serviços do Cartório que
      usam `declaracao` (reconhecimento, registo_empresa, alteracao_registo).
      Criou-se o tipo **`declaracao_fiscal`**, com dois slots — `declarante`
      (gerente, vínculo à empresa) e `contabilista` (pessoa exata, em nome
      próprio, como o TOC real) — e o `modelo22` passou a apontar-lhe.
      *Testado:* `declaracao` continua com 1 slot, o Cartório intacto.

- [x] **F0.4 — contabilista certificado criado e testado** ✅ 5 set
      **PP-2026-00011 · Rita Mendes** · `contabilista@prepara.pt` · papel
      `contabilista` · conta `PT50720452080895429597861`. Criada pela RPC
      própria `id_registar_pessoa`, que trata do utilizador de
      autenticação, da cédula e da abertura de conta. **Sem vínculo a
      empresa** — assina em nome próprio, como o TOC real.
      *Senha inicial `contab2026`; convém trocar.*

      **Teste do circuito completo do Modelo 22:**

      | # | passo | resultado |
      |---|---|---|
      | 1 | criar `declaracao_fiscal` com os 2 slots | criado, pendente |
      | 2 | anexar ficheiro | hash travado |
      | 3 | gerente assina `declarante` | `ASS-PP202600010-C6FC` |
      | 4 | **gerente tenta o slot do contabilista** | **recusado**: "Este slot exige papel contabilista" |
      | 5 | contabilista assina | `ASS-PP202600011-293D`, documento **completo** |
      | 6 | `ass_verificar` | `valido: true`, 2 de 2 |
      | 7 | `orgao_submeter` do Modelo 22 | rejeitado **na taxa**, não na assinatura — o portão abriu |
      | 8 | sem assinatura nenhuma | rejeitado por falta de documento assinado |
      | 9 | quem vê o documento (F0.1) | **só a AT**; Segurança Social e Cartório dão `false` |

- [x] **F0.5 — sobrecargas antigas fechadas ao browser** ✅ 5 set
      `subsight/sql/0008_fecha_sobrecargas_antigas.sql`.
      A `ass_assinar(uuid,text)` é inofensiva — valida tudo, só não grava a
      posição da assinatura na página. A
      `ass_criar_documento(text,text,jsonb)` **é um desvio**: cria o
      documento a partir de texto e trava o hash desse texto, sem PDF
      nenhum — e como o `0004` redefiniu "íntegro" como "tem hash", o
      `ass_verificar` dá **válido** a um documento sem ficheiro.
      Ambas estavam abertas a `anon`, que é a chave pública do HTML.
      *Provado:* documento `3ebc012f` — sem ficheiro, `completo`,
      `ass_verificar` diz `valido: true`. Ficou na base como prova.
      Revogadas (reversível), não apagadas.

- [ ] **F0.7 — `ass_verificar` exigir ficheiro para dizer "íntegro"**
      A correção de fundo do F0.5: fechar as sobrecargas tranca a porta,
      isto tapa o buraco. `integro` passaria a exigir também
      `arquivo_url is not null`. Muda o comportamento do Subsight, por isso
      fica à espera de aval. O `drop` das duas sobrecargas idem.

> **Fixtures de teste ficam.** Por decisão do Germano, os documentos e
> submissões dos testes não se limpam — é útil ter um exemplo visível de
> cada caminho possível, incluindo os que falham.

---

## Fase 1 — o SAF-T sai do Prepacoin

- [x] **F1.1 — `banco_saft(p_competencia)`** ✅ 5 set
      `pp-banco/sql/0011_saft.sql`. Monta o XML no servidor a partir das
      `faturas` onde a minha empresa é emitente no mês. Subconjunto do
      SAF-T (PT) com os nomes reais: `AuditFile` > `Header` +
      `SourceDocuments` > `SalesInvoices` > `Invoice`.
      Novo helper `fn_xml_texto` — as descrições são texto livre e um `&`
      partia o ficheiro. Moeda PPC e a cédula no lugar do NIF, porque não
      temos NIF no ecossistema.
      *Testado com sessão simulada (`set local role authenticated`):*
      3 faturas, base P$ 55,85 + IVA P$ 12,85 = P$ 68,70 de total; a
      fatura anulada ficou de fora; empresa sem faturas dá 0 entradas.
      *Bug apanhado no teste:* competência nula atravessava o guarda —
      `null !~ regex` dá **NULL**, não TRUE, e o `if` não disparava. Saía
      um SAF-T sem datas e vazio. Corrigido com `is null or`.

- [x] **F1.2 — botão “Exportar SAF-T”** ✅ 5 set
      Cartão no fundo de `emitir.html`, que é a casa das faturas no rail.
      Blob no browser, guarda `SAFT-<cedula>-<competencia>.xml`, e mostra o
      resumo (faturas, base, IVA, total) para se conferir antes de entregar.
      Versões recarimbadas: site em `35abd90740d2`.

---

## Fase 2 — o motor da AT

- [x] **F2.1 — bucket `fisco`, privado** ✅ 5 set
      `portal-financas/sql/0001_efatura.sql`. Caminho
      `<cedula>/<competencia>/saft.xml`. Novo helper `fn_fisco_visivel`,
      pelo mesmo motivo da `fn_documento_visivel`: a policy de Storage corre
      como quem chama e não pode usar a `fn_orgao_cedula` diretamente.
      Sem policy de update — um SAF-T entregue não se substitui em silêncio.

- [x] **F2.2 — `at_comunicar_saft`** ✅ 5 set
      Compara **fatura a fatura**, não só totais: comparar totais deixaria
      passar dois erros que se anulam. Guardas contra DTD/entidades
      (bomba de entidades), XML inválido, ficheiro > 2 MB, empresa errada e
      período errado. Regista em `at_efatura_periodos`, idempotente por
      (empresa, competência).

      | # | teste | resultado |
      |---|---|---|
      | 1 | ficheiro honesto | **aceite** — 3 faturas, base P$ 55,85, IVA P$ 12,85 |
      | 2 | valor adulterado 22,50 → 12,50 | recusado, aponta `FT-2026-000011`: ficheiro 1250, AT 2250 |
      | 3 | período errado | recusado |
      | 4 | `<!DOCTYPE>` com entidade | recusado |
      | 5 | XML partido | recusado |
      | 6 | faturas em falta no ficheiro | recusado, nomeia as que faltam |
      | 7 | ficheiro de outra empresa | recusado |
      | 8 | fatura inventada | recusado, nas duas direções |

      *Bug apanhado no teste:* com `unnest(xpath(...))`, o nó de contexto é o
      documento que envolve o `<Invoice>`, não o `<Invoice>` — `./InvoiceNo`
      devolvia vazio e **o ficheiro honesto era recusado**. Tem de ser
      `/Invoice/InvoiceNo`.

- [x] **F2.3 — `at_efatura(p_competencia, p_empresa)`** ✅ 5 set
      `portal-financas/sql/0002_efatura_leitura.sql`. Vendas e compras do
      período, linha a linha com base e IVA já separados, mais os totais e
      se o SAF-T daquele período já foi comunicado.
      `p_empresa` nulo = a minha; com valor, só a AT e o professor passam —
      é o que permitirá à app da AT abrir o e-Fatura de quem está a conferir.
      *Testado:* Padaria Central — 3 vendas (IVA P$ 12,85), 2 compras
      (IVA P$ 12,15). Outra empresa: **recusado**. O professor consegue,
      e viu a AT com 0 vendas e 2 compras.

- [x] **F2.4 — `at_guia_iva_proposta(p_competencia)`** ✅ 5 set
      A conta pré-preenchida: IVA liquidado nas vendas menos IVA dedutível
      nas compras. Devolve `a_entregar` e `a_recuperar` separados, porque o
      resultado tanto pode ser dívida como crédito, e `saft_comunicado`
      para o ecrã poder exigir a comunicação antes da entrega.
      *Testado:* liquidado 1285 − dedutível 1215 = **P$ 0,70 a entregar**.

- [x] **F2.5 — valor variável, sem tocar no que é partilhado** ✅ 5 set
      `portal-financas/sql/0003_guia_iva.sql`.
      **Mudei a abordagem depois de ler o código.** A lista dizia para tornar
      o `orgao_submeter` de valor variável — mas isso é código que o
      Cartório partilha, e o risco não valia a pena. Em vez disso, o
      `at_entregar_guia_iva` emite **um boleto com duas linhas**: taxa de
      processamento (fixa, P$ 20) + imposto (o que resultar). Uma referência
      só, e o `trg_orgao_boleto_pago` que já existia aprova e dá protocolo
      quando for paga. **Nenhuma função partilhada foi alterada.**

      | # | teste | resultado |
      |---|---|---|
      | 1 | valores inventados (5000 / 100) | recusado, mostrando lado a lado o declarado e o que a AT vê |
      | 2 | valores certos (5585 / 1285) | boleto `FT-2026-000012`, total P$ 32,85 |
      | 3 | entregar outra vez | recusado — uma guia por período |
      | 4 | período sem SAF-T comunicado | recusado — comunicar primeiro |
      | 5 | pagar o boleto | submissão **aprovada**, protocolo **AT-2026-000001** |
      | 6 | `orgao_verificar_protocolo` | `valido: true`, `integro: true` |
      | 7 | **regressão do Cartório** | certidão pedida na mesma, `FT-2026-000013` |

      Saldos: AT 0 → P$ 32,85; Padaria Central −P$ 32,85. O ciclo fiscal fecha
      de ponta a ponta pela primeira vez.

- [x] **F2.6 — taxas do Estado deixam de descontar IVA** ✅ 5 set
      *(encontrado ao escrever a F2.5)*
      A F2.3/F2.4 contavam **todas** as faturas recebidas como compras com
      IVA dedutível — incluindo as taxas do Cartório e do Diário. Imposto
      pago ao Estado não desconta. Novo helper `fn_e_orgao`; as taxas
      continuam a aparecer no e-Fatura, marcadas `dedutivel: false` e com
      total próprio `iva_nao_dedutivel`, mas fora do que desconta.
      **Mudou o número a declarar de P$ 0,70 para P$ 12,85.**

---

## Fase 3 — os ecrãs

Ordem de construção; cada um depende da fase anterior.

- [x] **F3.1 — Entrada** ✅ 5 set · `index.html` + bloco novo na `at.css`.
      Fundo com véu em gradiente, cartão à esquerda, `svh` + `clamp()` e um
      bloco para janelas baixas (`max-height: 780px`) que encolhe os espaços
      em vez de empurrar o botão para fora do ecrã. Em <900px o fundo sai e
      fica o cartão centrado.
      *Nota:* a auditoria de paleta que ocupava o `index.html` passou a
      `paleta.html` — a porta da app não pode ser um documento de design.

- [x] **F3.2 — Início** ✅ 5 set · `app.js`.
      Responde a três perguntas por ordem: estou em dia? o que falta neste
      período? o que já ficou entregue?
      O painel do topo muda de cor e de ícone conforme o estado, e a nota diz
      **o que fazer a seguir**, não só o diagnóstico. Os três passos do
      período — comunicar, declarar, pagar — estão numerados porque a ordem
      é mesmo uma sequência, e cada botão só aparece quando o passo anterior
      está feito.
      Filtra por `orgao === 'AT'`: o que é do Cartório mostra-se no Cartório.

      **Peças novas na biblioteca:** `at.js` (utilitários + barra lateral
      montada em JS, para não haver seis cópias que se desalinham), classes
      `.at-linha*`, `.at-linha-passo`, `.at-passo-num`, variantes de
      `.at-msg` e os estados do `.at-hero`.

      **Por ver:** nunca abri isto num browser — não tenho como. Confirmei
      que todas as classes usadas existem na `at.css` e que o JS analisa sem
      erros, mas o aspeto ainda não foi visto por ninguém.

      O sistema de versões passou de `pc-versao` para `at-versao`; site em
      `c5be3e5cabaf`.
- [x] **F3.3 — e-Fatura** ✅ 5 set · `efatura.html` + `efatura.js`
      Vendas e compras do período linha a linha, com as taxas do Estado
      marcadas *“Não desconta IVA”*. A comunicação do SAF-T guarda o
      ficheiro no bucket `fisco` **antes** de ser aceite — uma entrega
      recusada também é um facto — e, quando a AT recusa, mostra as
      divergências fatura a fatura: o que o ficheiro diz e o que a AT vê.

- [x] **F3.4 — Declarações** ✅ 5 set · `declaracoes.html` + `declaracoes.js`
      Guia de IVA pré-preenchida do e-Fatura, com o botão **desativado**
      enquanto o SAF-T do período não estiver comunicado — não se declara
      sobre faturas que a AT não recebeu. Modelo 22 com escolha da
      declaração assinada no Subsight (lista só as `completo`). Histórico
      com protocolos e com o motivo das recusadas.

- [x] **F3.5 — Pagamentos** ✅ 5 set · `pagamentos.html` + `pagamentos.js`
      Entidade e referência de cada boleto da AT; o pagamento faz-se no
      Prepacoin, como na vida real. Nova RPC **`at_boletos`**: saber que
      boletos são da AT obriga a cruzar o órgão com a conta dele, e a
      primeira versão tinha a cédula `EP-2026-00006` escrita à mão no
      browser. Passou para o servidor.

- [x] **F3.6 — Consultar protocolo** ✅ 5 set · `consultar.html` + `consultar.js`
      **Pública, sem sessão** — quem precisa de confirmar uma entrega é um
      terceiro, não a empresa que a fez. Distingue *íntegro* de *válido*:
      um protocolo pode existir e os dados terem sido alterados depois.
      Aceita `?protocolo=` no URL, que é como as outras páginas lhe apontam.

> **A biblioteca de design saiu da navegação** de todos os apps (AT,
> Cartório e Prepacoin), por decisão do Germano. Os ficheiros ficam no
> repositório como documentação; quem usa a app não tem nada a fazer lá.

---

## Fase 4 — fecho

- [ ] **F4.1 — Sem `sqlerrm` nas funções novas.** Nem uma. O
      `orgao_verificar_protocolo` já tem a fuga e está na lista geral do
      `CHECKLIST.md`; as novas não a acrescentam.
- [ ] **F4.2 — Correr o advisor de segurança** depois do SQL todo, e rever
      quem pode chamar as funções `at_*`.
- [ ] **F4.3 — Sistema de versões** — o `versoes.py` já está portado;
      confirmar que o `versao.json` e a meta `pc-versao` estão a ser
      carimbados no deploy.
- [ ] **F4.4 — Ver num telemóvel a sério.** Vale para os cinco apps e nunca
      foi feito.

---

## Ordem sugerida

**F0 → F1 → F2 → F3 → F4.** As fases 0 e 1 são pequenas e mexem noutros
repos; despachadas primeiro, a AT constrói-se sem voltar atrás. A F2.5 é a
única com risco de arrastar, porque altera uma RPC que o Cartório já usa —
convém testar o Cartório depois de lhe mexer.
