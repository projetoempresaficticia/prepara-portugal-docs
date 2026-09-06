# Pendências — para o fim

Tudo o que ficou por fechar, reunido num sítio a pedido do Germano, para
ser visto **depois** dos órgãos estarem todos feitos. A maior parte é
acabamento e responsividade; o que não é está marcado.

Medido a **6 de setembro de 2026**: 5 pessoas · 6 empresas · 85 funções ·
7 protocolos emitidos.

---

## 1. Decisões tuas, que eu não tomo

- [ ] **`drop` das duas sobrecargas antigas do `ass_*`.**
      `ass_criar_documento(text,text,jsonb)` e `ass_assinar(uuid,text)`.
      Estão revogadas (fechadas ao browser) mas vivas. Apagar é destrutivo,
      por isso espero pela tua palavra. *Prova do problema: documento
      `3ebc012f`, sem ficheiro, dado como válido — ficou na base de
      propósito.*
- [ ] **`ass_verificar` exigir `arquivo_url is not null` para dizer
      "íntegro".** É a correção de fundo daquele desvio: fechar as
      sobrecargas tranca a porta, isto tapa o buraco. Muda o comportamento
      do Subsight.
- [ ] **A certidão de contribuições ficar carimbada no despedimento.**
      Hoje é gerada no momento, a partir dos dados. Carimbá-la mudaria o
      significado: de *"o que a Segurança Social vê hoje"* para *"o que
      viu naquele dia"*. Diz-me se queres.
- [ ] **Séries e tipos de ato no Diário da República** — ver a secção
      própria no plano do Diário.

---

## 2. Portar para a AT o que a Segurança Social ganhou

A AT ficou para trás. É o mesmo código, já escrito e testado.

- [ ] **Pagamentos**: abrir o boleto em janela (pagos e por pagar), guardar
      em PDF e imagem, e clicar para copiar a entidade e a referência.
- [ ] **Protocolos com botão "Ver"** no histórico das Declarações e nas
      Entregas com protocolo do Início, com comprovativo imprimível.
- [ ] **`at_comprovativo`**, equivalente do `ss_comprovativo`.
- [ ] Se se repetir uma terceira vez, isto devia subir para a `pp-base` em
      vez de ser copiado outra vez.

---

## 3. Dívida técnica — atravessa todos os repos

- [ ] **26 das 85 funções devolvem `sqlerrm` cru ao browser**
      (`'Falha ao …: ' || sqlerrm`). Põe nomes de colunas e de constraints
      no ecrã de um formando, e viola o R1 da pp-base. Nenhuma das funções
      novas da AT ou da Segurança Social tem isto — é tudo herança.
- [ ] **Rever quem pode chamar o quê.** Desceu de 48 para 44 funções
      abertas a `anon` durante este trabalho, mas a maioria continua a não
      dever estar aberta. *A causa está documentada: o Supabase tem default
      privileges que dão execute a `anon` e `authenticated` a cada função
      nova — revogar só de PUBLIC não fecha nada.*
- [ ] **3 funções com `search_path` mutável** — `fn_digito_controlo`,
      `fn_entidade_de`, `fn_moeda`. Basta `set search_path = public`.
- [ ] **Ligar a proteção de passwords vazadas** no painel do Supabase.
- [ ] **`at_boletos()` está na base mas não no repo.** É a única divergência
      base↔repositório que conheço. Escrever o ficheiro SQL.

---

## 4. Responsividade e acabamento

- [ ] **Nunca foi visto num telemóvel.** Vale para os sete apps. As regras
      estão lá (`svh`, `clamp()`, grelhas que colapsam, listas em vez de
      tabelas) mas ninguém confirmou. **É a maior lacuna desta lista**, e a
      única que só se resolve com um telemóvel na mão.
- [ ] **O gerador de imagem (PNG) nunca correu.** O html2canvas a desenhar
      o conteúdo de um iframe é a única peça do projeto que nunca vi
      funcionar. Se falhar, os botões avisam e o PDF continua a servir.
- [ ] **O ícone `familia`** — o único do kit da Segurança Social que não
      temos. Falta ir buscá-lo ao Figma.
- [ ] **Os 9 kits do Figma da AT** — só 2 foram vistos; o rate limit da API
      cortou o resto. Modais, paginação e tabela vazia ficaram por comparar.

- [ ] **O AeroMail em três colunas num telemóvel.** Colapsa para uma só e
      troca a lista pela leitura conforme `data-lendo`, mas isso nunca foi
      visto a acontecer. Vale o mesmo aviso de cima.

---

## 5. AeroMail — o que ficou dito e não fechado

- [ ] **`document.execCommand` está obsoleto** e é o que dá o negrito, as
      listas e as ligações do editor. Não tem substituto: a API que o havia
      de substituir nunca chegou a existir em todos os browsers. Funciona
      hoje em todos eles; se um dia deixar de funcionar, a alternativa é
      um editor de biblioteca — e isso traz um passo de compilação que o
      projeto não tem.
- [ ] **Anexos ficam órfãos em três casos.** (1) Fechar a janela de escrita
      apaga os ficheiros já carregados, mas fechar o separador não corre
      código nenhum. (2) Quando quem **recebeu** é o último a apagar de vez,
      a linha morre e os caminhos são devolvidos, mas a policy do Storage só
      deixa apagar quem pôs lá o ficheiro — e quem o pôs foi o remetente.
      (3) Trocar de assinatura apaga o molde antigo, mas se essa chamada
      falhar ninguém volta a tentar. Falta uma limpeza periódica do que está
      no bucket `correio` sem linha em `correio_anexos`. Nenhum destes é
      visível para quem usa: são bytes a ocupar espaço.
- [ ] **Sem rascunhos.** Fechar a janela perde o que estava escrito.
- [ ] **As pastas não se arrastam.** Mover é por botão e por janela. Puxar
      uma mensagem para cima de uma pasta na lateral é o gesto que toda a
      gente tenta primeiro, e não faz nada.
- [ ] **Sem selecionar várias mensagens.** As funções do servidor
      (`correio_mover`, `correio_arrumar`) já recebem listas e foram
      testadas com duas de uma vez; falta a app deixar escolher mais do que
      uma. Hoje só "Esvaziar o lixo" usa a lista.
- [ ] **Imagens coladas de fora são deitadas fora.** Colar uma imagem da
      web no editor não a carrega como anexo — o corpo não aceita endereços,
      por isso a imagem desaparece sem explicação. Devia carregá-la, ou pelo
      menos dizer porquê.
- [ ] **Os ícones de lista são desenhados à mão** no `am.css`, porque o
      conjunto descarregado do Figma (User Interface + Arrows) não traz
      nenhum. Os dois ficheiros novos que o Germano indicou — Gmail UI
      Part 2 e Plan It — **nunca chegaram a ser lidos**: o rate limit da
      API do Figma bloqueou as duas tentativas. Falta lá buscar também as
      setas curvas de *responder* e *encaminhar*.

---

## 6. Arrumação

- [ ] **Renomear a pasta local `pp-banco` para `prepacoin`** — o
      repositório já se chama assim; a pasta está presa por um handle do
      Windows. Resolve-se fechando o VS Code.
- [ ] **`github.io/pp-banco/` dá 404.** O GitHub redireciona o repositório
      mas não o Pages. O favorito antigo está partido.
- [ ] **Revogar o token do Figma** partilhado no chat — os repos são
      públicos.

---

## 7. A escala

O projeto foi desenhado para **~1000 formandos em ~200 empresas**. Hoje há
**5 pessoas e 6 empresas**.

- [ ] **`pp-criar-empresa` é o que resolve isto**, e está preso na
      `pp-mensagens` porque atribui números de telemóvel. São dois apps.
- [ ] Decidir se as empresas entram todas de uma vez ou por turmas.

---

## O que eu faria com esta lista

Nada dela bloqueia usar o ecossistema. A ordem que faz sentido:

1. **O telemóvel primeiro** — é a única coisa aqui que pode estar partida
   sem ninguém saber, e afeta todos os apps ao mesmo tempo.
2. **Portar a AT** — é código já escrito, e a diferença entre os dois
   órgãos incomoda a quem usa os dois.
3. **O `sqlerrm` e as permissões** — a única dívida que fica mais cara
   quanto mais se adiar: cada app nova acrescenta funções à lista.
