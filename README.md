# FACULTECH EDUCACIONAL

Plataforma SaaS de gestão acadêmica (CRM, AVA, matrículas, financeiro, BI) construída com **TanStack Start** (React 19 + SSR), **Supabase** (autenticação) e **shadcn/ui** + **Tailwind CSS v4**.

Este documento mostra, passo a passo, como **colocar o sistema no ar**.

---

## 1. O que já está pronto e o que ainda falta

**Funciona hoje:**
- Site institucional (página inicial), login e cadastro de usuários.
- Autenticação real por e-mail/senha via Supabase, com criação automática de perfil.
- Painel completo com mais de 25 telas: dashboard, alunos, cursos, matrículas, professores, polos, financeiro, secretaria, provas, biblioteca, CRM, relatórios etc.
- **Dados reais no banco:** as telas de Alunos, Cursos, Matrículas, Professores e Polos (listas e páginas de detalhe) leem e exibem dados das tabelas do Supabase, com indicadores (cards) calculados a partir dos dados reais e estados de carregamento/erro/vazio.
- Build de produção gerando um servidor Node pronto para hospedar.

**Importante saber:** os botões de criação/edição ("Novo aluno", "Novo curso" etc.) e algumas telas auxiliares (financeiro, secretaria, provas, biblioteca, CRM) ainda usam conteúdo ilustrativo. O núcleo (alunos/cursos/matrículas/professores/polos) já é dado real — veja a seção 6 para os próximos passos.

---

## 2. Pré-requisitos

- **Node.js 20 ou superior** — https://nodejs.org
- Uma conta no **Supabase** — https://supabase.com (plano gratuito serve)
- Uma conta na plataforma de hospedagem escolhida (Render, Railway, Vercel, Cloudflare ou um servidor próprio)

---

## 3. Configurar o Supabase

O projeto precisa de um backend Supabase para autenticação.

1. Crie um projeto em https://supabase.com.
2. Vá em **Project Settings → API** e copie:
   - **Project URL** (ex.: `https://abcd1234.supabase.co`)
   - **anon / publishable key** (começa com `sb_publishable_...` ou `eyJ...`)
3. Aplique a estrutura do banco. No painel do Supabase, abra **SQL Editor** e rode o conteúdo dos arquivos, **na ordem**:
   - `supabase/migrations/20260531192906_*.sql` (cria a tabela `profiles`, RLS e o gatilho que cria o perfil no cadastro)
   - `supabase/migrations/20260531192922_*.sql`
   - `supabase/migrations/20260601120000_facultech_core_tables.sql` (cria `alunos`, `cursos`, `matriculas`, `professores`, `polos`, com RLS e os dados iniciais já populados)
4. Em **Authentication → URL Configuration**, adicione o endereço onde o site vai rodar (ex.: `https://seu-app.onrender.com`) tanto em **Site URL** quanto em **Redirect URLs**. Sem isso, a confirmação de e-mail e o login social falham.

> **Sobre segurança das chaves:** a chave `publishable`/`anon` foi feita para ser pública (vai para o navegador). O que protege os dados é o **Row Level Security (RLS)**, já ativado nas migrations. **Nunca** coloque a `service_role` key com o prefixo `VITE_` nem a exponha no front-end.

---

## 4. Rodar localmente (teste antes de publicar)

```bash
# 1. Instale as dependências
npm install

# 2. Crie o arquivo .env a partir do modelo e preencha com seus dados Supabase
cp .env.example .env
#   (edite o .env com a URL e a chave do passo 3)

# 3. Ambiente de desenvolvimento (hot reload)
npm run dev
#   abre em http://localhost:3000

# 4. Testar o build de produção localmente
npm run build
npm start
#   sobe o servidor de produção em http://localhost:3000
```

---

## 5. Publicar (escolha UMA opção)

O build padrão (`npm run build`) gera um **servidor Node** em `dist/`:
- `dist/server/index.mjs` → o servidor
- `dist/client/` → os arquivos estáticos
- Comando para iniciar: `node dist/server/index.mjs` (respeita a variável `PORT`)

### Opção A — Render (mais simples, tem plano gratuito)

1. Suba este projeto para um repositório no GitHub.
2. Em https://render.com → **New → Blueprint** e aponte para o repositório (o arquivo `render.yaml` já está incluído).
3. Quando pedir, preencha as variáveis de ambiente (as mesmas do seu `.env`):
   `SUPABASE_URL`, `SUPABASE_PUBLISHABLE_KEY`, `VITE_SUPABASE_URL`, `VITE_SUPABASE_PUBLISHABLE_KEY`, `VITE_SUPABASE_PROJECT_ID`.
4. Confirme. O Render roda `npm ci && npm run build` e inicia com `node dist/server/index.mjs`.

> Alternativa sem Blueprint: **New → Web Service**, Build Command `npm ci && npm run build`, Start Command `node dist/server/index.mjs`.

### Opção B — Railway

1. https://railway.app → **New Project → Deploy from GitHub**.
2. Em **Variables**, adicione as mesmas variáveis de ambiente acima.
3. Em **Settings**: Build `npm ci && npm run build`, Start `node dist/server/index.mjs`. Pronto.

### Opção C — Docker (qualquer VPS, Fly.io, etc.)

Um `Dockerfile` já está incluído.

```bash
docker build \
  --build-arg VITE_SUPABASE_URL="https://SEU_PROJETO.supabase.co" \
  --build-arg VITE_SUPABASE_PUBLISHABLE_KEY="sb_publishable_..." \
  --build-arg VITE_SUPABASE_PROJECT_ID="SEU_PROJETO" \
  -t facultech .

docker run -p 3000:3000 --env-file .env facultech
```

### Opção D — Cloudflare Workers (alvo nativo do Lovable)

```bash
NITRO_PRESET=cloudflare-module npm run build
```

Isso gera a saída no formato Cloudflare. Depois, instale e use o Wrangler (`npm i -D wrangler`, `npx wrangler deploy`) configurando as variáveis como *secrets* do Worker. As variáveis `VITE_*` precisam estar presentes no momento do build.

> **Atenção (login com Google):** o botão "Continuar com Google" usa o serviço de autenticação do Lovable e pode só funcionar em domínios hospedados pelo Lovable. O login por **e-mail/senha funciona em qualquer hospedagem**. Para ter o Google fora do Lovable, configure o provedor Google diretamente em **Authentication → Providers** no Supabase e ajuste a chamada de OAuth.

---

## 6. Próximos passos para evoluir o sistema

O núcleo já roda com dados reais. Para completar o ciclo de gestão:
1. **Formulários de criação/edição:** ligar os botões "Novo aluno", "Novo curso", "Nova matrícula" etc. a `INSERT`/`UPDATE` no Supabase. A camada de dados em `src/lib/api/queries.ts` já está pronta para receber mutations do TanStack Query (`useMutation` + `queryClient.invalidateQueries`).
2. **Demais módulos:** financeiro, secretaria, provas, biblioteca e CRM ainda usam conteúdo ilustrativo. Cada um pode ganhar suas próprias tabelas seguindo exatamente o mesmo padrão (tabela + RLS na migration, hook em `queries.ts`, consumo na rota).
3. **Multi-instituição (opcional):** adicionar uma coluna `institution_id` nas tabelas e ajustar as políticas de RLS para isolar os dados por instituição.
4. **Busca e paginação:** os campos de busca nas listas são visuais; dá para ligá-los a `.ilike()` e `.range()` do Supabase.

### Como os dados reais funcionam (referência rápida)
- Schema e dados iniciais: `supabase/migrations/20260601120000_facultech_core_tables.sql`
- Tipos TypeScript do banco: `src/integrations/supabase/types.ts`
- Hooks de leitura (TanStack Query): `src/lib/api/queries.ts`
- Estados de UI (carregando/vazio/erro): `src/components/DataStates.tsx`
- As páginas de detalhe casam pela coluna `slug` (cursos, professores, polos), `ra` (alunos) ou `proto` (matrículas).

---

## 7. Estrutura do projeto

```
src/
  routes/               # Páginas (TanStack Router por arquivos)
    index.tsx           # Site institucional
    login.tsx, signup.tsx
    _authenticated.tsx  # "Portão" de login + layout do painel
    _authenticated/     # Telas internas (dashboard, alunos, cursos, ...)
  components/           # AppSidebar, Logo, componentes de UI (shadcn)
  integrations/supabase # Cliente Supabase, middlewares de auth, tipos do banco
  lib/                  # auth-context, demo-data, utilitários
    api/queries.ts      # Hooks de leitura (TanStack Query) das tabelas
  components/DataStates.tsx # Estados de carregando/vazio/erro
supabase/migrations/    # Estrutura do banco (rodar no SQL Editor)
vite.config.ts          # Config de build (preset de deploy via NITRO_PRESET)
Dockerfile, render.yaml # Infra de deploy
```

---

## 8. Resolução de problemas

| Sintoma | Causa provável | Solução |
|---|---|---|
| Erro "Missing Supabase environment variable" | Variáveis não definidas na hospedagem | Configure as 5 variáveis no painel do serviço |
| Login funciona local mas não em produção | URL não autorizada no Supabase | Adicione o domínio em Authentication → URL Configuration |
| Tela branca após o build | Variáveis `VITE_*` ausentes no build | Elas precisam existir **no momento do build**, não só no runtime |
| Botão do Google não entra | OAuth do Lovable restrito ao domínio | Use e-mail/senha ou configure o Google direto no Supabase |
