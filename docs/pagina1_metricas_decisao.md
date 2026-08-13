# Página 1 — Panorama de Mercado & Precificação Regional
Documentação técnica das métricas DAX que alimentam os cartões, gráfico e matriz operacional desta página.

---

## MÉTRICA / CARTÃO 1: Preço Médio

**Tabela de Origem:** `fCombustiveis[preco_medio]`

**Tipo de Agregação:** Média Simples (AVERAGE)

**Fórmula Matemática:**
```
Preço Médio = ∑ (Preço Médio_i) / n
```

**Código DAX:**
```dax
[Preço Médio] =
AVERAGE(fCombustiveis[preco_medio])
```

### Relevância de Negócio
Estipula a linha de base (baseline) do mercado sob os filtros aplicados — é o número de referência contra o qual os demais cartões (Menor, Maior, Spread) e os gráficos da página são lidos.

### Nota Técnica — Limitação Conhecida e Alternativa Descartada
A coluna `preco_medio` já é, ela própria, uma média — calculada pela ANP por cidade, a partir de N postos pesquisados (coluna `amostras`). Ao aplicar `AVERAGE` sobre essa coluna no Power BI, o cálculo resultante é, na prática, uma **média de médias**, o que é estatisticamente delicado: uma cidade com 3 postos pesquisados passa a pesar igual a uma cidade com 20 postos na média final, distorcendo o resultado — a cidade de amostra pequena acaba com peso desproporcional ao que ela de fato representa no mercado.

A alternativa estatisticamente mais correta seria uma **média ponderada pelas amostras**, dando peso proporcional ao volume de postos pesquisados em cada cidade:

```dax
[Preço Médio Ponderado] =
DIVIDE(
    SUMX(fCombustiveis, fCombustiveis[preco_medio] * fCombustiveis[amostras]),
    SUM(fCombustiveis[amostras])
)
```

**Decisão:** manteve-se a média simples como métrica oficial do cartão nesta versão do projeto, por simplicidade e legibilidade imediata para o usuário do relatório. A limitação acima é documentada conscientemente, não por desconhecimento — a versão ponderada permanece disponível no relatório como métrica de apoio na matriz operacional, e como evolução natural do projeto.

---

## MÉTRICA / CARTÃO 2: Menor Preço Encontrado

**Tabela de Origem:** `fCombustiveis[preco_medio]`

**Tipo de Agregação:** Mínimo (MIN)

**Fórmula Matemática:**
```
Menor Preço = MIN(Preço Médio_i), para i = 1...n no contexto de filtro atual
```

**Código DAX:**
```dax
[Menor Preço] =
MIN(fCombustiveis[preco_medio])
```

### Relevância de Negócio
Ancora o "piso" do mercado dentro do filtro aplicado (produto, UF, período) — responde diretamente "onde está mais barato agora". Isoladamente é um resumo executivo; ganha poder de decisão quando cruzado com o gráfico "Top 10 Mais Baratas" e a matriz operacional, que revelam *qual* cidade sustenta esse valor.

### Nota Técnica
Opera sobre `preco_medio` (média por cidade), não sobre `preco_minimo` (menor valor pesquisado dentro de uma única cidade) — são grandezas diferentes e não devem ser confundidas.

---

## MÉTRICA / CARTÃO 3: Maior Preço Encontrado

**Tabela de Origem:** `fCombustiveis[preco_medio]`

**Tipo de Agregação:** Máximo (MAX)

**Fórmula Matemática:**
```
Maior Preço = MAX(Preço Médio_i), para i = 1...n no contexto de filtro atual
```

**Código DAX:**
```dax
[Maior Preço] =
MAX(fCombustiveis[preco_medio])
```

### Relevância de Negócio
Ancora o "teto" do mercado dentro do filtro aplicado — responde "onde evitar comprar, ou onde negociar com mais força". Espelha o Cartão 2 na lógica e alimenta, junto com ele, o cálculo do Spread (Cartão 4).

### Nota Técnica
Mesma ressalva do Cartão 2: opera sobre `preco_medio` (média por cidade), não sobre `preco_maximo` (maior valor pesquisado dentro de uma única cidade).

---

## MÉTRICA / CARTÃO 4: Spread / Amplitude de Preço

**Tabela de Origem:** Medida derivada (não acessa `fCombustiveis` diretamente)

**Tipo de Agregação:** Diferença entre medidas

**Fórmula Matemática:**
```
Spread = Maior Preço − Menor Preço
```

**Código DAX:**
```dax
[Spread de Preço] =
[Maior Preço] - [Menor Preço]
```

### Relevância de Negócio
Mede o tamanho da distância entre as pontas de preço no filtro atual — ou seja, o tamanho da oportunidade de negociação/economia disponível.

Um spread pequeno sinaliza mercado homogêneo naquela região/produto (pouca margem para "caçar preço"; a escolha da praça de abastecimento importa menos). Um spread grande sinaliza mercado fragmentado, onde escolher bem onde comprar gera economia real. Interpretação alinhada à orientação oficial da ANP: a diferença entre mínimo e máximo indica o nível de competição entre postos.

### Nota Técnica
Medida derivada de outras duas medidas (não uma nova agregação sobre a tabela) — boa prática de DAX: reaproveita `[Maior Preço]` e `[Menor Preço]`, herdando automaticamente qualquer ajuste futuro feito nelas.

**Observação de comportamento esperado:** no grão mais atômico da tabela (uma cidade + um produto + uma semana = uma linha), `MIN` e `MAX` operam sobre um único valor, logo `Spread = 0` por construção. O Spread só ganha significado analítico em níveis agregados (múltiplas cidades, múltiplos produtos ou visão nacional/estadual).

---

## GRÁFICO: Evolução do Preço Médio por Semana do Ano

**Tipo:** Gráfico de Colunas (não Linha)

**Eixo X:** `Semana do Ano` (`dCalendario`)

**Eixo Y:** `[Preço Médio]`

### Nota Técnica — Escolha de Colunas em vez de Linha
Testou-se inicialmente um gráfico de linha, que se mostrou enganoso: por natureza, uma linha desenha uma reta entre dois pontos quaisquer, independentemente da distância entre eles no eixo X. Como a coleta da ANP é rotativa (nem toda cidade é pesquisada toda semana), isso gerava a aparência de tendências suaves e contínuas em trechos onde havia apenas ausência de cobertura amostral — não uma variação real de mercado.

**Decisão:** substituição por Gráfico de Colunas. Colunas não interpolam: uma semana sem dado simplesmente não gera barra, comunicando a lacuna de forma honesta em vez de mascará-la com uma transição visual inexistente.

### Nota Técnica — Filtro de Produto obrigatório para leitura correta
Sem filtro de Produto, o eixo Y mistura preços de GNV (medido em R$/m³) com combustíveis líquidos (R$/litro), o que distorce qualquer agregação (MIN, MAX, AVERAGE) por comparar grandezas de unidades diferentes. **Solução aplicada:** slicer de Produto com **Gasolina Comum pré-selecionada** por padrão, com interação editável pelo usuário. Título do gráfico dinâmico (`SELECTEDVALUE`) reforça visualmente qual produto está em exibição.

### Nota Técnica — Exceção de interação com o slicer de UF
Diferente dos demais elementos da página, este gráfico **não** é filtrado pelo slicer de UF (interação definida como "Nenhum"). Motivo: a cobertura de coleta por estado é ainda mais esparsa que a cobertura nacional agregada — testes mostraram estados com apenas 1-2 semanas de dado no período, tornando a leitura de tendência praticamente inviável quando isolada por UF. Mantendo o gráfico sempre em visão nacional, garante-se robustez estatística suficiente para leitura de tendência.

---

## TABELA DE APOIO: Matriz Operacional

**Linhas:** `estado_uf` (com drill-down para `município`)

**Colunas:** Preço Médio, Menor Preço, Maior Preço, Spread de Preço, Preço Médio (Ponderado), Total de Amostras

### Função
Consulta detalhada linha a linha, complementando (não repetindo) os gráficos — enquanto os gráficos respondem "o que se destaca" (extremos via ranking), a matriz responde "eu quero ver tudo, com o detalhe de confiabilidade".

### Nota Técnica — Coluna de Amostras
A inclusão de `Total de Amostras` (`SUM(fCombustiveis[amostras])`) permite identificar leituras estatisticamente frágeis. Exemplo real observado no projeto: o estado do Amapá (AP) apresentou preço médio sustentado por apenas 2 leituras de posto, enquanto a Bahia (BA) apresentou o mesmo tipo de métrica sustentada por mais de 200 leituras — sem essa coluna, os dois números pareceriam igualmente confiáveis ao leitor.

### Nota Técnica — Slicer de Produto universal
Diferente do gráfico de tendência, a matriz **é** filtrada pelo slicer de UF normalmente (não há exceção de interação aqui) — o benchmarking regional é o propósito central deste elemento.
