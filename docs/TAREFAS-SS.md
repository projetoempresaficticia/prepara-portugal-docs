# Tarefas — Segurança Social Direta

Âmbito fechado a **5 de setembro de 2026**. A biblioteca de design está
feita; o que falta é o que está aqui.

---

## O que a app faz

Recebe **admissões de trabalhadores** e **contribuições (TSU)**. Duas
obrigações, já semeadas em `orgao_tipos`:

| tipo | o quê | quando | assinatura | taxa |
|---|---|---|---|---|
| `registo_trabalhador` | admissão — `pessoa_cedula`, `funcao`, `data_inicio` | única | **sim**, `contrato_trabalho` | 0 |
| `tsu` | contribuições do mês — `competencia`, `total_remuneracoes`, `n_trabalhadores` | por ciclo | não | 0 |

Taxa 0 nas duas: tal como no IVA, o valor **não é fixo**. Resolve-se como
na AT — um boleto com o valor calculado, sem tocar no `orgao_submeter`.

### A peça que faz isto valer a pena

Na AT, o IVA saía das faturas que já existiam. Aqui **não há equivalente, e
é por isso que este app importa**: hoje as empresas faturam-se umas às
outras mas ninguém é pago por trabalhar. Zero contratos de trabalho na base.

Pagar um salário passa a ser uma transferência com `categoria = 'salario'`.
A Segurança Social soma essas transferências do mês e compara com o que a
empresa declara — o mesmo confronto que apanha um SAF-T adulterado.

> **Descoberta ao ler o código:** o `banco_transferir` **já** trata
> `'salario'` como obrigação. Falhar um por saldo insuficiente já marca a
> empresa em `incumprimento`. O mecanismo existe há semanas e nunca disparou
> porque nunca ninguém pagou um salário. **Não é preciso mexer no banco.**

---

## Decisões

- **Tipografia: Inter** — a do próprio kit. Assumida na ausência de escolha
  explícita; a biblioteca mostra também Public Sans e Fraunces, e trocar é
  uma linha.
- **TSU real: 23,75% empresa + 11% trabalhador.** Ensina mais e não custa
  mais a programar do que uma taxa única.
- **Documento exigido = documento anexado e verificado.** Regra do Germano,
  válida para todos os órgãos: se o processo real obriga a juntar o contrato,
  a app exige o anexo e confere a assinatura por **duas** vias — o
  `ass_verificar` do Subsight **e** a comparação do SHA-256 com o original.
- **Barra lateral clara**, como no kit — ao contrário da AT, que é escura.
- **Ícones por máscara com `currentColor`** desde o início, em vez de uma
  variante por cor. É a lição da AT aplicada antes de doer.
- **Ouro é preenchimento.** `#F4B400` dá 1,85:1 sobre branco; para escrever
  usa-se `#8A5E00` (5,70:1).

---

## Fase 0 — o que falta antes de começar

- [x] **F0.1 — repositório criado e publicado** ✅ 6 set
      O repositório já existia (criado pelo Germano às 22:20 de 5 set) e
      estava vazio — eu tinha afirmado que não existia sem confirmar, o que
      me levou a uma explicação inteira sobre permissões que era escusada.
      Ligado o remote, enviados os ficheiros, GitHub Pages ligado:
      https://projetoempresaficticia.github.io/seguranca-social/
- [ ] **F0.2 — o ícone `familia`** — o único do kit que não temos.

---

## Fase 1 — trabalhadores

- [x] **F1.1 — `ss_admitir_trabalhador`** ✅ 5 set · `sql/0001_trabalhadores.sql`
      Embrulha o `orgao_pedir_servico('registo_trabalhador', …)` e acrescenta
      o que o genérico não sabe: que a pessoa existe, que não está já ao
      serviço, que o contrato é mesmo entre **esta** empresa e **esta**
      pessoa (pelos slots), e a **dupla verificação** do anexo.

      | # | teste | resultado |
      |---|---|---|
      | 1 | hash do anexo errado | recusado — *"não é o contrato que foi assinado"* |
      | 2 | sem anexo nenhum | recusado |
      | 3 | pessoa que não existe | recusado |
      | 4 | data mal escrita | recusado |
      | 5 | tudo certo | **aprovado**, protocolo **SS-2026-000003** |
      | 6 | admitir a mesma outra vez | recusado — um duplo clique não dobra a TSU |
      | 7 | `ss_trabalhadores` | 1 ao serviço: Rita Mendes, contabilista certificada |

      *Para o teste foi preciso criar o **primeiro contrato de trabalho do
      ecossistema*** — assinado pelo gerente da Padaria Central e pela Rita.
      Até hoje a tabela tinha zero.

- [x] **F1.2 — `ss_trabalhadores(empresa)`** ✅ 5 set
      `p_empresa` nulo = a minha; com valor, só o professor e a própria
      Segurança Social passam. É daqui que sai o `n_trabalhadores` da TSU,
      sem ninguém o inventar.

- [x] **F1.3 — cessação** ✅ 5 set · `sql/0002_cessacao.sql`
      Tipo novo no catálogo, `cessacao_trabalhador`, com **protocolo
      próprio**. A alternativa — marcar o fim dentro dos `dados` da
      admissão — estava fora de questão: esses dados foram carimbados com
      `hash_carimbo`, e mexer neles faria a admissão aparecer como
      **adulterada** na verificação pública.
      *Testado:* cessar quem não está ao serviço, data anterior ao início e
      sem motivo — os três recusados; e a admissão `SS-2026-000003`
      continua `integro: true`.
      *(A cessação positiva fica por correr para a Rita se manter ao serviço
      nos testes da Fase 2.)*

---

## Fase 2 — remunerações e TSU

- [x] **F2.1 — `ss_remuneracoes(competencia, empresa)`** ✅ 5 set
      `sql/0003_remuneracoes_e_tsu.sql`. Soma as transferências com
      `categoria='salario'` da empresa no mês, e **marca as que foram para
      quem não está registado**.
      *Testado com dois salários reais:* P$ 800 à Rita (registada) e P$ 300 a
      quem não está — o segundo sai como `registado: false` e vai para
      `fora_do_registo`, fora do total que conta para a TSU.

- [x] **F2.2 — `ss_tsu_proposta(competencia)`** ✅ 5 set
      80000 × 23,75% = **19000** (empresa) · 80000 × 11% = **8800**
      (trabalhador) · total **27800**.

- [x] **F2.3 — `ss_entregar_dmr`** ✅ 5 set

      | # | teste | resultado |
      |---|---|---|
      | 1 | declarar a menos (esconder o salário) | recusado, mostrando declarado vs visto |
      | 2 | inventar um trabalhador a mais | recusado |
      | 3 | valores certos | boleto `FT-2026-000014`, **P$ 278,00** |
      | 4 | entregar outra vez | recusado — não se cobra a TSU duas vezes |
      | 5 | **pagar** | submissão **aprovada**, protocolo **SS-2026-000004** |
      | 6 | `orgao_verificar_protocolo` | `valido: true`, `integro: true` |

      Saldos: Segurança Social 0 → P$ 278,00 · Padaria Central
      P$ 1942,35 → P$ 564,35 (menos P$ 1100 de salários e P$ 278 de TSU) ·
      Rita P$ 800,00. **O ciclo do trabalho fecha de ponta a ponta pela
      primeira vez.**

- [x] **F2.4 — `ss_minha_carreira()`** ✅ 6 set · `sql/0004_carreira_contributiva.sql`
      *Pedido do Germano a meio da Fase 2:* "o funcionário e a empresa têm
      que ver quanto de contribuição foi paga". Era um buraco real — quem
      trabalha entrava e levava *"sem empresa associada à sessão"*.
      Mostra, por mês e por empresa: remuneração recebida, os 23,75% da
      empresa, os 11% retidos, se foi declarado, se foi pago, e o protocolo.
      Mais os vínculos (onde entrou, de onde saiu).

      **BUG APANHADO NO TESTE, e do pior tipo.** A primeira versão casava a
      declaração da empresa por (empresa, mês) sem verificar se a pessoa
      tinha vínculo. Quem recebia salário **sem estar registado** via
      `declarado: true` e o protocolo da empresa — concluía que estava tudo
      em ordem. Era mentir a quem mais precisa da verdade.
      Corrigido: sem vínculo, contribuições a zero, `em_falta` com os 34,75%
      que deviam ter sido pagos, e um aviso escrito.

      | | Rita (registada) | sem registo |
      |---|---|---|
      | remuneração | P$ 800,00 | P$ 300,00 |
      | contribuição da empresa | P$ 190,00 | **P$ 0,00** |
      | retenção | P$ 88,00 | **P$ 0,00** |
      | declarado / pago | sim / sim | **não / não** |
      | em falta | 0 | **P$ 104,25** |

---

## Fase 3 — os ecrãs

- [x] **F3.1 — Entrada** ✅ 6 set
      Véu a **branco** e não a creme: o fundo já é dourado, e creme sobre
      dourado não separava o cartão. `svh` + `clamp()` e bloco para janelas
      baixas. Traz de origem o que a AT só ganhou depois: mostrar/esconder a
      senha e link público para a consulta de protocolos.

- [x] **F3.2 — Início** ✅ 6 set
      **Tem dois donos**, ao contrário dos outros órgãos. A empresa vê a
      situação contributiva e os quatro passos do mês (admitir → pagar
      salários → declarar → pagar a TSU). Quem trabalha não leva "sem
      empresa associada" — leva a sua carreira.
      O painel fica **vermelho** quando há salários pagos fora do registo,
      mesmo sem dívida nenhuma: gente a trabalhar sem existir para o Estado
      é mais grave do que dever dinheiro.

- [x] **F3.3 — Trabalhadores** ✅ 6 set
      A admissão **não abre um seletor de ficheiros**: lista os contratos já
      assinados no Subsight e usa a impressão digital do original, por isso
      o anexo nunca pode ser outro documento. A baixa é uma janela `<dialog>`
      nativa (foco preso, Escape, fundo inerte de graça).

- [x] **F3.4 — Declarações** ✅ 6 set
      DMR pré-preenchida, com **cada salário listado** e os que foram para
      quem não está registado marcados com "Ação necessária". O botão fica
      desativado se não houver salários pagos.

- [x] **F3.5 — Pagamentos** ✅ 6 set
      Entidade e referência de cada boleto, via a RPC nova **`ss_boletos`**.
      A cédula do órgão resolve-se no servidor — na AT a primeira versão
      tinha-a escrita à mão no browser e teve de ser refeita.

- [x] **F3.6 — Consultar protocolo** ✅ 6 set
      Pública, sem sessão. Distingue *íntegro* de *válido*.

- [x] **F3.7 — A minha carreira** ✅ 6 set
      O ecrã de quem trabalha. Mês a mês: quanto recebeu, quanto a empresa
      contribuiu, quanto lhe foi retido, e o protocolo. Quem recebeu sem
      vínculo vê um painel vermelho com o valor **em falta** e o que fazer.

---

## Fase 4 — fecho

- [ ] **F4.1 — Sem `sqlerrm`** em nenhuma função nova.
- [ ] **F4.2 — Advisor de segurança** e rever quem chama o quê.
- [ ] **F4.3 — Links internos com `?v=`** desde o início — a AT só aprendeu
      isto depois de o erro parecer não pegar.
- [ ] **F4.4 — Ver num telemóvel.** Continua por fazer nos cinco apps.

---

## Ordem

F1 → F2 → F3. A Fase 1 é pequena e destrava tudo: sem trabalhadores
registados não há remunerações para somar nem TSU para calcular.
