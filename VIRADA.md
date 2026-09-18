# Virada para o PostgreSQL

Passo a passo para trocar o sistema em arquivos JSON pelo sistema com banco
(branch `feat/postgres-fase1`). Decisões e contexto em `demandas.md` §17 e §18.

Ensaiado localmente em 2026-09-15: importação de uma cópia da pasta de dados,
boot do app contra o banco importado, prompts e critérios semeados sozinhos.

---

## 0. O que muda e o que não muda

- **Vai para o banco:** contas, logs, progresso, MMR, duelos, prompts, filas,
  notificações, conquistas, sidequests, antessala, Processo Seletivo, Comunidade,
  configurações, tags, uso de IA e **os catálogos** (pacientes, exercícios, neuro,
  competências da Trilha — `demandas.md` §20).
- **Continua no volume `/data` (obrigatório):** fotos (pacientes, exercícios,
  pool de avatares, comunidade), detalhe por critério das avaliações, raciocínio
  da Avaliação Independente, transcrições do benchmark.
- **O app roda em UMA instância.** Prompts, catálogos, configurações e
  sidequests têm cópia em memória.

## 1. Antes de mexer em qualquer coisa

1. Avisar os usuários do horário (a janela entre a cópia e a virada perde o que
   for feito no sistema antigo).
2. **Cópia de segurança do volume inteiro**, para fora do Railway:
   ```bash
   railway run bash
   tar czf /tmp/data-backup.tar.gz -C / data
   # trazer o arquivo para a sua máquina (ver DEPLOY.md, "Backup recomendado")
   ```
   É desta cópia que a importação lê. Ela também é o backup dos prompts (§9).
3. Baixar também o export pela tela (Administração → exportar), como segunda cópia.

## 2. Banco

1. Criar o projeto no **Neon**, Postgres **17**.
2. Anotar a connection string (com `?sslmode=require`).

## 3. Importar (ANTES do primeiro boot do app novo)

Na sua máquina, com a cópia do volume descompactada:

```bash
tar xzf data-backup.tar.gz          # cria ./data
DATABASE_URL='postgres://…neon…' node scripts/importar-volume.js ./data
```

- O script aplica as migrações e importa. O banco precisa estar vazio.
- Os catálogos entram junto (`freeplay-characters.json`, `neuro-characters.json`,
  `exercises.json`, `trilha-skills.json`), com a marca de semeado: o boot não
  põe os pacientes de exemplo por cima.
- Leia o relatório: registros ignorados saem com o motivo; contas excluídas que
  ainda eram referenciadas viram lápide; nomes repetidos só na caixa são
  renomeados com o id.
- Deu errado? Corrija e rode de novo com `--limpar` (apaga os dados do banco,
  menos os prompts).

## 3b. Prompts atualizados (textos que se ajustam à quantidade de critérios)

Os prompts não estão no git. Os textos do v34 foram atualizados na máquina do
Paulo (`avaliacao/` e `server/data/prompts/`, §20.1 de `demandas.md`) e só chegam
à produção pelo volume:

1. **Antes de copiar, compare** com os prompts do volume de produção
   (`/data/prompts/avaliacao/...`). Se algum foi editado em produção depois da
   cópia local, junte as duas edições antes — copiar por cima perderia a de
   produção.
2. Copie os `.md` atualizados para `/data/prompts/avaliacao/` (mesmos caminhos).
3. No boot, a semeadura (§20.2) insere o que falta e **atualiza o prompt que
   ninguém editou pelo painel**, com a versão anterior no histórico. No log:
   `[prompts] atualizado(s) pela semente …`. Prompt editado pelo painel aparece em
   `[prompts] editado(s) pelo admin, semente ignorada …` e fica como está.

Alternativa sem mexer no volume: depois do boot, colar os textos em
Administração → Prompts.

## 4. App novo no Railway

1. Projeto novo, apontando para a branch com o banco.
2. **Montar o MESMO volume em `/data`** (ou restaurar a cópia nele).
3. Variáveis: as de hoje (`JWT_SECRET` igual ao atual, `ADMIN_INITIAL_PASSWORD`,
   chaves de IA, VAPID, Graph, Turnstile, `APP_BASE_URL`) **mais** `DATABASE_URL`.
4. **Uma réplica só.**
5. Subir. No log do boot deve aparecer `[prompts] N prompt(s) semeado(s) no banco.`
   Não deve aparecer `[catalogo] semeado(s) no banco` — os catálogos já vieram da
   importação. Se aparecer, a importação não trouxe algum catálogo: confira o
   relatório do passo 3.

## 5. Domínio e Cloudflare

1. Domínio próprio atrás do Cloudflare (proxy laranja ligado).
2. **Não divulgar o domínio `*.up.railway.app`.**
3. Logado como admin, abrir `/api/admin/diagnostico-ip` **pelo domínio próprio**:
   - `conexaoEhCloudflare: true` → tudo certo, os limites de tentativa usam o IP real.
   - `false` → o Railway está escondendo o IP do Cloudflare. Os limites estariam
     contando todo mundo como um IP só. Definir `CONFIAR_CF_CONNECTING_IP=sempre`
     e, nesse caso, **desativar o domínio do Railway** (por ele o cabeçalho é forjável).

## 6. Conferência depois do boot

- [ ] Login do admin e de um aluno real.
- [ ] Administração → Prompts: os prompts de produção estão lá, e o painel
      "Critérios da régua" não mostra aviso de quantidade escrita à mão.
- [ ] Pacientes, casos de neuro, exercícios e competências da Trilha: os mesmos
      de antes, com as fotos.
- [ ] Um atendimento de Treinamento avaliado do começo ao fim.
- [ ] Ranking e Logs com os dados importados.
- [ ] Cadastro de teste recebe o e-mail de confirmação (Graph) e mostra o captcha.
- [ ] Processo Seletivo: dashboard com o histórico.
- [ ] Comunidade: um link antigo de discussão abre a mesma discussão.

## 7. Se precisar voltar

O sistema antigo continua no projeto antigo do Railway, com o volume intacto
(a importação só lê). Voltar = apontar o domínio de novo para ele. O que foi
feito no sistema novo depois da virada não volta junto.
