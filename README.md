# Sinastria — Calculadora

Aplicação web de página única (`index.html`, HTML/CSS/JS puro, sem dependências
externas além de fontes do Google Fonts) que lê o texto bruto de um relatório de
sinastria astrológica (aspectos entre duas cartas natais + sobreposições de casa) e
gera uma leitura quantificada e qualificada da conexão entre duas pessoas.

## O que o sistema faz

1. **Cola o relatório** — você cola o texto exportado de um app/site de astrologia
   (formato tipo `"Fulano's Sun in Aries Sextile Beltrano's Moon in Leo (Orb: 3°12')"`
   para aspectos, e `"Fulano's Venus in the 7th Beltrano's house"` para casas), escolhe
   o tipo de vínculo (Romântico / Amizade / Família) e clica em calcular.
2. **O motor interpreta o texto**, normaliza nomes de planetas/pontos, remove
   duplicatas e "ecos" geométricos automáticos (Nodo Norte/Sul, Ascendente/
   Descendente, MC/IC), e converte cada aspecto num peso numérico que depende do tipo
   de aspecto, do orbe (quão exato ele é) e de quais planetas estão envolvidos.
3. **Esses pesos alimentam várias camadas de leitura em paralelo:**
   - Um **percentual geral de harmonia** (o quanto, no total, os aspectos favorecem
     fluidez vs. tensão).
   - **5 categorias de conteúdo** (Intelectual, Emocional, Sexual/Atração, Prático,
     Afinidade), cada uma com sua própria presença (0–100) e sua própria harmonia.
   - **2 eixos de leitura** (Estrutura = permanência/compromisso; Destino =
     Nodo/Vértice, tema cármico/de significado).
   - Um número de **"força"** (quantos aspectos são muito exatos — mede o quanto dá
     pra confiar na leitura, não o quão "boa" ela é), calculado também **separado por
     lado** (harmônico/tenso) para uso interno do Veredito abaixo.
   - Um **"Veredito"** (`potentialScore`) — nota derivada em duas etapas (50%
     Compatibilidade Geral + 31,25% harmonia geral + 18,75% de significância — em
     vínculos românticos, a **média ponderada por peso de sinal** de Destino,
     Emocional e Intelectual, os três que ficam de fora do resto da fórmula; em
     amizade/família, Estrutura e Destino do mesmo jeito —, depois intensificada pela
     Nitidez **do lado que a própria base já apontou como vencedor** — nunca pela
     Nitidez geral, que poderia estar "emprestada" do lado que perdeu a leitura) usada
     para ordenar o histórico por "melhor em veredito". Mede solidez geral da leitura,
     não "melhor prospecto romântico" — para isso, Compatibilidade Geral + Perfil de
     Vínculo continuam sendo as métricas certas.
   - Uma dúzia de **marcadores narrativos** específicos (Saturno-compromisso,
     Nodo-destino, Vértice, Quíron, Lilith, Sol tocado por transpessoal, Fortuna,
     eixo Sol-Lua, convergências de casa, câmbio de luminares Sol↔Lua etc.), cada um
     virando um "chip" expansível na interface com o detalhe de cada contato — agrupados
     visualmente por Estrutura / Destino / Categorias / Só informativo, e ordenados
     dentro de cada grupo pelo número de contatos.
   - Um **"Perfil de Vínculo"** (ex.: "Bom Pra Construir — Com Química", "Intensidade
     que Ensina", "Vínculo de Crescimento/Lição") que cruza o eixo Estrutura com o
     eixo de Química (Atração+Afinidade combinadas) numa matriz 3×3.
   - Uma **Compatibilidade Geral** (um único número) — média geométrica entre
     Atração e Estrutura para vínculos românticos (pune desequilíbrio entre faísca e
     permanência de propósito), temperada por um ajuste fino de Afinidade (Júpiter)
     de até ±3,6 pontos; ou média das 5 categorias de conteúdo (incluindo Afinidade)
     ponderada pela presença de cada uma, para amizade/família — assim uma categoria
     com pouco sinal pesa menos na média em vez de empatar peso com uma bem
     estabelecida.
   - Um **texto de veredito** gerado dinamicamente (`classify()`), que identifica
     quais categorias dominam a conexão, se o domínio é claro ou "quase empate", se a
     leitura de harmonia é decisiva ou fica numa faixa mista, e sinaliza avisos
     (poucos dados, desequilíbrio entre eixos, etc.) em vez de forçar uma conclusão
     sem lastro.
4. **Gerencia um histórico de comparações**: cada cálculo pode ser salvo (persistido
   em `localStorage`), editado, recalculado (útil depois de mudanças no motor de
   peso), removido, visualizado em detalhe e exportado. A lista pode ser **filtrada**
   por tipo de vínculo (Romântico/Amizade/Família, com contagem de cada um) e
   **ordenada** por 6 critérios (Compatibilidade, Veredito, Harmonia geral, Destino,
   Força, ou total de marcadores), cada um com uma cadeia de desempates específica
   (ex.: empate em Compatibilidade cai pra Harmonia, depois Força, depois total de
   marcadores) antes do desempate final por id, garantindo uma ordem sempre
   determinística.
5. **Import/export**: exporta todas as comparações salvas em JSON (estrutura completa
   de cada cálculo, incluindo todos os marcadores e detalhes) ou CSV (tabela
   resumida). Importa um JSON de volta, com validação/normalização de entradas de
   versões antigas do formato.

## Por que existe esse redesign de categorias

O sistema **não** distribui cada aspecto "em fatias" de um bolo que soma 100% entre
as categorias (modelo antigo). Em vez disso, cada categoria é uma **checklist
independente**: um par específico de planetas/pontos (ex. `Mercury-Mercury`,
`Moon-Neptune`, `Mars-Venus`) só entra na categoria à qual esse contato é
classicamente associado na tradição astrológica (curadoria manual, documentada linha
a linha no código). Um aspecto que não bate com nenhuma lista continua contribuindo
para a harmonia geral e, se for o caso, para o eixo Estrutura/Destino — só não entra
em nenhuma categoria de conteúdo específica. Alguns pares têm "natureza dupla" e
contam cheio (sem diluir) em duas categorias ao mesmo tempo (ex. `Mars-Moon` conta em
Emocional *e* Atração).

## Estrutura do arquivo

Tudo está em um único `index.html`:
- **CSS** (estilo "observatório/papel antigo", tipografia serifada + monoespaçada).
- **Constantes de calibração e listas curadas** (`CALIBRATION`, `AXIS_BOOST`,
  `ORB_DECAY_DIVISOR`, os `*_PAIRS`/`*_ANCHORS`, `VINCULO_MATRIX`, textos por tipo de
  vínculo).
- **Motor de parsing** (`parseText`, `normPlanet`, `dedupeAspects`,
  `collapseAxisMirrors`).
- **Motor de peso e agregação** (`harmonicFraction`, `axisBoost`, `axisPoolFor`,
  `categoryPoolFor`/`categoryPoolForHouse`, `computeScores`).
- **Motor de veredito e perfil de vínculo** (`classify`, `computeVinculoProfile`).
- **UI/estado**: renderização das barras/categorias (com hover mostrando os
  marcadores por trás de cada uma, inclusive nos quadros de Estrutura/Destino), chips
  de marcador agrupados e ordenados, lista de comparações salvas, import/export, tudo
  em JS puro manipulando o DOM diretamente (sem framework).

## Histórico deste projeto

O motor de cálculo foi construído e vem sendo revisado de forma incremental, seção
por seção (parsing → motor de peso → categorias/casas → agregação/veredito), com
correções de gaps encontrados em auditoria registradas como comentários no próprio
código — por isso boa parte do arquivo tem comentários longos explicando "por que
esse valor" e "o que estava faltando antes". Isso é proposital: o objetivo é que o
raciocínio por trás de cada peso fique rastreável, não só o número final.

Para uma explicação detalhada de **como cada número do JSON exportado foi calculado**
— pensada especificamente para ser enviada junto do arquivo JSON de uma comparação
para outra IA analisar —, veja `CALCULO.md`.
