# Sinastria — Calculadora

Ferramenta em uma única página HTML que lê o texto de sinastria gerado por um app de astrologia (aspectos entre planetas + sobreposição de casas) e devolve uma leitura estruturada da conexão entre duas pessoas: força, harmonia, compatibilidade geral e uma separação entre "atração imediata" e "estrutura de longo prazo".

Não é um veredito científico — é uma leitura simbólica baseada em pesos heurísticos entre planetas, aspectos e casas, pensada para uso pessoal e comparação entre vínculos.

## O que faz

1. **Cole o texto bruto da sinastria** (o formato tipo `VL's Sun in Pisces Trine DD's Sun in Cancer (Orb: 4°31')`, incluindo aspectos e sobreposição de casas). A calculadora ignora o que não reconhece, então dá pra colar o arquivo inteiro (mapas individuais + sinastria + casas).
2. Os nomes das duas pessoas são preenchidos automaticamente se o texto tiver uma linha `Sinastria entre X e Y`, ou podem ser digitados à mão.
3. Escolha o tipo de vínculo (romântico, amizade ou família) — isso muda a leitura do veredito final.
4. Clique em **Calcular sinastria** para ver os resultados.

## Métricas calculadas

- **Força da conexão** — % de aspectos com orbe apertado (bem exatos). Indica o quanto o mapa entre as duas pessoas é "carregado".
- **Harmonia geral** — % do peso astrológico vindo de aspectos fáceis (trígono, sextil, conjunção) vs. tensos (quadratura, oposição), considerando todos os aspectos.
- **Compatibilidade Geral** — combina Pegação e Estrutura por média geométrica (não simples soma/média), então só fica alta quando os dois eixos abaixo são razoavelmente bons — um eixo forte não compensa sozinho o outro fraco.
- **Pegação (imediato)** — harmonia isolada nos eixos de atração e reconhecimento físico: Sol-Lua, Vênus-Marte, Vênus-Vênus, Marte-Marte, Lua-Lua, Lilith com Vênus/Marte, Ascendente com planetas pessoais.
- **Estrutura (longo prazo)** — harmonia isolada nos eixos de permanência: Saturno, Nodo, Vértice e Quíron com planetas pessoais/ângulos, e MC/IC com pessoais.
- **Distribuição por categoria** — intelectual, emocional, sexual e prático, como percentuais.
- **Marcadores especiais** — contatos narrativos específicos, cada um classificado como harmônico, ambivalente ou tenso:
  - Saturno–pessoais (indicador de compromisso)
  - Nodo–pessoais (sensação de destino)
  - Eixo dos Nodos
  - Vértice (encontro "fatídico")
  - Quíron (ferida compartilhada)
  - Lilith (magnetismo)
- **Sobreposição de casas** — em quais casas os planetas de uma pessoa caem na carta da outra, incluindo convergências recíprocas.
- **Veredito** — um título e descrição gerados a partir da combinação de todas as métricas acima e do tipo de vínculo escolhido.

## Comparar várias sinastrias

Toda sinastria calculada entra automaticamente numa lista de comparação (seção 3), permitindo colocar vários pares — ou vários candidatos para a mesma pessoa — lado a lado.

Recursos da lista:
- **Ordenar** por mais recente, maior compatibilidade geral, maior harmonia, maior força ou mais marcadores especiais.
- **Ver detalhes completos** de qualquer entrada salva.
- **Editar** nome, tipo de vínculo ou o texto colado de uma entrada.
- **Recalcular** uma entrada (ou todas de uma vez) aplicando a lógica de pontuação mais atual sobre o texto bruto original salvo — útil depois de ajustar os pesos/regras da calculadora.
- **Remover** entradas individualmente ou limpar tudo.
- **Exportar** para CSV ou JSON.
- **Importar** um JSON exportado anteriormente (com validação de estrutura).

## Armazenamento

Os dados ficam salvos em `localStorage`, no navegador, associados a esta página — não há backend nem envio de dados para fora. Se o navegador não permitir `localStorage` (comum em aba anônima), a calculadora avisa e continua funcionando só em memória: os resultados somem ao recarregar a página, então vale exportar em CSV/JSON antes de fechar.

## Como usar

Basta abrir o arquivo `calculadora-sinastria.html` em qualquer navegador — não precisa de servidor, instalação ou build. Único recurso externo é a fonte do Google Fonts (Cormorant Garamond, Space Grotesk, JetBrains Mono), carregada via CDN.

## Stack

HTML, CSS e JavaScript puro em um único arquivo, sem dependências ou frameworks.
