# O MMR do all_OS — documentação completa

O MMR é a pontuação competitiva da plataforma. Ele existe para responder a uma
pergunta que a nota do atendimento, sozinha, não responde:

> Um 70 num caso difícil vale mais que um 70 num caso fácil. **Quanto mais?**

O motor inteiro está em [`server/mmr.js`](server/mmr.js) — **funções puras**, sem
banco e sem rede. A persistência fica em [`server/repos/mmr.js`](server/repos/mmr.js).
Essa separação é deliberada: a regra pode ser testada sem subir nada, e é por
isso que existem 67 casos de teste cobrindo o assunto.

**Índice**

1. [O que o sistema estima](#1-o-que-o-sistema-estima)
2. [Uma partida, passo a passo](#2-uma-partida-passo-a-passo)
3. [Exemplo numérico completo](#3-exemplo-numérico-completo)
4. [Calibração](#4-calibração--as-3-primeiras-partidas)
5. [A regressão do paciente](#5-a-regressão-do-paciente)
6. [Duelo (PvP)](#6-duelo-pvp)
7. [Candidatos e visitantes](#7-candidatos-e-visitantes-a-camada-anônima)
8. [Onde isso é guardado](#8-onde-isso-é-guardado)
9. [Concorrência](#9-concorrência-por-que-há-transação-e-trava)
10. [O que aparece na tela](#10-o-que-aparece-na-tela)
11. [Tabela de constantes](#11-tabela-de-constantes)
12. [Cobertura de testes](#12-cobertura-de-testes)
13. [Decisões e histórico](#13-decisões-e-histórico)
14. [Perguntas frequentes](#14-perguntas-frequentes)

---

## 1. O que o sistema estima

O MMR não mede só o aluno. Ele estima **duas grandezas ao mesmo tempo**, cada uma
corrigindo a outra:

| Símbolo | O que é | Onde aparece | Início | Faixa |
|---|---|---|---|---|
| **P** | Habilidade do terapeuta | o MMR do Ranking | 50 | sem teto |
| **D** | Dificuldade do paciente simulado | ficha do paciente | 50 | 10 a 90 |

A ideia é a mesma do Elo (xadrez) e da TRI — Teoria de Resposta ao Item, a mesma
família de modelos usada no ENEM: **a dificuldade de um item não é uma opinião,
é uma medida**, que sai do desempenho corrigido pelo nível de quem respondeu.

O estado do jogador tem mais duas peças:

```js
{ P: 50, n: 0, W: [] }
```

- **`n`** — quantas partidas já concluiu. Controla a velocidade de ajuste.
- **`W`** — a janela das **20 partidas mais recentes**. Cada entrada guarda
  `{ S_aj, D, P }`. É o que dá memória ao sistema.

E o do paciente:

```js
{ D: 50, n_D: 0, alpha: null, beta: null, history: [] }
```

- **`n_D`** — quantas partidas de fato ajustaram a dificuldade dele.
- **`alpha` / `beta`** — os coeficientes da regressão própria dele (§5).
- **`history`** — até **200** pontos `{ P, D, S }` para alimentar a regressão.

---

## 2. Uma partida, passo a passo

A entrada é a nota **S** (0 a 100) do atendimento avaliado. Quem calcula essa
nota é `server/scoring.js` / o agregador do pipeline v34 — **não é a IA que emite
a nota final**; a IA emite as notas por critério e a conta é determinística.

O que segue é `updateMatch(player, character, S)`.

### Passo 1 — Quanto se esperava dessa pessoa nesse caso

```
S_esp = 50 + 0,5 × (P − D)
```

Só o **gap** entre habilidade e dificuldade importa. Um aluno 20 pontos acima da
dificuldade do caso "deveria" tirar 60; 20 pontos abaixo, 40.

O coeficiente 0,5 é a inclinação de partida: a cada ponto de vantagem, meio ponto
de nota esperada. Depois de 20 partidas, o paciente ganha uma reta própria (§5).

### Passo 2 — A dificuldade do paciente se move

```
ΔD = dWeight × 0,1 × (S_esp − S)
D  = clamp(D + ΔD, 10, 90)
```

O que move o D é a **surpresa**, não a nota:

| Situação | Leitura | Efeito |
|---|---|---|
| MMR alto, nota baixa | `S_esp > S` | o caso é mais difícil do que se pensava → **D sobe** |
| MMR baixo, nota alta | `S_esp < S` | o caso é mais fácil → **D desce** |
| nota igual à esperada | `S_esp = S` | nada a aprender → **D fica** |

**Por que corrigir pelo nível.** Sem essa correção, um paciente atendido só por
alunos iniciantes pareceria dificílimo — não por ser difícil, mas por causa de
quem o atendeu. É exatamente o viés que o modelo existe para evitar.

Três detalhes que só se veem no código:

- **Durante a calibração do jogador, este passo inteiro é pulado** (§4).
- O que vai para o `history` é o **D de antes do ajuste** — é contra esse D que a
  partida foi de fato jogada, logo é ele que explica a nota obtida.
- `dWeight` escala o ganho. Vale 1 para aluno com conta; menos para população
  anônima (§7).

### Passo 3 — O quanto o MMR pode se mover

```
K = 0,10 + 0,40 × e^(−0,15 × n)
```

`n` é a contagem **antes** desta partida.

| Partida | n | K |
|---|---|---|
| 1ª | 0 | **0,500** |
| 2ª | 1 | 0,444 |
| 4ª | 3 | 0,355 |
| 6ª | 5 | 0,289 |
| 11ª | 10 | 0,189 |
| 21ª | 20 | 0,120 |
| ∞ | — | **0,100** (assíntota) |

Novato se move rápido, porque o sistema ainda não sabe quem ele é. Veterano se
move devagar: um dia ruim não apaga um histórico longo.

### Passo 4 — A nota ajustada pela dificuldade

```
S_aj = S + (50 − S_esp)
```

**Este é o coração do sistema.** A nota bruta é convertida em *"quanto acima ou
abaixo do esperado essa pessoa foi, naquele caso"*, recentrada em 50.

| Nota obtida | Esperada | S_aj | Leitura |
|---|---|---|---|
| 70 | 50 | **90** | muito acima do esperado |
| 70 | 70 | **50** | exatamente o esperado |
| 70 | 85 | **35** | abaixo do esperado, apesar do 70 |

Não há `clamp` aqui, de propósito: um desempenho extremo deve aparecer como
extremo, e não ser cortado.

### Passo 5 — O novo MMR

```
P = (1 − K) × P_janela + K × S_aj
```

`P_janela` é a média das `S_aj` da janela `W`, com **pesos lineares
crescentes**:

```
w_i = (i + 1) / [ size × (size + 1) / 2 ]
```

com `i = 0` na partida mais antiga. Numa janela cheia de 20, a partida mais
recente pesa **20×** o que pesa a mais antiga, e os pesos somam 1.

Assim o MMR reflete a forma atual sem jogar fora o histórico.

> **Fallback:** na calibração, ou se a janela estiver vazia, a fórmula vira
> `P = (1 − K) × P + K × S_aj` — média exponencial sobre o próprio P.

### Passo 6 — Manutenção

```
W.push({ S_aj, D, P })     // se passar de 20, o mais antigo sai
n += 1
```

---

## 3. Exemplo numérico completo

Aluna com **P = 62**, já com 8 partidas (`n = 8`) e janela cheia cuja média
ponderada é **P_janela = 60**. Ela atende um paciente com **D = 45** e tira
**S = 70**.

**Passo 1 — esperada**

```
gap   = 62 − 45 = 17
S_esp = 50 + 0,5 × 17 = 58,5
```

**Passo 2 — dificuldade**

```
ΔD = 0,1 × (58,5 − 70) = −1,15
D  = 45 − 1,15 = 43,85
```

Ela foi melhor que o esperado → o caso era um pouco mais fácil do que parecia.

**Passo 3 — sensibilidade**

```
K = 0,10 + 0,40 × e^(−0,15 × 8) = 0,10 + 0,40 × 0,3012 = 0,2205
```

**Passo 4 — nota ajustada**

```
S_aj = 70 + (50 − 58,5) = 61,5
```

O 70 bruto "vale" 61,5, porque o caso estava abaixo do nível dela.

**Passo 5 — novo MMR**

```
P = (1 − 0,2205) × 60 + 0,2205 × 61,5
  = 0,7795 × 60 + 0,2205 × 61,5
  = 46,77 + 13,56
  = 60,33
```

**Resultado:** o MMR **caiu** de 62 para 60,3 — mesmo com uma nota 70 e mesmo
tendo ido acima do esperado. Por quê? Porque o que puxa o MMR é a **janela**, que
estava em 60, e o 61,5 desta partida é só um pouco melhor que ela. O número que a
aluna vê é `Math.round(60,33) = 60`.

É contraintuitivo na primeira vez e é o comportamento correto: o MMR é o retrato
das 20 últimas, não da última.

---

## 4. Calibração — as 3 primeiras partidas

Enquanto `n < CALIBRATION_MATCHES` (3), o jogador está em calibração:

1. **O MMR não aparece.** `playerView` devolve `mmr: null` e
   `matchesRemaining`, e a tela mostra quantas partidas faltam.
2. **O MMR se move por média exponencial simples**, sem usar a janela.
3. **A dificuldade dos pacientes não é tocada.**

O item 3 é o mais importante e o menos óbvio. Nas primeiras partidas o P ainda é
o chute inicial de 50, não o nível da pessoa. Se ele entrasse na conta, um aluno
forte recém-chegado derrubaria a dificuldade de todo paciente que encostasse — e
um aluno fraco a inflaria. O sistema espera o P convergir antes de confiar nele
como régua.

A mesma regra vale para as populações anônimas (§7).

---

## 5. A regressão do paciente

Depois de **20 partidas válidas** (`n_D ≥ 20`), um paciente deixa de usar a reta
genérica e passa a usar a **dele**:

```
S ≈ α + β × gap        , com gap = P − D
```

Ajustada por mínimos quadrados sobre o `history`:

```
β = Σ(gap_i − ḡap)(S_i − S̄) / Σ(gap_i − ḡap)²
α = S̄ − β × ḡap
```

- Refeita **a cada 5 partidas** novas (`n_D % 5 == 0`).
- Recusa ajustar se houver menos de 2 pontos, ou se o gap for praticamente
  constante (`Σ(gap − ḡap)² < 1e−9`) — nesse caso a reta seria indefinida e o
  paciente continua com a genérica.
- O `history` tem teto de **200** pontos; o mais antigo sai.

O ganho prático: um paciente pode ser "difícil para iniciante e fácil para
veterano" (β alto) ou ter desempenho parecido em todos os níveis (β baixo). A
reta genérica não captura isso; a própria, sim.

---

## 6. Duelo (PvP)

O MMR é **único** — não existe MMR de duelo separado. Num duelo, os dois alunos
atendem o **mesmo paciente** e recebem notas independentes.

### As travas, antes de tudo

| Trava | Regra | O que impede |
|---|---|---|
| **Calibração** | os **dois** precisam ter `n ≥ 3` | inflar MMR contra conta recém-criada |
| **Anti-smurf** | **nenhum** pode tirar menos de **25** | perder de propósito para transferir MMR |

Duelo reprovado devolve `{ ranked: false, reason: 'calibrating' | 'anti_smurf' }`.
**O feedback dos dois alunos acontece do mesmo jeito** — só MMR, janela,
contagem e dificuldade ficam intocados.

### A conta

```
aposta_A = 0,20 × P_A
aposta_B = 0,20 × P_B
pool     = aposta_A + aposta_B

frac_A   = S_A / (S_A + S_B)
frac_B   = S_B / (S_A + S_B)

delta_A  = frac_A × pool − aposta_A
delta_B  = frac_B × pool − aposta_B
```

Cada lado aposta **20% do próprio MMR**; o pool é dividido na proporção das
notas. Como `delta_A + delta_B = 0`, o duelo é **soma zero** entre os dois.

### A ordem importa

Depois do cálculo do pool, **cada jogador ainda passa pelo pipeline solo
completo** contra o paciente — primeiro A, depois B, encadeando o estado do
paciente:

```js
const upA = updateMatch(pA, char,           S_A);
const upB = updateMatch(pB, upA.character,  S_B);
```

Ou seja: **a dificuldade do paciente se move como em duas partidas em
sequência**, não em uma. Só depois disso o delta do PvP é somado por cima.

> Se `S_A + S_B = 0` (caso degenerado), a divisão vira 50/50.

---

## 7. Candidatos e visitantes: a camada anônima

Processo Seletivo e visitante também atendem pacientes, e essas notas são
informação valiosa sobre a dificuldade dos casos. Mas essas pessoas **não têm
MMR próprio**: o candidato é efêmero, o visitante recebe um id novo a cada
sessão.

**O erro a evitar.** Se entrassem com rating fixo de 50 e o grupo fosse de fato
mais fraco, o sistema leria as notas baixas como *"paciente difícil"* em vez de
*"respondente mais fraco"*, e empurraria para cima a dificuldade de todos os
casos — precisamente o viés que o modelo existe para eliminar.

**A solução.** Cada população é **um jogador persistente**. "Candidatos do
Processo Seletivo" é um jogador só, que começa em 50 e aprende o próprio nível
agregado ao longo das sessões. O candidato individual vira ruído em torno dessa
média — e a média é justamente o que se quer estimar.

Do ponto de vista do motor é um jogador comum: mesmo `updateMatch`, **inclusive a
calibração** (as 3 primeiras sessões da população não mexem no D, tempo de o
rating dela sair de 50). O que muda é só o `dWeight`, menor, porque sinal de
população é mais ruidoso e merece ganho menor.

### O peso de cada origem

| Origem | Peso | Onde se ajusta |
|---|---|---|
| **Aluno cadastrado** (Competitivo) | **1** | fixo — é a referência |
| **Processo Seletivo** | **0,35** | Administração → Acessos |
| **Visitante** | **0,5** | Administração → Acessos (sem efeito enquanto `VISITOR_TRI` não estiver ligado) |

O peso entra direto no ajuste da dificuldade:

```
ΔD = peso × 0,1 × (S_esp − S)
```

Ou seja, um atendimento do Seletivo move a dificuldade com pouco mais de **um
terço** da força de uma partida de aluno cadastrado. Duas razões:

- **Sinal mais ruidoso** — o rating usado é a média do grupo, não a habilidade
  daquela pessoa.
- **Volume** — o Seletivo tende a ter muito mais atendimentos que o Competitivo.
  Com pesos iguais, a dificuldade dos pacientes passaria a refletir sobretudo
  candidatos, e não alunos.

**Desde 2026-09-23 o peso é do admin**, na tela de Acessos, entre **0 e 1**
(0 desliga aquela população; 1 a iguala ao aluno cadastrado). Antes era só
variável de ambiente, e ajustar exigia deploy — ruim para um parâmetro de
calibração que só se afina com dados reais na mão. As variáveis
`TRI_PESO_SELECAO` e `TRI_PESO_VISITANTE` continuam existindo como **padrão de
fábrica**, para um ambiente novo nascer diferente.

Dois avisos que a própria tela dá:

- O valor novo vale **já no próximo atendimento avaliado** — a configuração é
  lida a cada atendimento, não no boot.
- **Não recalcula** as dificuldades já medidas. Muda só o quanto os próximos
  atendimentos pesam.

**O TRI do Seletivo é aplicado quando a avaliação volta do lote**, não quando o
candidato termina — sem nota não há sinal. E só com nota válida: avaliação que
deu erro não move nada.

**A dificuldade é única e compartilhada.** Competitivo, Seletivo e visitante
alimentam o **mesmo D**. Isso é o ponto de um sistema Elo/TRI: respondentes de
níveis diferentes devem convergir para a mesma estimativa de dificuldade.
Separar por população jogaria fora essa propriedade e devolveria apenas "nota
média por paciente".

---

## 8. Onde isso é guardado

No sistema em arquivos era o `mmr.json`. No PostgreSQL:

| Tabela | Chave | Conteúdo |
|---|---|---|
| `mmr_players` | `user_id` | `estado` JSONB — `{ P, n, W }` |
| `mmr_characters` | `character_id` | `estado` JSONB — `{ D, n_D, alpha, beta, history }` + `fontes` |
| `mmr_anon_players` | `pool` | estado da população anônima |
| `character_records` | `character_id` | o recorde 👑 do paciente no Competitivo |

O estado vai como **documento JSONB** porque é lido e gravado sempre inteiro,
pelo motor, e nunca consultado campo a campo.

`fontes` conta de onde vieram as partidas que moveram a dificuldade daquele
paciente (competitivo, seletivo, visitante) — e só é incrementado quando o motor
de fato mexeu no D, nunca durante a calibração.

Cada log de partida competitiva também guarda `mmrBefore` e `mmrAfter`, o que
permite mostrar o número andando partida a partida.

---

## 9. Concorrência: por que há transação e trava

Duas partidas simultâneas no mesmo paciente, no sistema antigo de arquivos, se
sobrescreviam — era uma das razões de peso para sair do JSON. No banco,
`aplicar()` resolve assim:

1. **Cria a linha antes de travar.** `SELECT ... FOR UPDATE` não trava linha que
   ainda não existe; sem o `INSERT ... ON CONFLICT DO NOTHING` antes, duas
   primeiras partidas simultâneas no mesmo paciente se perderiam.
2. **Trava o paciente e cada jogador** com `FOR UPDATE`, dentro de uma transação.
3. **Trava os jogadores sempre na mesma ordem** (ids ordenados). Sem isso, dois
   duelos cruzados travariam em círculo — *deadlock*.
4. **Grava só o que o cálculo devolveu**, e só para jogador travado ali.

Uma partida trava **apenas o paciente e os jogadores dela** — não o sistema
inteiro. Duas partidas em pacientes diferentes não se esperam.

---

## 10. O que aparece na tela

**`playerView(player)`** — o que o Ranking e o Perfil mostram:

```js
{
  n,                  // partidas jogadas
  calibrating,        // n < 3
  matchesRemaining,   // quantas faltam para sair da calibração
  mmr,                // Math.round(P) — null durante a calibração
  mmrRaw,             // P sem arredondar
}
```

**`characterDifficulty(character)`** — `Math.round(D)`, de 10 a 90. Paciente
nunca jogado mostra a baseline, 50.

**`characterAvgScore(character)`** — a média simples das notas do histórico. É o
número que o supervisor entende de imediato, exibido ao lado do D: a dificuldade
medida e a nota média dizem coisas diferentes, e ver as duas juntas evita
confundi-las.

---

## 11. Tabela de constantes

| Constante | Valor | O que significa |
|---|---|---|
| `P0` | 50 | MMR inicial do jogador |
| `D0` | 50 | dificuldade inicial do paciente |
| `D_MIN` / `D_MAX` | 10 / 90 | limites da dificuldade |
| `WINDOW` | 20 | tamanho da janela de partidas recentes |
| `CALIBRATION_MATCHES` | 3 | partidas em calibração |
| `CHAR_MATURE_AT` | 20 | `n_D` a partir do qual o paciente ganha regressão própria |
| `REGRESS_REFIT_EVERY` | 5 | de quantas em quantas partidas a regressão é refeita |
| `HISTORY_CAP` | 200 | teto do histórico do paciente |
| `PVP_STAKE` | 0,20 | fração do MMR apostada no duelo |
| `PVP_MIN_SCORE` | 25 | nota mínima para o duelo valer MMR |
| peso do Seletivo | 0,35 | ganho do ajuste de D vindo do Processo Seletivo (editável em Acessos) |
| peso do visitante | 0,5 | idem, para o visitante (hoje sem efeito) |
| — | 0,5 | inclinação da reta genérica de nota esperada |
| — | 0,1 | ganho do ajuste de dificuldade |
| — | 0,10 / 0,40 / 0,15 | piso, amplitude e decaimento da sensibilidade `K` |

---

## 12. Cobertura de testes

**67 casos** em cinco arquivos:

| Arquivo | Casos | Cobre |
|---|---|---|
| `tests/mmr.test.js` | 20 | o motor solo: nota esperada, sensibilidade, janela, pesos, calibração |
| `tests/tri-dificuldade.test.js` | 17 | o ajuste de D, inclusive no duelo, na calibração e abaixo do piso anti-smurf |
| `tests/tri.test.js` | 11 | a regressão do paciente e o amadurecimento |
| `tests/db-repo-mmr.test.js` | 11 | persistência, travas e importação |
| `tests/mmr-pvp.test.js` | 8 | pool, distribuição, soma zero e as duas travas do duelo |

Sem contar `tests/duel.test.js` e `tests/db-repo-duelos.test.js`, que cobrem o
fluxo do duelo em volta do motor.

---

## 13. Decisões e histórico

- **Calibração de 5 → 3 partidas (29/05/2026).** Cinco partidas desengajavam o
  aluno antes de o MMR aparecer.
- **O histórico do paciente guarda o D de ANTES do ajuste.** O pseudocódigo
  original gravava o D já ajustado; foi corrigido, porque é contra o D jogado que
  a nota foi obtida — é ele que a explica.
- **Dificuldade compartilhada entre populações**, em vez de um D por origem
  (§7).
- **Sem `clamp` na nota ajustada**, para preservar desempenhos extremos.
- **Peso do TRI virou configuração do admin (23/09/2026)**, em vez de variável
  de ambiente: é calibração, não constante de engenharia.
- **A nota final vem de código, não da IA.** A IA erra a aritmética com
  frequência; a conta de somar critérios e converter para 0–100 é determinística
  e vive em `server/scoring.js`.

---

## 14. Perguntas frequentes

**O MMR pode cair?**
Pode, e cai. Ele é a média ponderada da janela recente; uma sequência ruim
derruba. Não há proteção de piso. Ver o exemplo do §3, em que uma nota 70 acima
do esperado ainda assim baixou o MMR.

**Dá para subir atendendo só casos fáceis?**
Não. Caso fácil tem D baixo → `S_esp` alto → `S_aj` **abaixo** da nota bruta.
Farmar caso fácil rende pouco, por construção.

**E atendendo só casos difíceis?**
Ajuda, se você for bem. Caso difícil tem `S_esp` baixo, então a mesma nota vale
mais. Mas se você for mal nele, cai igual.

**Quem define a dificuldade dos pacientes?**
Ninguém digita esse número. Ela é **medida** a partir das notas obtidas,
corrigidas pelo nível de quem atendeu.

**Por que meu MMR não aparece?**
Faltam partidas para terminar a calibração (3). A tela mostra quantas.

**Por que meu MMR mal se mexeu?**
Quanto mais partidas você tem, menor o `K`. Depois de umas 20, cada partida pesa
cerca de 10%.

**Perdi o duelo e meu MMR não mudou. Bug?**
Provavelmente uma das travas: algum dos dois ainda estava em calibração, ou
alguém tirou menos de 25. O feedback sai mesmo assim.

**Dois alunos empataram no duelo. O que acontece?**
Notas iguais → `frac = 0,5` para os dois → cada um recebe de volta exatamente o
que apostou. Delta zero no PvP; o pipeline solo continua valendo para os dois.

**A nota final sai da IA?**
Não. A IA emite as notas por critério; a nota 0–100 é uma conta determinística
(`server/scoring.js`). O MMR parte dessa nota.
