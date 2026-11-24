# Quiz Claude Code

Quiz web de **Verdadeiro ou Falso** sobre Claude Code para treinamento interno. A especificação completa está em [prd.md](prd.md).

## Requisitos

- Node.js 24+
- npm 11+

## Primeiros passos

```bash
npm install
npm run dev:local
```

`dev:local` sobe o app **e** uma API local que imita o Supabase (ranking e histórico funcionando, sem conta nem Docker). Os dados ficam em memória e somem ao parar. O domínio de e-mail aceito localmente é `empresa.com.br` ([supabase/seed.sql](supabase/seed.sql)).

Para rodar só o front (sem ranking): `npm run dev`.

## Scripts

| Script | O que faz |
|--------|-----------|
| `npm run dev` | Servidor de desenvolvimento (Vite), sem ranking |
| `npm run dev:local` | App + API local do ranking (PGlite) |
| `npm run dev:api` | Só a API local, em `http://localhost:54321` |
| `npm run build` | Valida as perguntas, faz typecheck e gera `dist/` |
| `npm run preview` | Serve o build de produção localmente |
| `npm run lint` | Lint com oxlint |
| `npm run typecheck` | Checagem de tipos (`tsc -b`) |
| `npm test` | Testes do front, do motor e do SQL do Supabase (Vitest + PGlite) |
| `npm run validate:questions` | Valida `src/data/questions.json` |
| `npm run review:export` | Gera o checklist de revisão em `docs/questions-review.md` |
| `npm run review:mark -- <ids>` | Marca perguntas como revisadas (`--undo` desmarca) |

## Banco de perguntas

As perguntas ficam em [src/data/questions.json](src/data/questions.json) e seguem o schema em [src/data/schema.ts](src/data/schema.ts). Alterações entram por Pull Request.

O validador tem dois modos, definidos pela variável `QUESTIONS_MODE`:

- `draft` (padrão): valida schema, IDs e prefixos; contagem por nível, equilíbrio V/F e revisão viram avisos.
- `release`: exige 20 perguntas por nível, entre 40% e 60% de "Verdadeiro" por nível e `reviewed: true` em todas. Use em produção (Vercel → Production → `QUESTIONS_MODE=release`).

### Revisão das perguntas

As 60 perguntas foram geradas a partir da documentação oficial (`code.claude.com/docs`), cada uma com a página de origem em `sourceUrl`, e começam com `reviewed: false`. O especialista:

1. Abre [docs/questions-review.md](docs/questions-review.md) e confere cada pergunta contra a fonte.
2. Corrige o que for preciso em `src/data/questions.json`.
3. Marca as aprovadas: `npm run review:mark -- beg-001 beg-002 ...`
4. Regenera o checklist (`npm run review:export`) e abre um Pull Request.

Quando todas estiverem revisadas, troque `QUESTIONS_MODE` para `release` no CI e na produção da Vercel.

## Ranking e histórico (Supabase)

### Como funciona

- O banco está em [supabase/migrations/](supabase/migrations/). As tabelas têm RLS ligado e **nenhuma** policy: o navegador não lê nem grava tabelas diretamente.
- O front chama 3 funções RPC via PostgREST (`fetch`, sem `supabase-js`):
  - `submit_attempt(email, nome, tentativa)`: valida o domínio do e-mail, **recalcula pontuação e nível no servidor** e grava. É idempotente por `clientAttemptId`.
  - `get_leaderboard('all' | 'month')`: top 50 pela melhor tentativa de cada pessoa. Nunca devolve e-mails.
  - `get_my_attempts(email)`: histórico de um e-mail.
- Sem conexão, a tentativa fica numa fila no navegador e é reenviada ao abrir o app ou quando a rede volta.
- Sem Supabase configurado, o quiz funciona normalmente e o histórico fica só no navegador.

### Configurar um projeto Supabase

1. Crie o projeto e aplique a migração (com a [Supabase CLI](https://supabase.com/docs/guides/cli): `supabase link` e `supabase db push`, ou cole o SQL no SQL Editor).
2. Cadastre o(s) domínio(s) corporativo(s), editando [supabase/seed.sql](supabase/seed.sql) ou direto no SQL Editor:
   ```sql
   insert into public.allowed_domains (domain) values ('suaempresa.com.br');
   ```
3. Na Vercel (e no `.env.local`), defina:
   - `VITE_SUPABASE_URL`: URL do projeto
   - `VITE_SUPABASE_ANON_KEY`: chave pública (anon ou publishable). **Nunca** a service role.
   - `VITE_ALLOWED_EMAIL_DOMAINS`: os mesmos domínios, separados por vírgula (só para a mensagem no formulário)
4. Retenção de 12 meses (PRD 5): habilite a extensão `pg_cron` no painel e agende a limpeza:
   ```sql
   select cron.schedule('purge-quiz', '0 3 * * *', 'select private.purge_expired_data()');
   ```

### Testes do SQL

`supabase/ranking.test.ts` roda a migração real num Postgres em WebAssembly (PGlite), com os papéis `anon` e `authenticated` do Supabase. Ele cobre o recálculo da pontuação, a idempotência, os desempates, o filtro do mês, as validações e o bloqueio de acesso direto às tabelas.

## Estrutura

```
src/
  app/          rotas, App, ThemeProvider
  components/   Layout, ThemeToggle, LanguageSwitcher, ui
  pages/        Home, Identify, Quiz, Result, Review, Leaderboard, History
  features/     quiz (motor + provider), profile (identificação), ranking (API, fila offline)
  data/         questions.json, schema e validação
  i18n/         pt-BR.json, en.json, configuração do i18next
  lib/          storage seguro, formatação
scripts/        validação/revisão de perguntas, API local (dev-api.ts)
supabase/       migração, seed e testes do SQL
```

## Deploy

Vercel, com `vercel.json` já configurado (rewrite de SPA). Preview por PR com `QUESTIONS_MODE=draft`; produção com `QUESTIONS_MODE=release`.

## Status

- [x] **F1 — Fundação:** Vite + React + TS, Tailwind, tema claro/escuro, i18n pt-BR/en, rotas, schema e validador de perguntas, CI
- [x] **F2 — Motor do quiz:** sorteio progressivo, pontuação e nível, identificação, quiz com feedback imediato e atalhos de teclado, resultado e revisão
- [ ] **F3 — Conteúdo:** 60 perguntas pt-BR/en geradas a partir da documentação oficial e checadas contra as fontes; **aguardando revisão do especialista** ([checklist](docs/questions-review.md))
- [x] **F4 — Ranking e histórico:** migração Supabase com RLS e RPCs, fila offline, telas de Ranking e Histórico, API local para desenvolvimento. **Falta:** criar o projeto Supabase real e cadastrar os domínios.
- [ ] F5 — Badge e polimento
# QuizClaudeCode
