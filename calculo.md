# Como funcionam os pesos e cálculos (métricas)

Este documento detalha o motor de pontuação da Calculadora de Sinastria: como cada
aspecto/casa vira peso numérico, como os pesos viram percentuais, e como os percentuais
viram os números finais mostrados na tela (Compatibilidade, Harmonia, Nitidez, Veredito,
categorias, eixos e Perfil de Vínculo).

Toda a lógica descrita aqui vive nas funções `computeScores`, `axisBoost`,
`categoryPoolFor`, `axisPoolFor`, `harmonicFraction`, `potentialScore`,
`computeVinculoProfile` e no objeto `CALIBRATION`, dentro de `index.html`.

## Visão geral do pipeline

```
Texto colado
   → parseText()               (extrai aspectos e casas)
   → dedupeAspects() / collapseNodeMirrors() / collapseAxisMirrors()
   → computeScores(parsed, relType)
        para cada aspecto:
          orbW      = peso por orbe (decaimento exponencial)
          hFrac     = fração harmônica do aspecto (0 = tenso puro, 1 = harmônico puro)
          boost     = axisBoost() — múltiplo de peso por tradição astrológica
          categoria = categoryPoolFor() — Intelectual/Emocional/Atração/Prático/Afinidade
          eixo      = axisPoolFor()     — Estrutura/Destino
        → acumula harmoniousW / tenseW por categoria e por eixo
   → agrega em: harmonyPct, strength (Nitidez), categoryScores{},
                structureHarmonyPct, destinyHarmonyPct, compatibilityScore
   → potentialScore()    → Veredito final (0-100)
   → computeVinculoProfile() → rótulo de Perfil de Vínculo (matriz 3×3)
   → classify()          → texto narrativo do veredito
```

## 1. Peso de cada aspecto individual

Cada linha do relatório (ex.: "Sol conjunção Lua, orbe 2°14'") vira um objeto aspecto com
`planet1`, `planet2`, `aspect` (tipo geométrico) e `orb` (grau de exatidão). A partir daí,
três multiplicadores independentes decidem o peso final desse aspecto:

### 1.1 `orbW` — peso por exatidão do orbe

```js
orbW = Math.exp(-orb / ORB_DECAY_DIVISOR[aspect])
```

Decaimento **exponencial**: quanto mais próximo de 0° (aspecto exato), mais peso; quanto
mais largo o orbe, mais rápido o peso cai rumo a zero. O divisor varia por tipo de
aspecto — aspectos "maiores" toleram orbe mais largo sem perder muita força; os menores
enfraquecem mais rápido:

| Aspecto | Divisor |
|---|---|
| Conjunção, Oposição | 3.0 |
| Trígono, Quadratura | 2.5 |
| Sextil, Quincunx | 1.8 |
| Semisextil, Semiquadratura, Sesquiquadratura | 1.5 |

Exemplo: um trígono a 3° de orbe dá `exp(-3/2.5) ≈ 0.30`; a mesma distância numa
conjunção dá `exp(-3/3.0) ≈ 0.37` — a conjunção "aguenta" mais orbe.

### 1.2 `hFrac` — fração harmônica (`harmonicFraction`)

Cada aspecto recebe uma fração de 0 (puramente tenso) a 1 (puramente harmônico) — **não**
apenas pelo tipo geométrico, mas também por *quais planetas* estão envolvidos, porque a
tradição de sinastria trata alguns pares como ambivalentes por natureza, não como
harmônicos ou tensos "de fábrica":

- **Trígono / Sextil** → sempre `1.0`.
- **Quadratura / Oposição** → `0.0` no caso genérico, mas com exceções:
  - Júpiter-Saturno → `0.1` (bloqueio de expansão)
  - Júpiter (outro aspecto duro) → `0.3`
  - Marte-Vênus / Marte-Marte → `0.45` (atração com atrito, não puramente ruim)
  - Vênus com Mercúrio/Sol/Lua → `0.25`
- **Semisextil (30°)** → `0.5` (neutro)
- **Quincunx (150°)** → `0.2` (ajuste crônico, desconfortável)
- **Semiquadratura (45°)** → `0.2`; **Sesquiquadratura (135°)** → `0.15`
- **Conjunção** → o mais variável de todos, com um extenso conjunto de casos
  especiais em ordem de prioridade (cada um documentado no código com a razão
  tradicional), por exemplo:
  - Marte-Vênus/Marte-Marte → `0.65` ("química com fio desencapado")
  - Marte-Plutão, Lilith-Marte → `0.55` (magnetismo/paixão)
  - Marte-Urano, Marte-Mercúrio, Saturno sozinho (com um pessoal) → `0.5` (ambivalente 50/50)
  - Lilith-Saturno, Marte-Netuno → `0.4`
  - Marte-Saturno, Quíron-Marte fora dessa lista, planetas "duros" genéricos → `0.15`
  - Nenhum caso especial → `1.0` (conjunção "limpa" entre planetas benéficos)
  - **Conjunção "fora de signo"** (mesmo grau, mas signos diferentes — comum perto de
    0°/29°) é enfraquecida: o resultado é puxado 30% em direção ao neutro (0.5).

`hFrac` é usado tanto para separar o peso do aspecto entre `harmoniousW` (soma
`orbW × boost × hFrac`) e `tenseW` (soma `orbW × boost × (1-hFrac)`) quanto para
classificar o aspecto num "balde" narrativo (`markerCategory`): harmônico
(`hFrac ≥ 0.6`), ambivalente (`0.41–0.59`), tenso leve ou tenso puro.

### 1.3 `axisBoost` — peso por relevância tradicional do par

Multiplicador fixo por par de planetas/pontos (independente de orbe ou tipo de aspecto),
que reflete o quanto a literatura de sinastria considera aquele contato central. Definido
no mapa `AXIS_BOOST`, em **tiers**:

| Tier | Peso | Exemplos |
|---|---|---|
| 1 | **1.35** | Sol-Lua, Vênus-Marte, Lua-Lua, Sol-Sol, Mercúrio-pessoal, Saturno-pessoal, Ascendente com Sol/Lua/Vênus/Marte/Mercúrio |
| 2 | **1.30** | Nodo (Norte/Sul) tocando pessoal, Nodo-Nodo, Vértice tocando pessoal/ângulo, Vértice-Vértice, cruzamentos Nodo↔Vértice |
| 3 | **1.20** | Quíron/Lilith tocando pessoal, Marte/Vênus tocados por Urano/Netuno/Plutão, Mercúrio-Vênus/Marte, MC/IC/DSC com pessoal, Júpiter tocando pessoal, Sol/Lua tocados por transpessoal |
| baixo | **0.6** | Ascendente-Ascendente, Fortuna tocando pessoal (ponto derivado, evidência mais fraca) |
| genérico | **1.0** | qualquer par sem entrada explícita no mapa |

Qualquer par que não esteja listado recebe o peso neutro `1.0` — o boost nunca reduz um
aspecto abaixo do genérico, exceto os dois casos deliberadamente "soltos" (0.6) acima.

### Peso final de um aspecto

```
peso_harmonico = orbW × boost × hFrac
peso_tenso     = orbW × boost × (1 - hFrac)
```

Esses dois valores são somados nos acumuladores relevantes: harmonia geral, a(s)
categoria(s) de conteúdo do par (`categoryPoolFor`) e o eixo (`axisPoolFor`), se
aplicável.

## 2. Para onde o peso de cada aspecto vai

Um mesmo aspecto pode alimentar **mais de um destino** simultaneamente — não é um
sistema "pizza" (rateio fracionário somando 100%), e sim um checklist independente:

- **Harmonia geral (`harmonyPct`)** — soma todo aspecto reconhecido por *qualquer*
  categoria de conteúdo OU eixo (Estrutura/Destino). Um aspecto sem nenhum significado
  tradicional catalogado fica de fora até da Harmonia geral, propositalmente: a métrica
  quer ser "visão panorâmica curada", não "todo aspecto que apareceu no relatório".
- **Categorias de conteúdo** (`categoryPoolFor`) — Intelectual, Emocional,
  Atração/Sexual, Prático, Afinidade. Listas curadas de pares clássicos (ex.:
  Mercúrio-Mercúrio → Intelectual; Lua-Lua → Emocional; Vênus-Marte → Atração;
  Saturno-pessoal → Prático; Júpiter-pessoal → Afinidade). Três pares têm **dupla
  categoria** de propósito (`DUAL_CATEGORY_PAIRS`): Ascendente-Lua, Lua-Vênus e
  Marte-Lua contam cheio em Emocional *e* em Atração/Sexual, porque a tradição trata
  esses contatos como tendo duas naturezas genuinamente distintas.
- **Eixos Estrutura/Destino** (`axisPoolFor`) — Estrutura agrega Sol-Lua/Mercúrio
  mútuos e qualquer âncora de longo prazo (Saturno, MC/IC/DSC/Ascendente) tocando um
  pessoal do parceiro; Destino agrega Nodo/Nodo Sul/Vértice tocando pessoal, ângulo ou
  entre si.
- **Marcadores narrativos** (chips) — contadores à parte, sem peso próprio na nota
  final além do que já entra via categoria/eixo: Saturno-compromisso, Nodo-destino,
  eixo nodal mútuo, Vértice-encontro, Quíron-ferida, Lilith-magnetismo,
  Sol-transpessoal, Fortuna, Sol-Lua cruzado, câmbio de luminares (Sol de um no signo
  da Lua do outro, e vice-versa).

## 3. Sobreposições de casa (planeta de A caindo numa casa de B)

Casas não têm "orbe" (é dentro ou fora, sem grau de exatidão), então recebem peso fixo em
vez de decaimento exponencial:

- `houseMarkerWeight = 0.5` — peso padrão de um marcador de casa curado por afinidade
  temática (ex.: Mercúrio na 3ª/9ª, Júpiter na 9ª, Lua na 12ª).
- Marcadores de casa contam para a **presença** (`presence`) da categoria, mas **não**
  para o `harmonyPct` dela — casa não carrega informação de harmonia/tensão por si só.
- Exceção deliberada: Sol/Lua/Saturno na 1ª/4ª/7ª/10ª (`commitmentHouseStructureWeight
  = 0.6`) e Nodo/Nodo Sul/Vértice nesses mesmos ângulos (`destinyHouseWeight = 0.6`)
  **são** tratados como favoráveis por padrão e somam direto no lado harmônico de
  Estrutura/Destino — a leitura tradicional não trata cair nesses ângulos como neutro.

## 4. Harmonia geral (`harmonyPct`)

```
harmonyRaw = harmoniousW / (harmoniousW + tenseW) × 100      // 0–100, ponto neutro = 50
harmonyPct = clamp(50 + (harmonyRaw − 50) × harmonyStretch, 5, 95)
```

- `harmonyStretch = 1.6` — amplifica o desvio em relação ao ponto neutro (50%) antes de
  aplicar o teto/piso. Sem essa amplificação, somar dezenas de aspectos regride
  naturalmente para perto de 50%, escondendo tendências reais.
- Resultado sempre limitado entre 5% e 95% — nunca 0 ou 100 puros, refletindo que
  nenhuma sinastria é "perfeita" ou "sem nenhum sinal favorável".

A mesma fórmula de stretch/clamp é reaplicada, com o mesmo `harmonyStretch`, para cada
sub-pool (categoria e eixo) via `poolHarmonyPct(hW, tW)` — com uma diferença: se o peso
acumulado do sub-pool (`hW + tW`) for menor que `minAxisSignalWeight = 0.5`, a função
retorna `null` em vez de um número, para não fingir precisão sobre sinal fraco demais
(equivalente ao peso de um único aspecto a ~3° de orbe num par tier 1).

## 5. Nitidez (`strength`)

```
tightRatio = (peso de aspectos com orbe < 2°) / (peso total elegível) × 100
strength   = min(100, round(tightRatio × 2.3))
```

Mede o quão "carregado"/exato é o mapa entre os dois — não é bom nem ruim por si só,
é um **intensificador**. Só entram aspectos já reconhecidos por alguma categoria ou eixo
(mesmo pool curado da Harmonia geral), e excluem-se aspectos entre planetas
"outer-outer" (geracionais, não pessoais do casal). Também existe uma versão separada
por lado — `strengthHarmonic` e `strengthTense` — calculada com a mesma fórmula, mas só
sobre o sub-pool harmônico ou só o tenso, usada no Veredito (seção 7).

## 6. Categorias de conteúdo (`categoryScores`)

Para cada categoria (Intelectual, Emocional, Atração/Sexual, Prático, Afinidade):

```
presence  = min(100, round((harmoniousW + tenseW + houseW) × categoryPresenceScale))
                  // 0 se o total < minAxisSignalWeight
harmonyPct = poolHarmonyPct(harmoniousW, tenseW)     // null se sinal insuficiente
weight     = harmoniousW + tenseW + houseW           // peso bruto, sem teto — usado
                                                      // para ponderar categorias entre si
```

`categoryPresenceScale = 22` — fator "no olho", calibrado para que a maioria das leituras
não sature sempre em 100 nem fique sempre baixa. Cada categoria é **independente**: não
existe um "total do mapa" fixo sendo repartido entre elas (diferente de um antigo modelo
"pizza" abandonado, em que cada aspecto ratearia peso fracionário entre 4 categorias
somando sempre 100% — decisão revertida por não corresponder à forma como a tradição
trata essas áreas).

## 7. Compatibilidade Geral (`compatibilityScore`)

A fórmula muda pelo **tipo de vínculo**:

### Romântico

```
compatibilityScore = round( sqrt(Atração.harmonyPct × Estrutura.harmonyPct) )
```

Média **geométrica** entre o eixo Atração (`categoryScores.sexual.harmonyPct` — a
antiga fusão do eixo "Imediato" com a categoria Sexual) e o eixo Estrutura
(`structureHarmonyPct`). A geométrica pune desequilíbrio de propósito: uma relação com
muita faísca e nenhuma estrutura (ou o inverso) tende a ser mais instável do que uma
média simples sugeriria — só fica alta quando os **dois** eixos são razoavelmente bons.

Depois, um **nudge de Afinidade** (Júpiter) é somado, com teto:

```
nudge = (Afinidade.harmonyPct − 50) × affinityNudgeWeight    // affinityNudgeWeight = 0.08
compatibilityScore = clamp(compatibilityScore + nudge, 0, 100)
```

Afinidade fica **fora** da raiz geométrica (Júpiter é sobre facilidade/sorte, não sobre
os dois pilares que decidem se um vínculo romântico "tem chance real"), mas entra depois
como ajuste fino — desvio máximo de ±45 pontos × 0.08 = nudge máximo de **±3,6 pontos**,
suficiente para desempatar, insuficiente para decidir sozinho.

### Amizade / Família

```
compatibilityScore = média ponderada por presence das 5 categorias
                      (Intelectual, Emocional, Atração, Prático, Afinidade)
```

Sem a punição geométrica de faísca×estrutura (uma amizade sólida não precisa nascer com
faísca), mas ainda pondera cada categoria pela própria `presence` — uma categoria com
pouco sinal (ex.: 3 aspectos) influencia proporcionalmente menos que uma bem estabelecida
(ex.: 13 aspectos), em vez de todas pesarem igual numa média simples 1/5.

## 8. Eixos Estrutura e Destino

Calculados exatamente como as categorias (mesma `poolHarmonyPct`), mas sobre o sub-pool
de aspectos que `axisPoolFor` reconhece como Estrutura (Sol-Lua/Mercúrio mútuos, âncoras
de longo prazo tocando pessoal) ou Destino (Nodo/Vértice). Existem para não deixar a
média geral de dezenas de aspectos esconder tensão concentrada especificamente onde ela
mais importa para o tipo de vínculo.

## 9. Veredito / Potencial (`potentialScore`)

Nota final de 0–100, combinação ponderada de três blocos + um intensificador:

```
base = média ponderada de:
  compatibilityScore          peso 0.50
  harmonyPct                  peso 0.3125
  significanceScore           peso 0.1875
```

`significanceScore` (média ponderada pelo peso bruto de cada sub-pool) varia por tipo:

- **Romântico**: Destino, Emocional e Intelectual.
- **Amizade/Família**: Estrutura e Destino.

Se algum bloco faltar (por exemplo, `compatibilityScore` nulo por falta de sinal), os
pesos restantes são renormalizados entre si (`weightedAvgOrSimple`).

### Intensificador de Nitidez

A Nitidez **não soma** na base — ela decide o **quanto** o desvio da base em relação ao
ponto neutro (50) é acentuado ou amortecido:

```
sideStrength = base >= 50 ? strengthHarmonic : strengthTense   // Nitidez só do lado vencedor
intensityFactor = min + (max − min) × (sideStrength / 100)     // min=0.7, max=1.35
potentialScore = clamp(50 + (base − 50) × intensityFactor, 0, 100)
```

Mapa pouco carregado (Nitidez baixa) **amortece** o desvio — pouco "testemunho" para
confiar na leitura. Mapa muito carregado (Nitidez alta) **acentua** o desvio na mesma
direção que a base já apontava. O fator nunca inverte o sentido do resultado sozinho —
só multiplica a distância até o ponto neutro. Importante: usa-se a Nitidez **do lado que
já venceu** (harmônico se `base ≥ 50`, tenso caso contrário), não a Nitidez geral —
evitando que aspectos que discordam da direção vencedora "emprestem" força a ela.

## 10. Perfil de Vínculo (`vinculoProfile`)

Rótulo curto + descrição, derivado de uma **matriz 3×3**: eixo Estrutura (linha) × eixo
Química (coluna), cada um caindo em uma de três zonas (`vinculoHarmonyZone`):

```
zona = pct < 45  → "tenso"
       pct >= 60 → "harmonico"
       senão     → "misto"          // CALIBRATION.harmonyZone = { low: 45, high: 60 }
```

O eixo **Química** é uma combinação própria, diferente da Atração pura:

```
chemistryHarmonyPct = round(Atração.harmonyPct × 0.7 + Afinidade.harmonyPct × 0.3)
```

Atração pesa mais (0.7) por ser o marcador mais direto de "química"; Afinidade entra como
reforço mais leve (0.3), no mesmo espírito do peso reduzido que Júpiter já recebe em
`AXIS_BOOST`.

Cada uma das 9 combinações Estrutura×Química tem um rótulo e uma descrição própria — por
exemplo "Bom Pra Construir — Com Química" (ambos harmônicos) ou "Intensidade que Ensina"
(química forte, mas base tensa). Quando falta sinal suficiente em algum eixo (`null`), a
célula cai em "misto" para escolha do rótulo, mas o texto sinaliza explicitamente que é
uma leitura em aberto, não um resultado contraditório.

Separadamente, se o eixo Destino divergir muito da Estrutura (diferença ≥
`imbalanceThreshold = 25` pontos, em qualquer direção), uma nota extra é adicionada ao
texto — "sensação de destino" e "base que sustenta o dia a dia" são tratadas como
perguntas diferentes, e o sistema não fica em silêncio quando elas discordam fortemente.

## 11. Textos de aviso e limites de confiança

| Constante | Valor | Efeito |
|---|---|---|
| `minAspectsForConfidence` | 10 | Abaixo disso, aviso de "leitura baseada em poucos dados" (conta bruta de aspectos — mede se o relatório colado está completo). |
| `minAxisSignalWeight` | 0.5 | Abaixo disso (peso acumulado, não contagem), a categoria/eixo mostra "dado insuficiente" em vez de um número. |
| `nearTiePctGap` | 3 pontos percentuais | Se a 2ª e 3ª categoria (em %) estiverem mais perto que isso, o "tema" da sinastria é sinalizado como menos decisivo. |
| `imbalanceThreshold` | 25 pontos | Diferença mínima entre eixo Imediato/Estrutura (ou Estrutura/Destino) para o texto avisar sobre desequilíbrio. |
| `strongTenseThreshold` / `strongHarmonicThreshold` | 38 / 72 | Fora da `harmonyZone`, esses limiares acionam o sufixo "(bastante tensa)"/"(bastante harmônica)" no título do veredito. |

## Resumo em uma frase por métrica

- **Nitidez**: quão exato/carregado é o mapa (intensificador, não ingrediente somado).
- **Harmonia geral**: visão panorâmica do que já foi curado como astrologicamente
  relevante (categoria ou eixo).
- **Categorias**: presença + harmonia de cada área de conteúdo, independentes entre si.
- **Estrutura/Destino**: sub-pools dedicados para não esconder tensão concentrada.
- **Compatibilidade Geral**: nota técnica — geométrica (romance) ou média ponderada
  (amizade/família) — com nudge de Afinidade no romance.
- **Veredito/Potencial**: síntese final, intensificada pela Nitidez do lado vencedor.
- **Perfil de Vínculo**: leitura qualitativa (matriz Estrutura × Química), separada da
  nota numérica, com nota à parte quando Destino diverge muito de Estrutura.
