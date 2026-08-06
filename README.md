# Sinastria — Calculadora

Aplicativo web single-file (HTML + CSS + JS puro, sem build/dependências) para calcular e
interpretar **sinastrias astrológicas** (compatibilidade entre dois mapas) a partir de um
relatório de aspectos/casas colado como texto. O sistema pontua a relação em múltiplos
eixos e categorias, gera um "veredito" narrativo, guarda comparações entre várias
sinastrias, mantém um dicionário pessoal de significados por padrão de aspecto/casa, e
exporta relatórios para impressão.

> ⚠️ O próprio app se descreve como "uma leitura simbólica de entretenimento baseada em
> pesos heurísticos entre planetas, aspectos e casas — não um veredito científico".

## Como usar

1. Abra `index.html` em qualquer navegador moderno (não precisa de servidor, build ou
   instalação — é um arquivo estático autocontido).
2. Na aba **Calculadora**, cole o texto do relatório de sinastria (lista de aspectos e,
   opcionalmente, sobreposições de casa) na área de texto, informe os nomes das duas
   pessoas e o tipo de vínculo (Romântico / Amizade / Família).
3. Clique em calcular para ver: compatibilidade geral, veredito, barras por categoria
   (Intelectual, Emocional, Atração, Prático, Afinidade), eixos Estrutura/Destino,
   marcadores narrativos (chips) e o perfil de vínculo.
4. Salve a leitura para compará-la com outras sinastrias já calculadas (lista com
   ordenação, filtros por tipo de vínculo, exportação/importação em CSV/JSON).
5. Na aba **Dicionário**, cadastre o significado de cada padrão de aspecto/casa
   encontrado — pode ser específico de uma sinastria ou reutilizável entre todas.
6. Na aba **Relatório**, monte um PDF/impresso combinando uma ou mais sinastrias
   salvas, escolhendo quais seções incluir (resumo, números, categorias, marcadores,
   significados) e, com duas ou mais selecionadas, uma tabela comparativa.

## Estrutura de uma leitura

| Conceito | O que é |
|---|---|
| **Compatibilidade Geral** (`compatibilityScore`) | Nota 0–100 que resume o "veredito técnico" da sinastria; a fórmula muda por tipo de vínculo (ver `calculo.md`). |
| **Harmonia geral** (`harmonyPct`) | % do peso astrológico total que vem de aspectos harmônicos vs. tensos, sobre um pool curado de aspectos reconhecidos. |
| **Nitidez** (`strength`) | % de aspectos muito exatos (orbe apertado) dentro do pool curado — mede "quão carregado" é o mapa, não se é bom ou ruim. |
| **Categorias** (`categoryScores`) | Intelectual, Emocional, Atração/Sexual, Prático, Afinidade — cada uma com `presence` (quanto sinal existe) e `harmonyPct` (quão harmônico é esse sinal). |
| **Eixos Estrutura/Destino** | Estrutura = permanência/compromisso de longo prazo; Destino = Nodo/Vértice, "sensação de destino/fatalidade". |
| **Veredito / Potencial** (`potentialScore`) | Nota final que combina Compatibilidade + Harmonia + significância dos eixos secundários, intensificada pela Nitidez. |
| **Perfil de Vínculo** (`vinculoProfile`) | Rótulo curto (ex. "Bom Pra Construir — Com Química") derivado do cruzamento Estrutura × Química numa matriz 3×3. |
| **Marcadores narrativos** | Chips específicos (Sol-Lua, Saturno-compromisso, Nodo-destino, Vértice, Quíron, Lilith, Fortuna, convergência de casas etc.) que ilustram *por que* o número saiu daquele jeito. |

Veja a explicação completa dos pesos e fórmulas em **[`calculo.md`](./calculo.md)**.

## Funcionalidades principais

- **Parser de relatório**: lê texto colado de relatórios de sinastria (aspectos com
  planeta, tipo de aspecto e orbe; sobreposições de casa), normaliza nomes de planetas/
  pontos (ex.: "Midheaven" → `MC`, variações de "Lilith" → `Lilith`) e remove duplicatas/
  ecos espelhados (ex.: aspectos de Nodo Norte/Sul que a própria geometria já implica).
- **Motor de pontuação** (`computeScores`): calcula todas as métricas acima a partir dos
  aspectos e casas parseados — orbe, tipo de aspecto, quais planetas estão envolvidos e o
  tipo de vínculo escolhido decidem o peso de cada contato.
- **Classificação textual** (`classify`): converte os números num texto de veredito
  (tema dominante, se é harmônico/tenso/misto, avisos de poucos dados ou desequilíbrio
  entre faísca e estrutura).
- **Comparações salvas**: lista de sinastrias calculadas, com ordenação por vários
  critérios (compatibilidade, veredito, harmonia, categoria específica...), filtro por
  tipo de vínculo, edição, exclusão e export/import (CSV para leitura tabular, JSON para
  backup/restauração completa, incluindo o texto original colado).
- **Dicionário de significados**: cadastro de interpretação por *padrão* (a combinação
  planeta+aspecto+planeta, ou planeta+casa) — não por sinastria individual — com escopo
  "só esta sinastria" ou "reutilizável" (por tipo de vínculo ou geral), e detecção
  automática dos padrões presentes no texto colado de cada leitura.
- **Configuração de prompt**: monta um prompt pronto (com os aspectos detectados) para
  colar em uma IA externa e gerar interpretações de texto — o app não chama nenhuma API
  de IA sozinho, apenas formata o pedido.
- **Relatório para impressão**: gera uma página HTML de impressão (`window.print()`) com
  as seções escolhidas para uma ou várias sinastrias, incluindo tabela comparativa.

## Arquitetura técnica

- **Single-file**: todo o HTML, CSS e JavaScript vivem em `index.html`. Não há
  dependências externas de runtime além das fontes do Google Fonts (`@import` no CSS).
- **Sem backend**: tudo roda no navegador. Comparações e dicionário ficam em memória
  durante a sessão (persistidos via `localStorage`/estado do app, exportáveis
  manualmente em JSON/CSV para backup).
- **Sem frameworks**: DOM manipulado diretamente via `document.getElementById` /
  `querySelectorAll`, sem React/Vue e sem bundler.
- **Idioma**: interface e comentários de código em português (pt-BR); nomes internos de
  planetas/pontos em inglês (`Sun`, `Moon`, `Node`, `Vertex`, `Fortune`, `Ascendant`,
  `MC`, `IC`, `DSC`, `Lilith`, `Chiron` etc.), para bater com os relatórios astrológicos
  de origem, tipicamente em inglês.
- **Constantes de calibração centralizadas**: quase todo peso/limiar "mágico" do sistema
  vive no objeto `CALIBRATION`, comentado linha a linha com a justificativa astrológica
  e o histórico de ajustes — ver `calculo.md` para o detalhamento completo.

## Limitações conhecidas / avisos do próprio código

- É um sistema heurístico, calibrado "no olho" (nas palavras dos próprios comentários),
  não um cálculo astronômico nem um padrão validado empiricamente.
- Depende inteiramente da qualidade/completude do texto de relatório colado pelo
  usuário; relatórios parciais geram avisos de "poucos dados", mas não impedem o
  cálculo.
- Aspectos entre planetas "outer-outer" (planetas geracionais entre si) são excluídos da
  métrica de Nitidez por serem considerados sinais geracionais, não pessoais do casal.
- Pontos derivados (Parte da Fortuna) e marcadores de casa recebem peso propositalmente
  reduzido/fixo por serem evidência mais "macia" que um aspecto de orbe apertado.
