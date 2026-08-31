# PRD-05 — Comunicação

> **Skills responsáveis:** `pp-correio` · `pp-mensagens`
> **Dependência:** PRD-01. Reproduz o tripé de comunicação de um escritório real.

---

## 1. Três registos

| Canal | Uso | Skill |
|---|---|---|
| Correio interno | Correspondência **formal** (ofícios, faturas, contratos, avisos) | `pp-correio` |
| App de mensagens | Troca **rápida/informal** por número fictício | `pp-mensagens` |
| Google Meet | Voz/vídeo (reuniões, atendimento) | externo, sem app |

Distinto do **Gmail/Workspace real das empresas**, que é externo a este sistema.

## 2. `pp-correio` — Correio Interno

**Objetivo:** canal de correspondência **formal** entre pessoas do projeto, por cédula.

**Escopo:**
- Cada pessoa tem a sua **caixa**; envia e recebe mensagens de outras pessoas.
- **Tempo real** via **Supabase Realtime**; autenticação real.
- É onde `pp-utilities` e `pp-clientes` **entregam** faturas e pedidos.

**Nota técnica:** a tabela precisa de estar na publicação `supabase_realtime` para o inbox atualizar ao vivo.

## 3. `pp-mensagens` — App de Mensagens

**Objetivo:** registo **informal/rápido** tipo chat entre pessoas do projeto.

**Escopo:**
- Identificação por **número fictício de telemóvel PT** (`9XX XXX XXX`) ligado à cédula.
- **Conversa contínua** (várias mensagens seguidas) e **grupos** (várias pessoas na mesma conversa).
- **Tempo real** via Supabase Realtime; autenticação real.

**Decisões a fechar (em aberto):**
- Formato exato do número fictício.
- Chat contínuo vs. mensagens avulsas.
- Suporte a grupos (confirmado como desejado; falta especificar o modelo de dados).

## 4. Critérios de aceitação

- Uma mensagem enviada aparece na caixa/conversa do destinatário em tempo real, sem refresh.
- `pp-mensagens` suporta uma conversa com ≥3 participantes.
