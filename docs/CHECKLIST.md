# Checklist — Prepara Portugal

Estado apurado a **6 de setembro de 2026**, medido no Supabase e nos
repositórios, não de memória.

---

## Onde estamos

| | |
|---|---|
| Apps no ar | **7** de 12 |
| Funções na base | **92** |
| Protocolos do Estado emitidos | **7** |
| Empresas ativas | **6** |
| Pessoas ativas | **5** |

O ecossistema já fecha **dois ciclos completos** de ponta a ponta:

**O ciclo fiscal.** A empresa fatura no Prepacoin → exporta o SAF-T →
comunica-o à AT, que o confere contra as faturas que vê → declara o IVA →
paga o boleto → recebe protocolo → qualquer pessoa o confirma.

**O ciclo do trabalho.** Contrato assinado no Subsight → admissão na
Segurança Social, com o PDF anexado e conferido pelo hash → salário pago no
Prepacoin → contribuições declaradas e conferidas contra o banco → boleto
pago → protocolo. E quem trabalha vê a sua carreira contributiva, incluindo
o que ficou por declarar em seu nome.

**O aviso que fecha os dois.** Quando um órgão emite protocolo, um gatilho
escreve no AeroMail a quem submeteu **e a quem assinou** o documento. Os
quatro órgãos passaram a avisar sem que nenhum deles saiba que o correio
existe — e um quinto órgão avisará sem se lembrar de nada.

---

## Feito

- [x] **pp-base** — fundação: auditoria, `api()`, porta única, diretório
- [x] **ClassCard** (pp-identidade) — carteirinha, admin, verificação · azul
- [x] **Subsight** (pp-assinatura) — slots, PDF com hash, verificação · laranja
- [x] **Prepacoin** (pp-banco) — faturas, boletos, SAF-T · lima
- [x] **Cartório Notarial** — certidão permanente · verde
- [x] **Portal das Finanças (AT)** — e-Fatura, IVA, Modelo 22 · roxo
- [x] **Segurança Social** — trabalhadores, TSU, carreira contributiva · ouro
- [x] **AeroMail** (pp-correio) — correio interno, anexos, Realtime · turquesa

---

## O que falta

### 1. O último órgão — adiado por decisão

- [ ] **Diário da República** — *"é legal para as leis do nosso sistema,
      vamos deixar por último"*. É o único órgão cujo produto é **público**:
      a capa é uma lista de leitura sem login. Destrava a publicação de
      vagas e os editais. Âmbito já discutido; falta decidir se as
      publicações têm tipo de ato.

### 2. A outra metade do ecossistema — nada construído

Os cinco repositórios que faltam existem e estão vazios.

| app | precisa de | pode começar? |
|---|---|---|
| **pp-mensagens** | fundação | **já** |
| **pp-emprego** | fundação | **já** |
| **pp-utilities** | correio + órgãos | **já** — o correio está feito |
| **pp-criar-empresa** | banco + **mensagens** | falta mensagens |
| **pp-clientes** | correio + criar-empresa | falta criar-empresa |
| **PRD-08** | tudo | portal único, painel do professor, mapa da cidade |

### 3. A escala — o problema de fundo

O projeto foi desenhado para **~1000 formandos em ~200 empresas**. Hoje há
**5 pessoas e 6 empresas**, quase todas fixtures.

Tudo o que está construído foi testado por **uma** empresa. Cada app nova
será testada por uma empresa também. O projeto só muda de natureza quando
houver muitas — e **`pp-criar-empresa` é a única coisa que o resolve**.
Está preso na `pp-mensagens`, porque atribui números de telemóvel.

### 4. Pendências

Reunidas em [`PENDENCIAS.md`](PENDENCIAS.md), a ver no fim: decisões que
são tuas, o que falta portar da Segurança Social para a AT, a dívida do
`sqlerrm` e das permissões, e a responsividade — que nunca foi vista num
telemóvel em nenhum dos sete apps.

---

## O que eu faria a seguir

**`pp-mensagens` → `pp-criar-empresa`.** Não porque a app de mensagens seja
interessante, mas porque é a única coisa entre aqui e o povoamento. Sem
ela, encher o ecossistema é trabalho manual vezes duzentos.

A alternativa defensável é **`pp-utilities`**, que o AeroMail acabou de
destravar: já há por onde entregar as faturas de água, luz e internet, e é
a única fonte de despesa recorrente que obriga as empresas a gerir
tesouraria em vez de acumular saldo.
