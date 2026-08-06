# Como os resultados da Calculadora de Sinastria foram calculados

Este documento explica, campo a campo e regra a regra, como o app "Sinastria —
Calculadora" chegou nos números e textos que aparecem no JSON exportado
(`comparacao-sinastrias.json`). O objetivo é que, ao enviar esse JSON junto com este
documento para outra IA, ela entenda **o raciocínio por trás de cada valor** — não só
o resultado final — e possa analisar a relação sabendo exatamente o que cada campo
significa e não significa.

Este NÃO é um sistema "oficial" de astrologia nem uma fórmula acadêmica — é um modelo
de pesos heurístico, calibrado manualmente (por tentativa, ajuste e discussão) por
quem construiu o app, a partir de leituras tradicionais de sinastria. Trate os números
como um jeito estruturado e consistente de organizar sinais astrológicos clássicos,
não como uma verdade matemática sobre a relação.

---

## 1. Entrada: o que foi processado

O app recebe um texto colado de relatório de sinastria com dois tipos de linha:

- **Linhas de aspecto**: `"P1's Planeta in Signo Aspecto P2's Planeta in Signo (Orb:
  Dº M')"` — ex. `"Ana's Sun in Aries Trine Beto's Moon in Leo (Orb: 2°15')"`.
- **Linhas de sobreposição de casa**: `"P1's Planeta in the Nª P2's house"` — ex.
  `"Ana's Venus in the 7th Beto's house"`.

O parser (`parseText`) reconhece esses dois formatos, aceita duas variações de ordem
das palavras (relatórios diferentes formatam de jeitos ligeiramente distintos) e
descarta silenciosamente linhas com `"Orb:"` que não batem com nenhum formato
conhecido — contando quantas foram descartadas (`unrecognizedCount`, exposto na
interface, não no JSON de cada entrada).

### 1.1 Limpeza antes de calcular qualquer coisa

- **Normalização de nomes** (`normPlanet`): variações de nome de relatório
  ("Midheaven"/"Medium Coeli" → `MC`, "Black Moon Lilith"/"Dark Moon Lilith" → `Lilith`,
  "South Node" → `SouthNode` etc.) são unificadas para os rótulos internos usados no
  resto do sistema.
- **Deduplicação** (`dedupeAspects`): quando o mesmo par de pessoas+planetas+aspecto
  aparece mais de uma vez no relatório (comum em exports que listam A→B e B→A), fica
  só a leitura com o orbe mais exato — sem isso o peso desse aspecto dobraria.
- **Colapso de "ecos" geométricos** (`collapseAxisMirrors`, usado 3× — para Nodo
  Norte/Sul, Ascendente/Descendente e MC/IC): esses pares de pontos são **sempre**
  opostos a 180° um do outro. Um aspecto ao Nodo Norte tem, por geometria pura, um
  "eco" automático e equivalente ao Nodo Sul (conjunção vira oposição, trígono vira
  sextil, etc. — ver `NODE_MIRROR_ASPECT`). Muitos relatórios listam os dois lados
  como se fossem sinais independentes; o sistema detecta esse padrão e mantém só uma
  leitura (a do ponto "primário": Nodo Norte, Ascendente ou MC), descartando o eco —
  senão um único alinhamento contaria 2× (ou 4×) no peso final.

O JSON de cada comparação expõe `aspectsCount`/`housesCount` já **depois** dessa
limpeza — são os números que efetivamente entraram no cálculo.

---

## 2. Motor de peso: de cada aspecto individual a um número

Para cada aspecto que sobrou depois da limpeza, o sistema calcula um **peso de
contribuição** (`harmContribution`) e uma **fração harmônica** (`hFrac`, de 0 a 1: 0 =
puramente tenso, 1 = puramente harmônico). Esses dois números, multiplicados,
distribuem-se entre "harmonioso" e "tenso" e alimentam todas as somas do sistema.

### 2.1 Peso por orbe (`orbW`) — decaimento exponencial

```
orbW = exp( -orbe / divisor )
```

O divisor varia pelo tipo de aspecto (`ORB_DECAY_DIVISOR`): aspectos maiores
(Conjunção/Oposição = 3.0, Trígono/Quadratura = 2.5) toleram orbe mais largo sem
perder muita força; aspectos menores (Sextil/Quincúncio = 1.8, Semisextil/
Semiquadratura/Sesquiquadratura = 1.5) enfraquecem bem mais rápido conforme o orbe
abre. Ou seja: **quanto mais exato o aspecto (orbe menor), mais peso ele carrega**, e
esse peso cai depressa — não linearmente — à medida que o orbe aumenta.

### 2.2 Fração harmônica (`hFrac`) — o "quão bom" de cada aspecto

Não é só "trígono = bom, quadratura = ruim". A leitura considera **quais planetas
estão envolvidos**, porque a tradição astrológica trata pares diferentes de forma
diferente mesmo dentro do mesmo tipo geométrico de aspecto:

- **Trígono/Sextil → sempre 1.0** (harmônico puro).
- **Quadratura/Oposição → depende do par**: Júpiter-Saturno = 0.1 (bloqueio
  clássico), outros aspectos duros de Júpiter = 0.3, Marte-Vênus/Marte-Marte = 0.45
  (química com atrito, não puramente ruim), Vênus tocando Sol/Lua/Mercúrio/Vênus =
  0.25, qualquer outro par = 0.0 (tenso puro).
- **Semisextil = 0.5** (neutro — sem harmonia real de elemento/modalidade entre
  signos vizinhos), **Quincúncio = 0.2** (ajuste crônico de baixo grau), **Semiquadratura
  = 0.2**, **Sesquiquadratura = 0.15**.
- **Conjunção → o caso mais elaborado**, porque conjunção "funde" as naturezas dos
  dois planetas, e cada par tem sua própria leitura tradicional:
  - Marte-Vênus/Marte-Marte (química física): 0.65
  - Marte-Lua: 0.45 · Marte-Saturno (bloqueio puro): 0.15 · Marte-Mercúrio: 0.5
  - Quíron-Marte: 0.5 · Lilith-Marte: 0.55 · Lilith-Saturno: 0.4
  - Marte-Urano: 0.5 · Marte-Netuno: 0.4 · Marte-Plutão: 0.55
  - Saturno sozinho (tocando um pessoal, sem Marte): 0.5 — ambivalente, não maléfico
    ("aspecto de compromisso" pode ancorar ou pesar, dependendo de fatores fora do
    alcance deste sistema)
  - Qualquer planeta "duro" (Marte/Saturno, `HARD_PLANETS`) sem regra própria acima: 0.15
  - Transpessoal (Urano/Netuno/Plutão) sem regra própria: 0.5
  - Quíron/Lilith (`AMBIVALENT_CONJUNCTION_POINTS`) sem regra própria: 0.5
  - Qualquer outro par: 1.0 (conjunção "neutra" tratada como fusão favorável)
  - **Conjunção "fora de signo"** (dissociada — os dois planetas próximos em grau mas
    em signos diferentes, comum perto de 0°/29°): o resultado é puxado 30% em direção
    ao neutro (0.5), porque a tradição lê isso como um contato mais fraco/ambíguo do
    que uma fusão plena no mesmo signo.

### 2.3 Peso por eixo (`axisBoost`, `AXIS_BOOST`)

Certos pares recebem um multiplicador extra porque a tradição os trata como
indicadores especialmente fortes ou especialmente fracos, independente do tipo de
aspecto:
- **Acima de 1.0** (contato mais significativo que o genérico): Sol/Lua tocados por
  um transpessoal (1.20), Ascendente-Netuno (1.20).
- **Abaixo de 1.0** (evidência auxiliar, nunca decisiva sozinha): Ascendente-Ascendente
  e Fortuna tocando um pessoal (0.6 cada) — a Parte da Fortuna é um ponto derivado
  (calculado, não observado), por isso pesa menos.
- Pares sem entrada explícita usam peso 1.0.

### 2.4 Outros multiplicadores

- **Desconto geracional** (0.3×): quando os dois lados do aspecto são planetas
  transpessoais entre si (Urano/Netuno/Plutão-Urano/Netuno/Plutão) — esses aspectos
  tendem a ser compartilhados por qualquer par nascido na mesma época, então dizem
  pouco sobre o vínculo pessoal específico.
- **Reforço de reciprocidade** (1.15×): quando o mesmo par de planetas conecta as
  duas pessoas nos dois sentidos (ex. Vênus de A com Marte de B **e** Marte de A com
  Vênus de B) — um reforço mútuo genuíno (dois aspectos diferentes "fechando o
  eixo"), não uma duplicata (já removida na limpeza).

### 2.5 Fórmula final por aspecto

```
harmContribution = orbW × axisBoost × generationalDiscount × reciprocityBoost
harmoniousW  += harmContribution × hFrac
tenseW       += harmContribution × (1 − hFrac)
```

Cada aspecto contribui simultaneamente para vários "baldes" somados em paralelo: o
total geral, a(s) categoria(s) de conteúdo a que pertence, o eixo (Estrutura/Destino)
a que pertence, e — se o orbe for apertado o bastante (`orbW >
CALIBRATION.significantMarkerOrbWeight`, ou seja, ainda relevante e não ruído de
fundo) — os marcadores narrativos específicos (Saturno-compromisso, Nodo-destino etc.,
ver seção 5).

---

## 3. Categorias de conteúdo (checklist, não "pizza")

Cinco categorias — **Intelectual, Emocional, Sexual/Atração, Prático, Afinidade** —
cada uma definida por uma lista curada de pares planeta-planeta ou planeta-casa
específicos (não "todo aspecto conta pra alguma categoria"). Um aspecto que não bate
com nenhuma lista simplesmente não entra em nenhuma categoria (mas ainda conta pro
total geral). Alguns pares têm natureza dupla e contam cheio (sem diluir) em duas
categorias (ex.: `Mars-Moon` conta em Emocional e Atração; `Pluto` na casa 7 conta em
Sexual e Prático).

Resumo do que compõe cada uma:
- **Intelectual**: Mercúrio com Sol/Lua/Mercúrio/Ascendente/Júpiter/Urano/Netuno/
  Plutão; casas 3ª/9ª (Mercúrio), 9ª (Júpiter).
- **Emocional**: Lua com Lua/Sol/Vênus/Marte/Ascendente/Netuno/Plutão/Urano; Quíron-Lua;
  casas 4ª (Lua/Vênus/Sol), 12ª (Lua), 11ª (Vênus/Lua), 7ª/8ª (Quíron).
- **Sexual/Atração** (fusão do antigo "eixo Imediato" com a antiga categoria Sexual):
  Marte-Vênus, Marte-Marte, Vênus-Vênus, Lilith/Plutão/Netuno/Urano tocando Marte ou
  Vênus, Ascendente com Marte/Vênus/Sol/Lua/Netuno, Sol com Vênus/Marte/Plutão/Netuno/
  Urano, Mercúrio com Marte/Vênus; casas 5ª/8ª (Vênus/Marte), 8ª/5ª (Lilith), 8ª
  (Plutão), 1ª/7ª (Netuno).
- **Prático**: Saturno/Nodo/Nodo Sul/MC/IC/Vértice entre si ou tocando um pessoal;
  casas 1ª/7ª/10ª (Sol/Lua), 7ª (Vênus/Marte), 2ª (Vênus), 1ª/4ª/7ª/10ª (Saturno), 7ª
  (Plutão, junto de Sexual).
- **Afinidade**: Júpiter com Sol/Lua/Vênus/Marte/Júpiter/Ascendente; Ascendente-
  Ascendente; Fortuna tocando um pessoal (peso reduzido, ponto derivado); casas
  11ª/7ª/1ª (Júpiter). Não entra na raiz geométrica da Compatibilidade Geral romântica
  (só dois eixos decidem se o vínculo "pode" ser alto), mas modula o resultado depois,
  como ajuste fino — ver seção 5. Em amizade/família entra como uma das 5 categorias
  ponderadas.

Cada categoria acumula seu próprio `harmoniousW`/`tenseW` (aspectos) e `houseW`
(marcadores de casa, peso fixo de 0.5 cada, sem gradação por orbe — casa é "dentro ou
fora", não tem "quão exata").

### 3.1 De peso acumulado a números exibidos

- **`presence` (0–100)**: soma do peso acumulado da categoria (aspectos + casas),
  multiplicada por um fator de escala (22) e limitada em 100. Mede **quanto essa área
  aparece** no mapa, não se é favorável.
  Se o peso total ficar abaixo de um piso de confiança (`minAxisSignalWeight = 0.5`
  — equivalente a mais ou menos um aspecto de orbe ~3° num eixo de peso 1.35), a
  categoria fica com `presence = 0` em vez de mostrar um número pouco confiável.
- **`weight`**: o mesmo peso acumulado bruto usado pra calcular `presence` acima, mas
  **sem** a escala ×22 nem o teto de 100 — não tem exibição própria na interface.
  Existe só pra ponderar esta categoria contra outras categorias/eixos na fatia de
  Significância do Veredito (seção 6.1), onde o teto de `presence` distorceria a
  ponderação entre sub-pools de volume de sinal muito diferente.
- **`harmonyPct` (0–100 ou `null`)**: só olha `harmoniousW/tenseW` de aspectos (não
  inclui casas, que não carregam informação de harmonia). Fórmula:
  `raw = harmoniousW / (harmoniousW + tenseW) × 100`, depois "esticada" em torno do
  ponto neutro (50%) por um fator de 1.6 e limitada entre 5% e 95% (evita que dezenas
  de aspectos regridam pra perto de 50% e evita 0%/100% de falsa certeza absoluta).
  Fica `null` (não `50`) quando o peso do sub-pool for baixo demais pra confiar —
  distinção importante: `null` significa "sem dado suficiente", não "neutro".

---

## 4. Eixos: Estrutura e Destino

Além das categorias de conteúdo, dois eixos "de papel" (não de conteúdo) agregam
outro corte dos mesmos aspectos:

- **Estrutura** (`structureHarmonyPct`): planetas/pontos ligados a permanência e
  compromisso — Saturno, MC, IC, Descendente, Quíron — tocando um pessoal do parceiro
  ou entre si, mais Sol/Lua/Mercúrio entre si (pares "estruturais" de luminares e
  mente). Sol/Lua/Saturno caindo nas casas 1ª/4ª/7ª/10ª do parceiro somam um empurrão
  fixo e sempre favorável (peso 0.6, "aspecto maior de orbe moderado") a este eixo —
  a tradição não trata "estar nesses ângulos" como neutro.
- **Destino** (`destinyHarmonyPct`): Nodo Norte/Sul e Vértice tocando um pessoal ou
  entre si. Mesmo empurrão fixo (0.6) para Nodo/Vértice nas casas 1ª/4ª/7ª/10ª.

Cada eixo também expõe um `structureWeight`/`destinyWeight` — o peso acumulado bruto
do sub-pool (mesmo papel do `weight` de categoria, seção 3.1: sem escala nem teto de
100), usado só pra ponderar o eixo contra outros eixos/categorias na Significância do
Veredito (seção 6.1). Não tem exibição própria na interface.

O antigo "eixo Imediato" (química rápida/reconhecimento à primeira vista) foi
fundido dentro da categoria Sexual/Atração — por isso `immediateHarmonyPct` no JSON
é, tecnicamente, o mesmo valor de `categoryScores.sexual.harmonyPct` (mantido como
campo próprio por compatibilidade com leituras salvas antes dessa fusão).

---

## 5. Compatibilidade Geral (`compatibilityScore`)

A fórmula muda pelo tipo de vínculo (`relType`), guardado em cada entrada:

- **Romântico**: **média geométrica** entre Atração (`immediateHarmonyPct`) e
  Estrutura (`structureHarmonyPct`): `√(atração × estrutura)`. Proposital: pune
  desequilíbrio. Um par com muita faísca e nenhuma estrutura (ou o oposto) tende a
  ser mais instável do que uma média simples sugeriria — só fica alto quando os
  **dois** eixos são razoavelmente bons. Fica `null` se qualquer um dos dois eixos
  não tiver dado suficiente.

  Depois de calculada a raiz, entra um **nudge de Afinidade** (Júpiter): o desvio de
  `categoryScores.afinidade.harmonyPct` em relação ao ponto neutro (50) é multiplicado
  por `CALIBRATION.affinityNudgeWeight` (0.08) e somado ao resultado, com o total
  sempre travado entre 0 e 100 — desvio máximo ±45 × 0.08 = ajuste de até ±3,6 pontos.
  Júpiter tocando pessoal é lido como bênção clássica em sinastria de casamento, mas a
  tradição não trata "é fácil/gostoso estar perto" como um dos dois pilares que
  decidem se o vínculo tem chão pra funcionar — por isso ele **modula** a nota central
  em vez de **votar** dentro dela: fica de fora da raiz geométrica (não muda se o
  Compat "pode" ser alto — só Atração e Estrutura decidem isso) e entra só depois,
  como ajuste aditivo com teto. Só se aplica se o Compat já existir (não "tempera" uma
  nota `null`) e se Afinidade tiver sinal (`afinidadeHarmonyPct` não-nulo).
- **Amizade/Família**: **média das 5 categorias de conteúdo** (Intelectual, Emocional,
  Sexual/Atração, Prático, Afinidade — Afinidade agora entra aqui também), **ponderada
  pela `presence` de cada categoria**, não mais uma média simples de peso igual.
  Motivo: um sub-pool fino (ex. Afinidade com `presence` baixa, poucos aspectos)
  puxava a média com o mesmo peso que uma categoria bem estabelecida (`presence` alta,
  muitos aspectos), deixando pouco sinal distorcer o resultado tanto quanto sinal
  confiável. Ponderar por `presence` deixa uma categoria pouco evidenciada influenciar
  pouco, sem excluí-la — o sinal ainda conta, só proporcional à confiança que temos
  nele. Cai de volta pra média aritmética simples no caso raro de todos os pesos
  ficarem em zero. Aqui a punição por desequilíbrio Atração×Estrutura (a raiz
  geométrica do caso romântico) não se aplica — amizade não precisa nascer com
  faísca —, mas desequilíbrio real entre as áreas de conteúdo ainda pesa a média pra
  baixo.

Quando `immediateHarmonyPct` e `structureHarmonyPct` existem os dois e a diferença
entre eles é grande (≥25 pontos, `imbalanceThreshold`), o texto do veredito
(`classify()`) adiciona uma nota específica de desequilíbrio (ex.: "muita faísca,
pouca estrutura" ou o inverso) — sinalizado no texto, não escondido atrás de um
número único.

---

## 6. "Força" (`strength`)

Mede **confiabilidade**, não qualidade: `% de aspectos com orbe < 2°` (excluindo
aspectos entre dois transpessoais, que são geracionais), escalado por 2.3 e limitado
em 100. Um `strength` baixo (<15) faz o veredito parar antes de apontar qualquer tema
dominante — "poucos aspectos bem exatos, não há carga concentrada em nenhuma direção
específica" —, porque um número decisivo apoiado em 1–2 aspectos soltos é mais
enganoso do que um número no meio do caminho. Este é o número **cego a lado** — não
distingue se os aspectos apertados são harmônicos ou tensos — e é o que aparece na
interface e nos textos do veredito.

### 6.0.1 Nitidez por lado (`strengthHarmonic`/`strengthTense`) — auditoria pós-discussão

Um mapa pode ter `strength` geral alto por causa de aspectos apertados de **ambos**
os lados ao mesmo tempo — ex.: base geral pendendo tensa (mais peso tenso que
harmônico no total), mas os aspectos *mais exatos* especificamente sendo os
harmônicos (só que em menor volume/peso). Nesse caso, o `strength` geral sobe
"emprestando" confiança de um lado que, na verdade, discorda da direção que a base
apontou — um ponto cego real, que não tinha nenhuma conferência antes desta revisão.

Por isso o motor agora calcula, com o mesmo `hFrac` fracionário já usado em
`harmoniousW`/`tenseW` (seção 2.5), dois números adicionais, mesma fórmula/escala do
`strength` geral, mas cada um só sobre o sub-pool do seu lado:
- `strengthHarmonic`: % de orbe apertado (<2°) dentro do peso **elegível harmônico**
  (`hFrac` de cada aspecto elegível).
- `strengthTense`: % de orbe apertado dentro do peso **elegível tenso** (`1 − hFrac`).

Esses dois números não aparecem sozinhos na interface — eles existem só para
alimentar o intensificador do Veredito (seção 6.1) com a Nitidez do lado certo, em
vez da Nitidez geral cega a lado.

---

## 6.1 Veredito (`potentialScore`)

Nota derivada de duas etapas, usada pra ordenar o histórico ("Melhor em veredito") e
mostrada na interface como "Veredito". Internamente a chave continua se chamando
`potentialScore` — só o nome exibido mudou.

**Etapa 1 — base (soma 100%):**
- 50% Compatibilidade Geral (`compatibilityScore`, seção 5)
- 31,25% Harmonia geral (`harmonyPct`, seção 2)
- 18,75% Significância (auditoria pós-discussão) — o que fica de fora dos dois números
  acima, ou seja, o que não tem NENHUMA outra fatia dedicada na fórmula:
  - Em vínculos **românticos**: antes era Destino puro (`destinyHarmonyPct`), com o
    argumento de que Estrutura já entra dentro de Compatibilidade Geral. Mas essa
    mesma lógica — "o que fica de fora do resto entra aqui, sem favorecer um em cima
    do outro" — também vale pra Emocional e Intelectual, que não entram em NENHUM
    outro lugar da fórmula romântica (Atração já está "falada" dentro de
    Compatibilidade Geral; Afinidade via o nudge de Júpiter, seção 5) e ficavam de
    fora do Veredito quase por completo, só diluídas de forma anônima dentro da
    Harmonia geral (junto com tudo mais, sem sinal próprio — ver seção 3 pra saber o
    que compõe cada categoria). Não há lastro astrológico pra tratar Destino como mais
    "significativo" que Emocional/Intelectual a ponto de ele sozinho preencher a
    fatia toda. Agora: **média ponderada por peso bruto de sinal** de Destino +
    `categoryScores.emocional.harmonyPct` + `categoryScores.intelectual.harmonyPct`.
  - Em **amizade/família**: por consistência com a mudança acima, também virou média
    **ponderada** de Estrutura + Destino (antes era simples) — mesmo raciocínio: os
    dois ficam de fora do resto da fórmula igualmente nesse tipo de vínculo, mas não
    necessariamente com o mesmo volume de sinal disponível cada vez.

  A ponderação usa **peso bruto** (`destinyWeight`, `structureWeight`,
  `categoryScores[k].weight` — mesma unidade/mesmo cálculo que alimenta `presence`,
  seção 3.1, mas SEM a escala ×22 nem o teto de 100), não o `presence` já exibido na
  interface: `presence` satura em 100, o que faria dois sub-pools de volume de sinal
  bem diferente (ex. Destino, tipicamente um pool pequeno de aspectos de Nodo/
  Vértice, vs Emocional/Intelectual, com listas de pares bem maiores) pesarem igual
  na média sempre que os dois batessem no teto — justamente o problema que a
  ponderação existe pra evitar (mesmo raciocínio já usado em `compatibilityScore` de
  amizade/família, seção 5, só que ali a ponderação usa `presence`, não peso bruto,
  porque lá os cinco sub-pools sendo combinados são mais parecidos em volume típico).
  Cai de volta pra média aritmética simples se todos os pesos vierem zero/ausentes
  (ex. entrada antiga recalculada antes desta revisão, sem `destinyWeight`/
  `structureWeight`/`categoryScores[k].weight` salvos).

  Qualquer um dos três/dois componentes que estiver `null` simplesmente não entra na
  média (nem no peso). Se todos os componentes da Significância ficarem `null`, ela
  sai `null` e é excluída da soma da Etapa 1 (o peso das outras duas fatias é
  renormalizado sobre o que sobrou) — se as três fatias da Etapa 1 estiverem `null`,
  `potentialScore` sai `null`.

**Etapa 2 — Nitidez como intensificador (alinhada por lado):** a base da etapa 1 não
é o número final. Uma Nitidez decide o **quanto** o desvio dessa base em relação ao
ponto neutro (50) é acentuado ou amortecido — nunca inverte o sentido, só intensifica
ou amortece: `resultado = 50 + (base − 50) × fator`, onde `fator` varia entre
`CALIBRATION.vereditoIntensifier.min` (0.7, com Nitidez=0 — pouco "testemunho" pra
confiar na leitura, desvio amortecido mas não zerado) e `.max` (1.35, com Nitidez=100
— mapa muito carregado, desvio acentuado).

A Nitidez usada aqui **não é** o `strength` geral cego a lado (seção 6) — é
`strengthHarmonic` (seção 6.0.1) se a base ficou ≥ 50, ou `strengthTense` se a base
ficou < 50. Motivo (auditoria pós-discussão): usar a Nitidez geral podia amplificar a
base com "combustível" parcialmente emprestado do lado que perdeu a leitura — ex.
base pendendo tensa mas com Nitidez geral alta por causa de aspectos harmônicos bem
exatos, que não é evidência a favor da tensão que está sendo amplificada. Com a
Nitidez alinhada, só o lado que a base já apontou como vencedor pesa no fator — se o
lado tenso (o que venceu) tiver poucos aspectos apertados, mesmo com Nitidez geral
alta, o desvio sai amortecido, não acentuado. Cai de volta pro `strength` geral só em
entradas antigas recalculadas antes desta revisão (sem `strengthHarmonic`/
`strengthTense` salvos).

O resultado final é arredondado e travado entre 0 e 100.

Importante: mais Nitidez não torna o vínculo "melhor" — só torna a leitura (seja ela
harmônica ou tensa) mais decisiva. E o Veredito mede **solidez geral da leitura**, não
"melhor prospecto romântico": ele mistura de propósito coisas que a Compatibilidade
Geral mantém separadas (ex. Destino, que é sobre significância, não sobre "dar
certo"). Pra decidir especificamente se um vínculo romântico tem chão pra funcionar,
Compatibilidade Geral + Perfil de Vínculo (seção 8) são as métricas com esse foco —
o Veredito é complementar, não substituto.

---

## 7. Marcadores narrativos (os "chips" da interface)

Contadores dedicados que rastreiam contatos específicos tradicionalmente citados em
sinastria, cada um com sua própria contagem de harmônico/ambivalente/tenso/tenso-leve
(`markerCategory`, baseado em faixas de `hFrac`: harmônico ≥0.6, ambivalente 0.41–0.59,
tenso-leve 0.2–0.4, tenso <0.2) e sua lista de detalhes textuais:

| Marcador | O que rastreia | Também conta para |
|---|---|---|
| `saturnCommitment*` | Saturno tocando um planeta pessoal do parceiro | Eixo Estrutura + categoria Prático |
| `nodeDestiny*` | Nodo Norte/Sul tocando um pessoal | Eixo Destino + categoria Prático |
| `nodeAxis*` | Nodo/Vértice tocando outro ponto de Destino (eixo nodal mútuo) | Eixo Destino + categoria Prático |
| `vertexFated*` | Vértice tocando um pessoal ou ângulo | Eixo Destino + categoria Prático |
| `chironWound*` | Quíron tocando um pessoal | Eixo Estrutura (contatos Quíron-Lua também contam para Emocional) |
| `lilithMagnetic*` | Lilith tocando um pessoal | Só categoria Atração |
| `sunTranspersonal*` | Sol tocado por Urano/Netuno/Plutão | Só categoria Atração |
| `fortune*` | Parte da Fortuna tocando um pessoal (peso auxiliar) | Só categoria Afinidade |
| `sunMoon*` | Sol de um com Lua do outro (eixo de reconhecimento) | Eixo Estrutura + categoria Emocional |
| `houseConvergenceContacts` | Sobreposições de casa recíprocas ou em casas temáticas (5ª/8ª) | Nenhum eixo/categoria — só informativo |
| `commitmentHouseContacts` | Sol/Lua/Saturno de um nas casas 1ª/4ª/7ª/10ª do outro | Só eixo Estrutura |
| `destinyHouseContacts` | Nodo/Vértice nas mesmas 4 casas | Só eixo Destino |
| `friendshipHouseContacts` | Júpiter na 11ª casa do outro | Só categoria Afinidade |
| `chironPartnershipHouseContacts` | Quíron na 7ª casa (posição, não aspecto) | Só categoria Prático/Sexual |
| `plutoPartnershipHouseContacts` | Plutão na 7ª casa | Só categoria Prático/Sexual |
| `isLuminarySwap`/`luminarySwapDetail` | Sol de cada um cai no signo da Lua do outro | Nenhum eixo/categoria — só informativo |

Esses marcadores **não** mudam o `harmonyPct` geral por conta própria (esse efeito já
vem do peso do aspecto em si, via `HARD_PLANETS`/`AXIS_BOOST`) — eles existem para dar
visibilidade textual a contatos que a tradição trata como especialmente citáveis,
mesmo quando "diluídos" dentro de uma média maior.

**Na interface**, os chips aparecem agrupados em 4 blocos — ⚖️ Estrutura, ☊ Destino,
🧭 Categorias (marcadores que não alimentam eixo nenhum) e 📝 Só informativo —
seguindo a coluna "Também conta para" acima; um marcador que alimenta eixo e
categoria ao mesmo tempo (Saturno, Nodo, Vértice, Sol-Lua, Quíron) aparece só no
grupo do eixo, com uma nota no próprio chip lembrando da outra categoria, pra não
esconder a informação. Dentro de cada grupo, os chips saem ordenados por número de
contatos (maior primeiro). Perfil de Vínculo (seção 8) fica fixo no topo, fora dos
grupos, por ser a síntese dos dois eixos, não mais um item pra categorizar. Passar o
mouse nos quadros de Estrutura/Destino (na seção anterior aos chips) também mostra a
lista de aspectos que compõem aquele eixo especificamente, separados em harmônico e
tenso — útil quando o hover das barras de categoria não deixa claro o que pesou num
eixo em particular.

---

## 8. Perfil de Vínculo (`vinculoProfile`)

Classificação separada, numa matriz 3×3, que cruza:
- **Eixo Estrutura** (`structureHarmonyPct`, o mesmo da seção 4)
- **Eixo Química** (`combineChemistryHarmonyPct`: combinação de Atração +
  Afinidade)

Cada eixo é classificado em 3 zonas (`vinculoHarmonyZone`): **tenso**, **misto** ou
**harmônico** (ou "sem sinal", se `null` por falta de peso suficiente — tratado como
"misto" para fins de escolha de célula, mas sinalizado à parte como ressalva no
texto). O cruzamento das duas zonas escolhe uma das 9 células da `VINCULO_MATRIX`,
cada uma com um rótulo e um texto-modelo (ex.: canto harmônico×harmônico = "Bom Pra
Construir — Com Química"; tenso×harmônico = "Intensidade que Ensina" — química forte,
mas a base que sustentaria algo duradouro vem tensa).

O campo `signals` (lista de bullets) é **só ilustrativo** — mostra quais contadores da
seção 7 estão por trás do rótulo (ex. "🟢 2 contato(s) Sol-Lua harmônico(s)"), mas não
decide mais o rótulo (isso mudou numa revisão do sistema: antes os marcadores somavam
pontos direto num score próprio; agora o rótulo vem só de `structureHarmonyPct` +
química, e os marcadores só explicam o "porquê").

---

## 9. Texto do veredito (`verdictTitle` no JSON, `classify()` no código)

Gerado por regras, na seguinte ordem de prioridade:

1. **`strength < 15`** → "Conexão sutil": poucos dados, não aponta tema.
2. **Nenhuma categoria com `presence > 0`** → aviso de que não há marcadores
   reconhecidos, sem inventar um tema.
3. **Domínio extremo de Sexual/Atração** (mais de 13 pontos de `presence` acima da
   maior das outras três, comparando por PROMINÊNCIA — ver item 5) → título/texto
   dedicado a "é sobre química", separado por relType e por harmônico/tenso.
4. **Caso geral**: as categorias são ordenadas por **prominência**, não só por
   `presence` bruta — `prominence = presence × pesoDeDecisão(harmonyPct da própria
   categoria)`, onde o peso de decisão vai de 0.65 (quando a harmonia daquela
   categoria está em 50% exato — "campo de disputa em aberto", sem lado claro) até
   1.0 (quando a harmonia é 0% ou 100% — extremo decisivo). Ou seja: duas categorias
   com a mesma presença não pesam igual no "tema" da conexão se uma tem sinal
   claramente favorável/tenso e a outra está em cima do muro.
   - As duas (ou mais, se houver empate real — `nearTiePctGap = 3` pontos) categorias
     de maior prominência formam a frase-âncora do veredito, usando o texto
     pré-escrito para aquele par específico (`PAIR_INFO_BY_TYPE`, que também muda
     pelo tipo de vínculo).
   - Notas adicionais são anexadas conforme o caso: separação de tom quando uma
     categoria do grupo-topo é claramente favorável e outra claramente não
     (`hasSplit`), menção à categoria "logo atrás" quando a diferença pro 3º lugar é
     pequena, menção à categoria que ficou por último, aviso de faixa mista
     (`harmonyPct` entre 45% e 60%) e a nota de desequilíbrio Atração×Estrutura (seção 5).

O texto é sempre montado dinamicamente a partir dos números — não há veredito
"fixo" independente dos dados de entrada.

---

## 10. Glossário rápido dos campos do JSON exportado

Cada entrada no JSON exportado é o objeto `entryData` de uma comparação salva:

- `n1`, `n2`: nomes das duas pessoas (como extraídos do texto colado).
- `raw`: o texto original colado (permite reprocessar/recalcular depois).
- `relType`: `'romantico'` | `'amizade'` | `'familia'`.
- `categoryScores`: objeto com `intelectual`/`emocional`/`sexual`/`pratico`/`afinidade`,
  cada um com `presence`, `weight` (peso bruto, seção 3.1), `harmonyPct` (ou `null`),
  `eligibleCount` e listas de detalhes textuais por marcador (harmônico/ambivalente/
  tenso/casa).
- `harmonyPct`: harmonia geral (todos os aspectos, seção 2).
- `strength`: confiabilidade da leitura, cega a lado (seção 6), 0–100.
- `strengthHarmonic`, `strengthTense`: confiabilidade separada por lado (seção
  6.0.1), 0–100 — usados internamente pelo intensificador do Veredito (seção 6.1),
  não têm exibição própria na interface.
- `immediateHarmonyPct`: = `categoryScores.sexual.harmonyPct` (ver seção 4).
- `structureHarmonyPct`, `destinyHarmonyPct`: os dois eixos (seção 4).
- `structureWeight`, `destinyWeight`: peso bruto de sinal de cada eixo, sem escala
  nem teto (seção 4) — uso interno da ponderação em Significância (seção 6.1), sem
  exibição própria.
- `structureHarmonicDetails`, `structureTenseDetails`, `destinyHarmonicDetails`,
  `destinyTenseDetails`: listas de strings formatadas com os aspectos (e, no caso
  favorável, também os empurrões fixos de casa) que compõem cada eixo — mesma função
  que as listas de detalhe das categorias, só que para Estrutura/Destino (alimentam o
  hover dos quadros de eixo na interface, seção 7).
- `compatibilityScore`: número único (seção 5).
- `potentialScore`: nota do "Veredito" (seção 6.1), 0–100 ou `null`.
- `verdictTitle`: título do veredito textual (seção 9; a descrição completa não é
  reexportada no JSON, só o título — o texto completo aparece na interface).
- `aspectsCount`, `housesCount`: quantos aspectos/casas entraram no cálculo, já após
  limpeza (seção 1.1).
- Um bloco de contadores por marcador narrativo (seção 7): `<nome>Contacts`,
  `<nome>Harmonic`, `<nome>Ambivalent`, `<nome>Tense`, `<nome>TenseLight`,
  `<nome>Details` (lista de strings formatadas, uma por contato daquele tipo).
- `vinculoProfile`: `{ label, description, structureHarmonyPct, chemistryHarmonyPct,
  signals[] }` (seção 8) — ou `null` se não houver sinal suficiente em nenhum dos dois
  eixos nem em nenhum marcador narrativo.
- `isLuminarySwap`, `luminarySwapDetail`, `luminarySwapCategory`: se o Sol de cada
  pessoa cai no signo da Lua da outra, e se esse contato específico Sol-Lua é
  harmônico/tenso (quando há orbe suficiente pra dizer).

## 11. O que este sistema NÃO faz (limites importantes para quem for interpretar)

- Não usa dados de trânsitos, progressões, retornos solares nem timing — é uma
  leitura estática de sinastria (aspectos + casas entre dois mapas fixos).
- Não pondera fatores fora do mapa (fase de vida, se as duas pessoas topam o peso de
  um aspecto, contexto da relação, comunicação real entre elas) — o próprio código
  comenta isso explicitamente em vários pontos (ex.: Saturno-pessoal é "ambivalente"
  porque qual lado domina "depende de fatores que este sistema não enxerga").
  Os números descrevem potenciais e temas prováveis, não determinam desfechos.
- `null` em qualquer `harmonyPct` significa "dado insuficiente pra confiar", não
  "50/50" — não deve ser lido como neutro.
- Os valores de peso, `hFrac` e `AXIS_BOOST` são calibração heurística de quem
  desenvolveu o app junto com leituras tradicionais de sinastria — outra pessoa ou
  outra escola astrológica poderia calibrar esses mesmos pares de forma diferente.
