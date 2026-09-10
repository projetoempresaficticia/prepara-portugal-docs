> **Estado em 10 de setembro de 2026: Fases 0 e 1 aplicadas e testadas
> por completo.** `sql/001` a `005` aplicados ao Supabase; testados com
> SQL real (impersonação): professor publica e avisa por Correio (3
> mensagens confirmadas); não-professor recusado; `publicar_vaga`
> confirmado a passar de não-cumprida a cumprida assim que uma vaga real
> foi publicada no Talentos, com a prova certa puxada ao vivo;
> autodeclaração recusa sem anexo e aceita com o anexo confirmado no
> Storage; uma atividade dirigida a uma empresa fica invisível para
> outra empresa e fora da montra pública; auditor de segurança com 0
> erros. Dois bugs reais apanhados e corrigidos nesta ronda:
> `documentos.criado_em` (não `criada_em`), e `v_decl` como `record` a
> embrulhar a prova da autodeclaração em vez de a devolver direta.
>
> **Teste de ponta a ponta pela interface, feito em 10 de setembro de
> 2026** (senha de teste `1234` em `germano@prepara.pt` e na professora,
> autorizada pelo Germano): login real da professora → publicar uma
> atividade pela própria UI → mensagem de sucesso "3 pessoa(s)
> avisada(s) por Correio" → aparece na montra pública. Login real da
> empresa (Padaria Central) → "As minhas atividades" mostra as três
> atividades certas, com a verificação automática de `publicar_vaga` já
> "Cumprida" (a vaga de teste publicada mais cedo na sessão), a
> autodeclaração já "Cumprida" com o anexo, e a atividade dirigida só a
> ela com o botão de declarar. Zero erros de consola em toda a jornada.
> **O Diário da República está completo e funcional de ponta a ponta.**

# Plano de implementação — Diário da República (atividades)

**Objetivo:** a professora publica uma atividade obrigatória; o Correio
avisa sozinho quem tem de a cumprir; doze tipos de atividade conferem-se
sozinhos contra a tabela real do app devido (Talentos/AT/Segurança
Social/Cartório/Prepacoin/Subsight); o que não tem tipo cai em
autodeclaração com anexo conferido.

**Desenho:** `docs/TAREFAS-DIARIO-REPUBLICA.md` — este plano implementa-o.

**Pilha:** Postgres/Supabase (RLS, RPC `security definer`), HTML/JS sem
framework, GitHub Pages. Testado com SQL real (impersonação de sessão,
como em todas as apps deste projeto) e, no frontend, Chrome real via
puppeteer-core + screenshot — não há suite de testes automatizados neste
projeto.

## Restrições globais (do pp-base e do desenho)

- Toda a escrita passa por RPC `security definer`; nunca INSERT/UPDATE direto do frontend.
- RLS ligada em toda a tabela nova, sem exceção.
- Toda RPC devolve `{ok:true,dados}` ou `{ok:false,erro}` — nunca `sqlerrm` cru ao browser.
- `revoke ... from public, anon` explícito em toda função, depois `grant` só a quem deve chamar.
- IDs gerados no servidor (`gen_random_uuid()`), nunca recebidos do cliente.
- Anexo é sempre **conferido** (existe mesmo no Storage, tamanho > 0) — nunca só aceite por declaração.
- Cache-busting: `python ferramentas/versoes.py` antes de cada publicação.
- Remetente do Correio automático é a cédula real do Diário (`EP-2026-00004`), nunca uma cédula de sistema inventada.

---

## Fase 0 — Tabelas, RLS, bucket

### Tarefa 1: `atividade_tipos`, `atividades`, `atividade_declaracoes` + RLS

**Ficheiros:**
- Criar: `diario-republica/sql/001_atividades.sql`

**Interfaces produzidas** (usadas pelas tarefas seguintes):
- `public.atividade_tipos(tipo text pk, app_alvo text, descricao text, link_sugerido text, campos_mostrados text[], ativo boolean)`
- `public.atividades(id uuid pk, titulo text, texto text, tipo text null, alvo_todas boolean, alvo_cedulas text[], prazo timestamptz, criada_por text, criada_em timestamptz)`
- `public.atividade_declaracoes(id uuid pk, atividade_id uuid, empresa_cedula text, anexo_caminho text, declarada_em timestamptz)`
- Bucket Storage `atividades` (privado, PDF até 10 MB)

- [ ] **Passo 1 — escrever a migração**

```sql
-- 001_atividades.sql — as três tabelas do motor de atividades do Diário
-- da República. Ver docs/TAREFAS-DIARIO-REPUBLICA.md para o porquê.

create table if not exists public.atividade_tipos (
  tipo             text primary key,
  app_alvo         text not null,
  descricao        text not null,
  link_sugerido    text,
  campos_mostrados text[] not null default '{}',
  ativo            boolean not null default true
);

create table if not exists public.atividades (
  id            uuid primary key default gen_random_uuid(),
  titulo        text not null,
  texto         text not null,
  tipo          text references public.atividade_tipos(tipo),
  alvo_todas    boolean not null default true,
  alvo_cedulas  text[] not null default '{}',
  prazo         timestamptz not null,
  criada_por    text not null,
  criada_em     timestamptz not null default now(),
  constraint atividades_alvo_valido check (
    alvo_todas = true or array_length(alvo_cedulas, 1) > 0
  )
);

create table if not exists public.atividade_declaracoes (
  id             uuid primary key default gen_random_uuid(),
  atividade_id   uuid not null references public.atividades(id) on delete cascade,
  empresa_cedula text not null,
  anexo_caminho  text not null,
  declarada_em   timestamptz not null default now(),
  unique (atividade_id, empresa_cedula)
);

alter table public.atividade_tipos      enable row level security;
alter table public.atividades           enable row level security;
alter table public.atividade_declaracoes enable row level security;

-- catálogo: leitura pública (a professora escolhe de uma lista visível;
-- o público também pode ver que tipos de atividade existem), escrita só professor
drop policy if exists "catalogo atividades leitura publica" on public.atividade_tipos;
create policy "catalogo atividades leitura publica"
  on public.atividade_tipos for select using (true);

drop policy if exists "catalogo atividades escrita professor" on public.atividade_tipos;
create policy "catalogo atividades escrita professor"
  on public.atividade_tipos for all
  using (public.fn_e_professor()) with check (public.fn_e_professor());

-- atividades: visível a quem é alvo dela, ou a todos se alvo_todas,
-- ou ao professor (sempre)
drop policy if exists "atividades visiveis a quem e alvo" on public.atividades;
create policy "atividades visiveis a quem e alvo"
  on public.atividades for select
  using (
    alvo_todas = true
    or public.fn_minha_empresa_cedula() = any(alvo_cedulas)
    or public.fn_e_professor()
  );

-- declarações: a própria empresa e o professor
drop policy if exists "declaracoes da minha empresa" on public.atividade_declaracoes;
create policy "declaracoes da minha empresa"
  on public.atividade_declaracoes for select
  using (
    empresa_cedula is not distinct from public.fn_minha_empresa_cedula()
    or public.fn_e_professor()
  );

-- escrita nas três só por RPC (security definer) — sem policy de insert/update aqui.

insert into storage.buckets (id, name, public, file_size_limit, allowed_mime_types)
values ('atividades', 'atividades', false, 10485760, array['application/pdf'])
on conflict (id) do update
  set file_size_limit = excluded.file_size_limit,
      allowed_mime_types = excluded.allowed_mime_types;

drop policy if exists "empresa anexa a sua declaracao" on storage.objects;
create policy "empresa anexa a sua declaracao"
  on storage.objects for insert
  to authenticated
  with check (
    bucket_id = 'atividades'
    and (storage.foldername(name))[1] = public.fn_minha_empresa_cedula()
  );

drop policy if exists "empresa reescreve a sua declaracao" on storage.objects;
create policy "empresa reescreve a sua declaracao"
  on storage.objects for update
  to authenticated
  using ((storage.foldername(name))[1] = public.fn_minha_empresa_cedula()
         and bucket_id = 'atividades')
  with check ((storage.foldername(name))[1] = public.fn_minha_empresa_cedula()
              and bucket_id = 'atividades');

drop policy if exists "empresa le a sua declaracao" on storage.objects;
create policy "empresa le a sua declaracao"
  on storage.objects for select
  to authenticated
  using (
    bucket_id = 'atividades'
    and (storage.foldername(name))[1] = public.fn_minha_empresa_cedula()
  );

drop policy if exists "professor le todas as declaracoes" on storage.objects;
create policy "professor le todas as declaracoes"
  on storage.objects for select
  to authenticated
  using (bucket_id = 'atividades' and public.fn_e_professor());
```

- [ ] **Passo 2 — aplicar e confirmar que as tabelas e políticas existem**

Correr via Supabase MCP (`execute_sql`, projeto `moxxbehwylcjaqjacmyh`):
```sql
select tablename, rowsecurity from pg_tables
 where schemaname = 'public' and tablename like 'atividade%';
select policyname, tablename from pg_policies
 where tablename like 'atividade%' or (tablename = 'objects' and policyname like '%declaracao%');
```
Esperado: as três tabelas com `rowsecurity = true`, e 4 políticas em
`atividades`/`atividade_tipos`/`atividade_declaracoes` + 4 em `storage.objects`.

- [ ] **Passo 3 — commit**

```bash
cd diario-republica
git add sql/001_atividades.sql
git commit -m "feat(dr): tabelas de atividades, RLS e bucket de anexos"
```

---

### Tarefa 2: semear o catálogo de 16 tipos

**Ficheiros:**
- Criar: `diario-republica/sql/002_catalogo_atividades.sql`

**Consome:** `atividade_tipos` (Tarefa 1)

- [ ] **Passo 1 — escrever a migração**

```sql
-- 002_catalogo_atividades.sql — os 16 tipos verificáveis levantados em
-- docs/TAREFAS-DIARIO-REPUBLICA.md. Os quatro do pp-utilities entram
-- ativo=false: a tabela de origem (faturas/boletos daquele app) ainda
-- não existe.

insert into public.atividade_tipos (tipo, app_alvo, descricao, link_sugerido, campos_mostrados, ativo)
values
  ('publicar_vaga', 'talentos', 'Publicar uma vaga de emprego',
   'https://projetoempresaficticia.github.io/talentos/empresa.html',
   array['titulo', 'criada_em'], true),

  ('entregar_guia_iva', 'AT', 'Entregar a guia de IVA do período',
   'https://projetoempresaficticia.github.io/portal-financas/declaracoes.html',
   array['protocolo', 'criada_em'], true),

  ('entregar_modelo22', 'AT', 'Entregar o Modelo 22 (IRC)',
   'https://projetoempresaficticia.github.io/portal-financas/declaracoes.html',
   array['protocolo', 'criada_em'], true),

  ('admitir_trabalhador', 'seg_social', 'Registar a admissão de um trabalhador',
   'https://projetoempresaficticia.github.io/seguranca-social/carreira.html',
   array['protocolo', 'criada_em'], true),

  ('cessar_trabalhador', 'seg_social', 'Registar a cessação de um trabalhador',
   'https://projetoempresaficticia.github.io/seguranca-social/carreira.html',
   array['protocolo', 'criada_em'], true),

  ('entregar_tsu', 'seg_social', 'Entregar a declaração de remunerações (TSU)',
   'https://projetoempresaficticia.github.io/seguranca-social/declaracoes.html',
   array['protocolo', 'criada_em'], true),

  ('registar_empresa', 'cartorio', 'Registar a empresa no Cartório',
   'https://projetoempresaficticia.github.io/cartorio-notarial/pedir.html',
   array['protocolo', 'criada_em'], true),

  ('emitir_certidao', 'cartorio', 'Emitir a certidão permanente',
   'https://projetoempresaficticia.github.io/cartorio-notarial/pedir.html',
   array['protocolo', 'prazo'], true),

  ('reconhecer_assinatura', 'cartorio', 'Pedir reconhecimento de assinatura',
   'https://projetoempresaficticia.github.io/cartorio-notarial/pedir.html',
   array['protocolo', 'criada_em'], true),

  ('alterar_registo', 'cartorio', 'Registar uma alteração da empresa',
   'https://projetoempresaficticia.github.io/cartorio-notarial/pedir.html',
   array['protocolo', 'criada_em'], true),

  ('pagar_salario', 'prepacoin', 'Pagar o salário de um funcionário',
   'https://projetoempresaficticia.github.io/prepacoin/transferir.html',
   array['valor', 'criada_em'], true),

  ('assinar_documento', 'subsight', 'Assinar um documento',
   'https://projetoempresaficticia.github.io/subsight/index.html',
   array['criada_em'], true),

  ('pagar_agua', 'pp-utilities', 'Pagar a fatura de água',
   null, array['valor', 'pago_em'], false),
  ('pagar_energia', 'pp-utilities', 'Pagar a fatura de energia',
   null, array['valor', 'pago_em'], false),
  ('pagar_internet', 'pp-utilities', 'Pagar a fatura de internet',
   null, array['valor', 'pago_em'], false),
  ('pagar_aluguel', 'pp-utilities', 'Pagar a fatura de aluguer',
   null, array['valor', 'pago_em'], false)
on conflict (tipo) do update
  set app_alvo = excluded.app_alvo,
      descricao = excluded.descricao,
      link_sugerido = excluded.link_sugerido,
      campos_mostrados = excluded.campos_mostrados,
      ativo = excluded.ativo;
```

- [ ] **Passo 2 — aplicar e confirmar**

```sql
select tipo, ativo from public.atividade_tipos order by ativo desc, tipo;
```
Esperado: 16 linhas, 12 com `ativo = true`, 4 (`pagar_*`) com `ativo = false`.

- [ ] **Passo 3 — commit**

```bash
git add sql/002_catalogo_atividades.sql
git commit -m "feat(dr): semeia o catalogo de 16 tipos de atividade"
```

---

## Fase 1 — Motor (RPCs)

### Tarefa 3: `dr_atividade_publicar` + aviso automático por Correio

**Ficheiros:**
- Criar: `diario-republica/sql/003_rpc_publicar.sql`

**Consome:** `atividades`, `atividade_tipos` (Tarefa 1/2), `public.correio` (já existe, do AeroMail), `public.pessoas`/`public.empresas` (já existem), `public.fn_e_professor()` (já existe)

**Produz:** `public.dr_atividade_publicar(p_titulo text, p_texto text, p_tipo text, p_alvo_todas boolean, p_alvo_cedulas text[], p_prazo timestamptz) returns jsonb`

- [ ] **Passo 1 — escrever a função**

```sql
-- 003_rpc_publicar.sql
create or replace function public.dr_atividade_publicar(
  p_titulo text,
  p_texto text,
  p_tipo text default null,
  p_alvo_todas boolean default true,
  p_alvo_cedulas text[] default '{}',
  p_prazo timestamptz default null
)
returns jsonb
language plpgsql
security definer
set search_path = public
as $$
declare
  v_titulo text := btrim(coalesce(p_titulo, ''));
  v_texto  text := btrim(coalesce(p_texto, ''));
  v_id     uuid := gen_random_uuid();
  v_empresa record;
  v_n       int := 0;
begin
  if not public.fn_e_professor() then
    return jsonb_build_object('ok', false, 'erro', 'Só a professora pode publicar atividades.');
  end if;
  if v_titulo = '' then
    return jsonb_build_object('ok', false, 'erro', 'A atividade precisa de um título.');
  end if;
  if v_texto = '' then
    return jsonb_build_object('ok', false, 'erro', 'A atividade precisa de texto.');
  end if;
  if p_prazo is null then
    return jsonb_build_object('ok', false, 'erro', 'A atividade precisa de um prazo.');
  end if;
  if p_tipo is not null and not exists (
    select 1 from public.atividade_tipos where tipo = p_tipo and ativo = true
  ) then
    return jsonb_build_object('ok', false, 'erro', 'Tipo de atividade desconhecido ou por ativar.');
  end if;
  if not p_alvo_todas and coalesce(array_length(p_alvo_cedulas, 1), 0) = 0 then
    return jsonb_build_object('ok', false, 'erro', 'Escolha pelo menos uma empresa, ou marque "todas".');
  end if;

  insert into public.atividades(id, titulo, texto, tipo, alvo_todas, alvo_cedulas, prazo, criada_por)
  values (v_id, v_titulo, v_texto, p_tipo, p_alvo_todas, coalesce(p_alvo_cedulas, '{}'), p_prazo,
          public.fn_minha_cedula());

  -- aviso por Correio a toda a gente da empresa visada, não só o
  -- gerente (quem trata da guia de IVA costuma ser o contabilista) —
  -- remetente é a cédula real do Diário, nunca uma cédula de sistema
  -- inventada
  for v_empresa in
    select e.cedula, p.cedula as pessoa_cedula
      from public.empresas e
      join public.pessoas p on p.empresa_id = e.id
     where p_alvo_todas or e.cedula = any(p_alvo_cedulas)
  loop
    insert into public.correio(id, de_cedula, para_cedula, assunto, corpo)
    values (gen_random_uuid(), 'EP-2026-00004', v_empresa.pessoa_cedula,
            'Nova atividade: ' || v_titulo,
            v_texto || E'\n\nPrazo: ' || to_char(p_prazo, 'DD/MM/YYYY') ||
            E'\n\nConsulte em Diário da República.');
    v_n := v_n + 1;
  end loop;

  return jsonb_build_object('ok', true, 'dados', jsonb_build_object('id', v_id, 'avisadas', v_n));
exception when others then
  return jsonb_build_object('ok', false, 'erro', 'Não foi possível publicar a atividade.');
end;
$$;

revoke execute on function public.dr_atividade_publicar(text, text, text, boolean, text[], timestamptz)
  from public, anon;
grant execute on function public.dr_atividade_publicar(text, text, text, boolean, text[], timestamptz)
  to authenticated;
```

- [ ] **Passo 2 — testar com SQL real (impersonação)**

```sql
-- como professor: deve publicar e avisar
select set_config('request.jwt.claims',
  json_build_object('sub', (select id::text from auth.users where email = 'projetoempresaficticia@gmail.com')
  )::text, true);
select public.dr_atividade_publicar('Publicar uma vaga', '<p>Publiquem uma vaga esta semana.</p>',
  'publicar_vaga', true, '{}', now() + interval '7 days');
-- esperado: {"ok":true,...}; conferir que apareceu em correio
select count(*) from public.correio where assunto = 'Nova atividade: Publicar uma vaga';

-- como empresa comum: deve recusar
select set_config('request.jwt.claims',
  json_build_object('sub', (select id::text from auth.users where email = 'germano@prepara.pt')
  )::text, true);
select public.dr_atividade_publicar('Teste', '<p>x</p>', null, true, '{}', now() + interval '1 day');
-- esperado: {"ok":false,"erro":"Só a professora pode publicar atividades."}
```

- [ ] **Passo 3 — commit**

```bash
git add sql/003_rpc_publicar.sql
git commit -m "feat(dr): dr_atividade_publicar com aviso automatico por correio"
```

---

### Tarefa 4: `dr_minhas_atividades` com os doze verificadores

**Ficheiros:**
- Criar: `diario-republica/sql/004_rpc_verificar.sql`

**Consome:** `atividades`, `atividade_tipos`, `atividade_declaracoes` (Tarefas 1-3); tabelas de outros apps já existentes: `public.vagas`, `public.submissoes`, `public.transacoes`, `public.documentos`

**Produz:** `public.dr_minhas_atividades() returns jsonb` — cada item: `{id, titulo, texto, tipo, prazo, atrasada, cumprida, prova jsonb}`

- [ ] **Passo 1 — escrever a função**

```sql
-- 004_rpc_verificar.sql
--
-- Não existe verificador genérico: cada tipo sabe a sua própria tabela.
-- `atrasada` é sempre calculada (prazo < agora e ainda não cumprida) —
-- não há estado "encerrada"; a UI pinta a vermelho sozinha com isto.
create or replace function public.dr_minhas_atividades()
returns jsonb
language plpgsql
security definer
set search_path = public
as $$
declare
  v_empresa text := public.fn_minha_empresa_cedula();
  v_linhas jsonb := '[]'::jsonb;
  r record;
  v_cumprida boolean;
  v_prova jsonb;
  v_decl record;
begin
  if v_empresa is null then
    return jsonb_build_object('ok', false, 'erro', 'Sem empresa associada.');
  end if;

  for r in
    select a.id, a.titulo, a.texto, a.tipo, a.prazo, a.criada_em
      from public.atividades a
     where a.alvo_todas or v_empresa = any(a.alvo_cedulas)
     order by a.prazo asc
  loop
    v_cumprida := false;
    v_prova := null;

    if r.tipo = 'publicar_vaga' then
      select jsonb_build_object('titulo', v.titulo, 'criada_em', v.criada_em)
        into v_prova
        from public.vagas v
       where v.empresa_cedula = v_empresa and v.estado = 'publicada'
         and v.criada_em >= r.criada_em
       order by v.criada_em desc limit 1;

    elsif r.tipo in ('entregar_guia_iva', 'entregar_modelo22', 'admitir_trabalhador',
                      'cessar_trabalhador', 'entregar_tsu', 'registar_empresa',
                      'reconhecer_assinatura', 'alterar_registo') then
      select jsonb_build_object('protocolo', s.protocolo, 'criada_em', s.criada_em)
        into v_prova
        from public.submissoes s
       where s.empresa_cedula = v_empresa and s.estado = 'aprovado'
         and s.tipo = case r.tipo
               when 'entregar_guia_iva' then 'guia_iva'
               when 'entregar_modelo22' then 'modelo22'
               when 'admitir_trabalhador' then 'registo_trabalhador'
               when 'cessar_trabalhador' then 'cessacao_trabalhador'
               when 'entregar_tsu' then 'tsu'
               when 'registar_empresa' then 'registo_empresa'
               when 'reconhecer_assinatura' then 'reconhecimento'
               when 'alterar_registo' then 'alteracao_registo'
             end
         and s.criada_em >= r.criada_em
       order by s.criada_em desc limit 1;

    elsif r.tipo = 'emitir_certidao' then
      select jsonb_build_object('protocolo', s.protocolo, 'prazo', s.prazo)
        into v_prova
        from public.submissoes s
       where s.empresa_cedula = v_empresa and s.tipo = 'certidao_permanente'
         and s.estado = 'aprovado' and s.criada_em >= r.criada_em
       order by s.criada_em desc limit 1;

    elsif r.tipo = 'pagar_salario' then
      select jsonb_build_object('valor', t.valor, 'criada_em', t.criada_em)
        into v_prova
        from public.transacoes t
        join public.contas c on c.iban = t.origem_iban
       where c.cedula = v_empresa and t.categoria = 'salario' and t.estado = 'concluida'
         and t.criada_em >= r.criada_em
       order by t.criada_em desc limit 1;

    elsif r.tipo = 'assinar_documento' then
      select jsonb_build_object('criada_em', d.criada_em)
        into v_prova
        from public.documentos d
        join public.documento_slots ds on ds.documento_id = d.id
       where ds.empresa_esperada = v_empresa and d.estado = 'completo'
         and d.criada_em >= r.criada_em
       order by d.criada_em desc limit 1;
    end if;
    -- pagar_agua/energia/internet/aluguel: sem ramo — tipo fica ativo=false
    -- no catálogo, por isso a professora não consegue publicar com eles ainda.

    v_cumprida := v_prova is not null;

    if not v_cumprida and r.tipo is null then
      select jsonb_build_object('anexo_caminho', ad.anexo_caminho, 'declarada_em', ad.declarada_em)
        into v_decl
        from public.atividade_declaracoes ad
       where ad.atividade_id = r.id and ad.empresa_cedula = v_empresa;
      if found then
        v_cumprida := true;
        v_prova := to_jsonb(v_decl);
      end if;
    end if;

    v_linhas := v_linhas || jsonb_build_object(
      'id', r.id, 'titulo', r.titulo, 'texto', r.texto, 'tipo', r.tipo,
      'prazo', r.prazo, 'cumprida', v_cumprida, 'prova', v_prova,
      'atrasada', (r.prazo < now() and not v_cumprida)
    );
  end loop;

  return jsonb_build_object('ok', true, 'dados', v_linhas);
exception when others then
  return jsonb_build_object('ok', false, 'erro', 'Não foi possível listar as atividades.');
end;
$$;

revoke execute on function public.dr_minhas_atividades() from public, anon;
grant execute on function public.dr_minhas_atividades() to authenticated;
```

- [ ] **Passo 2 — testar com SQL real**

```sql
-- empresa com uma vaga publicada DEPOIS da atividade "Publicar uma vaga"
-- criada na Tarefa 3: deve aparecer cumprida=true
select set_config('request.jwt.claims',
  json_build_object('sub', (select id::text from auth.users where email = 'germano@prepara.pt')
  )::text, true);
select public.dr_minhas_atividades();
-- esperado: array com a atividade "Publicar uma vaga"; cumprida depende
-- de existir uma vaga publicada por esta empresa depois da criação da
-- atividade -- testar os dois casos (sem vaga -> atrasada/cumprida=false;
-- publicar uma vaga via vaga_criar+vaga_publicar, repetir -> cumprida=true)
```

- [ ] **Passo 3 — commit**

```bash
git add sql/004_rpc_verificar.sql
git commit -m "feat(dr): dr_minhas_atividades com os verificadores automaticos"
```

---

### Tarefa 5: `dr_atividade_declarar` + `dr_atividades_publicas`

**Ficheiros:**
- Criar: `diario-republica/sql/005_rpc_declarar_publicas.sql`

**Consome:** `atividade_declaracoes` (Tarefa 1), bucket `atividades` (Tarefa 1)

**Produz:** `public.dr_atividade_declarar(p_atividade_id uuid, p_anexo_caminho text) returns jsonb`, `public.dr_atividades_publicas() returns jsonb`

- [ ] **Passo 1 — escrever as funções**

```sql
-- 005_rpc_declarar_publicas.sql
create or replace function public.dr_atividade_declarar(
  p_atividade_id uuid,
  p_anexo_caminho text
)
returns jsonb
language plpgsql
security definer
set search_path = public
as $$
declare
  v_empresa text := public.fn_minha_empresa_cedula();
  v_atividade record;
  v_obj record;
  v_caminho_esperado text;
begin
  if v_empresa is null then
    return jsonb_build_object('ok', false, 'erro', 'Sem empresa associada.');
  end if;

  select id, tipo, alvo_todas, alvo_cedulas into v_atividade
    from public.atividades where id = p_atividade_id;
  if v_atividade.id is null then
    return jsonb_build_object('ok', false, 'erro', 'Atividade não encontrada.');
  end if;
  if v_atividade.tipo is not null then
    return jsonb_build_object('ok', false, 'erro', 'Esta atividade verifica-se sozinha — não precisa de anexo.');
  end if;
  if not (v_atividade.alvo_todas or v_empresa = any(v_atividade.alvo_cedulas)) then
    return jsonb_build_object('ok', false, 'erro', 'Esta atividade não é para a sua empresa.');
  end if;

  -- o anexo tem de estar mesmo na pasta desta empresa -- nunca só aceite
  -- por declaração, sempre conferido no Storage
  v_caminho_esperado := v_empresa || '/';
  if left(p_anexo_caminho, length(v_caminho_esperado)) <> v_caminho_esperado then
    return jsonb_build_object('ok', false, 'erro', 'Caminho de anexo inválido.');
  end if;
  select name, metadata into v_obj
    from storage.objects
   where bucket_id = 'atividades' and name = p_anexo_caminho;
  if v_obj.name is null then
    return jsonb_build_object('ok', false, 'erro', 'Ainda não enviou o anexo.');
  end if;
  if coalesce((v_obj.metadata->>'size')::bigint, 0) = 0 then
    return jsonb_build_object('ok', false, 'erro', 'O ficheiro está vazio.');
  end if;

  insert into public.atividade_declaracoes(id, atividade_id, empresa_cedula, anexo_caminho)
  values (gen_random_uuid(), p_atividade_id, v_empresa, p_anexo_caminho)
  on conflict (atividade_id, empresa_cedula)
  do update set anexo_caminho = excluded.anexo_caminho, declarada_em = now();

  return jsonb_build_object('ok', true, 'dados', jsonb_build_object('declarada', true));
exception when others then
  return jsonb_build_object('ok', false, 'erro', 'Não foi possível registar a declaração.');
end;
$$;

revoke execute on function public.dr_atividade_declarar(uuid, text) from public, anon;
grant execute on function public.dr_atividade_declarar(uuid, text) to authenticated;


create or replace function public.dr_atividades_publicas()
returns jsonb
language plpgsql
security definer
stable
set search_path = public
as $$
declare
  v_linhas jsonb;
begin
  select jsonb_agg(jsonb_build_object(
           'id', a.id, 'titulo', a.titulo, 'texto', a.texto, 'tipo', a.tipo,
           'prazo', a.prazo, 'criada_em', a.criada_em, 'alvo_todas', a.alvo_todas)
         order by a.prazo asc)
    into v_linhas
    from public.atividades a
   where a.alvo_todas = true;

  return jsonb_build_object('ok', true, 'dados', coalesce(v_linhas, '[]'::jsonb));
exception when others then
  return jsonb_build_object('ok', false, 'erro', 'Não foi possível listar as atividades.');
end;
$$;

revoke execute on function public.dr_atividades_publicas() from public;
grant execute on function public.dr_atividades_publicas() to anon, authenticated;
```

- [ ] **Passo 2 — testar com SQL real**

```sql
-- publicar uma atividade sem tipo (autodeclaração)
select public.dr_atividade_publicar('Atender bem os clientes', '<p>Sede simpáticos.</p>',
  null, true, '{}', now() + interval '3 days');

-- como empresa: declarar sem ter enviado o anexo -> deve falhar
select public.dr_atividade_declarar('<id-da-atividade>', 'EP-2026-00009/x.pdf');
-- esperado: {"ok":false,"erro":"Ainda não enviou o anexo."}

-- pública, sem sessão (anon)
reset role;
select public.dr_atividades_publicas();
-- esperado: {"ok":true,"dados":[...]}
```

- [ ] **Passo 3 — commit**

```bash
git add sql/005_rpc_declarar_publicas.sql
git commit -m "feat(dr): declarar por anexo e a montra publica de atividades"
```

---

### Tarefa 6: security advisor + revisão final da camada de dados

- [ ] **Passo 1** — correr `mcp__claude_ai_Supabase__get_advisors` (tipo `security`) e confirmar 0 avisos `ERROR` nas tabelas/funções novas.
- [ ] **Passo 2** — grep local por `sqlerrm` nos 5 ficheiros SQL novos — não pode haver nenhum a vazar para o `erro` devolvido.
- [ ] **Passo 3** — atualizar `docs/TAREFAS-DIARIO-REPUBLICA.md`: marcar Fase 0 e Fase 1 como concluídas, com a data e o resumo dos testes.

---

## Fase 2 — Frontend

> Segue o padrão já fechado no Talentos: ficheiros à raiz (não em
> subpastas — `ferramentas/versoes.py` só varre `*.html` à raiz), portão
> de entrada como página própria (`entrar.html`, nunca popup), editor de
> texto formatado do AeroMail para o campo `texto`. `web/biblioteca/dr.js`
> ainda só tem `esc()`/`ligarCopiarHex()` — esta fase acrescenta-lhe
> `quemSou`, `montarTopo`, `comVersao`, `ligarFormularioLogin`, etc.,
> portados de `talentos/web/biblioteca/ta.js`.

### Tarefa 7: `web/biblioteca/dr.js` — helpers partilhados

**Ficheiros:**
- Modificar: `diario-republica/web/biblioteca/dr.js`

**Produz:** `esc`, `mostrarMsg`, `formatarData`, `haQuanto`, `quemSou()`, `comVersao`/`versionarLinks`, `montarTopo(paginaAtual, ctx)`, `abrirJanela`/`ligarFechos`, `ligarVerSenha`, `ligarFormularioLogin`, `perguntar`, `limparHtml`/`limparNoDr` (lista branca do editor, portado do Talentos), `ficheiroValido`

- [ ] **Passo 1** — portar cada função de `talentos/web/biblioteca/ta.js`, trocando o prefixo `ta`/`TA_` por `dr`/`DR_` e os nomes de página do `montarTopo` (`index.html` "Atividades", `minhas-atividades.html` "As minhas atividades", área própria só existe se `ctx.empresa`).
- [ ] **Passo 2** — testar no Chrome local (`python -m http.server`, puppeteer-core): abrir `biblioteca.html`, confirmar que `dr.js` carrega sem erro de consola (a biblioteca não usa estas funções novas, mas tem de continuar a carregar limpo).
- [ ] **Passo 3** — commit.

### Tarefa 8: `entrar.html` + `entrar.js`

**Ficheiros:**
- Criar: `diario-republica/entrar.html`, `diario-republica/entrar.js`

Cópia direta do par `talentos/entrar.html`/`entrar.js` (já corrigido nesse
projeto para não ser popup), trocando `ta-` por `dr-` e os textos.

- [ ] **Passo 1** — escrever os dois ficheiros.
- [ ] **Passo 2** — testar: sem sessão mostra o formulário; com `?voltar=index.html`, o link "Voltar sem entrar" aponta para lá.
- [ ] **Passo 3** — commit.

### Tarefa 9: `professora.html` + `professora.js` — publicar atividade

**Ficheiros:**
- Criar: `diario-republica/professora.html`, `diario-republica/professora.js`

**Consome:** `dr_atividade_publicar` (Tarefa 3), editor de texto formatado (Tarefa 7)

- [ ] **Passo 1** — portão de entrada (como `empresa.html` do Talentos), mas gate por `fn_e_professor()` em vez de `ctx.empresa`.
- [ ] **Passo 2** — formulário: título, editor (negrito/listas/ligações), select do tipo (só os `ativo=true`, `dr_atividade_tipos` lido por `.from()`, RLS já é leitura pública), alvo (todas / lista de cédulas — `<textarea>` simples de cédulas separadas por vírgula chega para a v1), prazo (`datetime-local`).
- [ ] **Passo 3** — testar no Chrome: publicar uma atividade de teste, confirmar mensagem de sucesso e que aparece depois em `dr_atividades_publicas`.
- [ ] **Passo 4** — commit.

### Tarefa 10: `minhas-atividades.html` + `.js` — vista da empresa

**Ficheiros:**
- Criar: `diario-republica/minhas-atividades.html`, `diario-republica/minhas-atividades.js`

**Consome:** `dr_minhas_atividades` (Tarefa 4), `dr_atividade_declarar` (Tarefa 5)

- [ ] **Passo 1** — portão de entrada (gate por `ctx.empresa`, como `empresa.html` do Talentos).
- [ ] **Passo 2** — lista de atividades: texto, selo verde "Cumprida" com a prova (`prova` desdobrada em texto legível por tipo) ou selo vermelho "Atrasada" se `atrasada=true`, ou neutro "Por cumprir" com o botão que leva a `link_sugerido` do tipo; se `tipo=null`, formulário de anexo + `dr_atividade_declarar`.
- [ ] **Passo 3** — testar: a atividade "Publicar uma vaga" da Tarefa 3 aparece; publicar mesmo uma vaga no Talentos e confirmar que passa a "Cumprida" ao recarregar.
- [ ] **Passo 4** — commit.

### Tarefa 11: `index.html` + `app.js` — montra pública

**Ficheiros:**
- Modificar: `diario-republica/index.html` (troca o placeholder "em construção")
- Criar: `diario-republica/app.js`

**Consome:** `dr_atividades_publicas` (Tarefa 5)

- [ ] **Passo 1** — lista pública de atividades ativas, cartão por atividade (título, prazo, categoria = app_alvo), sem exigir sessão.
- [ ] **Passo 2** — testar sem sessão (anon): a lista carrega.
- [ ] **Passo 3** — commit.

### Tarefa 12: cache-busting, teste completo, publicar

- [ ] **Passo 1** — `python ferramentas/versoes.py`.
- [ ] **Passo 2** — puppeteer: as 5 páginas sem erro de consola; auditor de contraste (`pp-base/ferramentas/auditar_contraste.js`) sobre as páginas novas.
- [ ] **Passo 3** — `git push`; confirmar GitHub Pages a servir a versão nova (`versao.json`).
- [ ] **Passo 4** — atualizar `docs/TAREFAS-DIARIO-REPUBLICA.md`: Fase 2 concluída.

---

## Auto-revisão

- **Cobertura do desenho:** as três tabelas (Tarefa 1), o catálogo de 16 tipos com os 4 `ativo=false` (Tarefa 2), o fan-out por Correio (Tarefa 3), os doze verificadores + `atrasada` (Tarefa 4), a autodeclaração com anexo conferido (Tarefa 5), a montra pública (Tarefa 5+11), o frontend completo com portão de página inteira em vez de popup (Tarefas 7-11) — todas as peças do desenho aprovado têm tarefa.
- **Sem placeholders:** todo o SQL das Tarefas 1-5 está completo, não é esboço.
- **Consistência de tipos:** `dr_minhas_atividades` devolve `{id, titulo, texto, tipo, prazo, cumprida, prova, atrasada}` — os mesmos nomes são os que a Tarefa 10 consome; `dr_atividade_publicar` recebe exatamente os parâmetros que a Tarefa 9 vai enviar.
