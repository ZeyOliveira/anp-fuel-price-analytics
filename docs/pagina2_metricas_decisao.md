# Página 2 — Análise de Decisão & Paridade de Combustíveis
Documentação técnica das métricas DAX que alimentam os cartões, gráfico e matriz de oportunidades desta página.

---

## Medidas de Apoio (reaproveitadas da Página 1, filtradas por produto)

```dax
[Preço Médio Etanol] =
CALCULATE([Preço Médio], dProdutos[produto] = "ETANOL HIDRATADO")

[Preço Médio Gasolina] =
CALCULATE([Preço Médio], dProdutos[produto] = "GASOLINA COMUM")
```

Ambas reaproveitam a medida `[Preço Médio]` já validada na Página 1, apenas aplicando um filtro de produto via `CALCULATE`. Essa reutilização reduz a superfície de erro: qualquer ajuste futuro na lógica de `[Preço Médio]` se propaga automaticamente para as duas.

---

## MÉTRICA / CARTÃO 1: Razão de Paridade Média (Etanol/Gasolina)

**Tabela de Origem:** `fCombustiveis[preco_medio]`, filtrado por `dProdutos[produto]`

**Tipo de Agregação:** Razão entre duas médias filtradas

**Fórmula Matemática:**
```
Paridade = Preço Médio Etanol ÷ Preço Médio Gasolina
```

**Código DAX:**
```dax
[Razão de Paridade %] =
DIVIDE([Preço Médio Etanol], [Preço Médio Gasolina])
```

### Relevância de Negócio
Base da regra de decisão de frota flex: quando o etanol custa até 70% do preço da gasolina, ele rende mais por real gasto, mesmo consumindo mais litros por trajeto (menor poder calorífico). Ver seção "Racional de Negócio" no README principal para a fundamentação física/econômica completa.

### Nota Técnica — Validação Empírica de Cobertura
Antes de validar esta medida em nível agregado (Brasil/estado), testou-se a hipótese de que Etanol e Gasolina poderiam ter cobertura de cidades/semanas divergente entre si — o que enviesaria a paridade regionalmente (comparando, por exemplo, etanol pesquisado numa região com gasolina pesquisada em outra, na mesma janela temporal).

**Teste aplicado:**
```dax
[Teste Cobertura Etanol] =
CALCULATE(DISTINCTCOUNT(fCombustiveis[municipio]), dProdutos[produto]="ETANOL HIDRATADO")

[Teste Cobertura Gasolina] =
CALCULATE(DISTINCTCOUNT(fCombustiveis[municipio]), dProdutos[produto]="GASOLINA COMUM")
```

**Resultado:** cobertura de 414 municípios (Etanol) vs. 416 municípios (Gasolina) — sobreposição de aproximadamente 99%, tanto por semana quanto por UF. A hipótese de viés geográfico foi **refutada**: a ANP aparenta coletar múltiplos produtos na mesma visita/posto, tornando a paridade em nível agregado estatisticamente confiável sem necessidade de tratamento adicional.

### Refinamento Futuro Considerado e Adiado
Avaliou-se "blindar" a medida via `INTERSECT` das tabelas de municípios de cada produto, restringindo o cálculo apenas a cidades/semanas com registro simultâneo de ambos. Descartado por ora: o teste de cobertura acima mostrou sobreposição de ~99%, tornando o ganho de precisão marginal frente à complexidade adicional de DAX (múltiplas `CALCULATETABLE` aninhadas, potencial necessidade de intersecção também por semana, não só por município). Mantido como possível evolução do projeto.

---

## MÉTRICA / CARTÃO 2: Indicador de Recomendação

**Tabela de Origem:** Medida derivada de `[Razão de Paridade %]`

**Tipo de Agregação:** Condicional (IF)

**Fórmula Matemática:**
```
Recomendação = SE(Paridade < 0,70; "ABASTECER ETANOL"; "ABASTECER GASOLINA")
```

**Código DAX:**
```dax
[Status Recomendação] =
IF(
    [Razão de Paridade %] < 0.70,
    "ABASTECER ETANOL",
    "ABASTECER GASOLINA"
)
```

### Relevância de Negócio
Traduz o número da paridade em uma ação direta — a peça central da tese de "decisão de frota" da página. É a ponte entre dado (a paridade, um número) e decisão (o que fazer com ele).

### Nota Técnica
A regra de 70% é a referência mais comum no mercado brasileiro para motor flex; pode variar entre 65% e 75% conforme a eficiência/calibração do motor (ver README para detalhamento). Valor documentado como premissa assumida do projeto, não como constante universal.

---

## MÉTRICA / CARTÃO 3: Economia Potencial por Litro (R$)

**Tabela de Origem:** Medida derivada de `[Preço Médio Etanol]` e `[Preço Médio Gasolina]`

**Tipo de Agregação:** Diferença condicional

**Fórmula Matemática:**
```
Economia = SE(Paridade < 0,70; Preço Gasolina − Preço Etanol; 0)
```

**Código DAX:**
```dax
[Economia Potencial por Litro] =
IF(
    [Razão de Paridade %] < 0.70,
    [Preço Médio Gasolina] - [Preço Médio Etanol],
    0
)
```

### Relevância de Negócio
Fecha o encadeamento lógico dos 3 cartões: quanto é a paridade → o que fazer → quanto se ganha fazendo isso.

### Nota Técnica — Correção de Inconsistência Identificada
A primeira versão desta medida considerava a diferença absoluta de preço (`|Gasolina − Etanol|`) independentemente da recomendação. Isso gerava uma inconsistência: em cenários com paridade acima de 70% (ex.: Gasolina R$5,00, Etanol R$4,00 → paridade 80%), a subtração ainda retornava um valor positivo (R$1,00), sugerindo "economia" mesmo quando a Gasolina já era objetivamente a opção mais vantajosa por litro rodado — o etanol, nesse cenário, rende menos do que essa diferença de preço nominal sugere.

**Correção aplicada:** a medida agora retorna R$0,00 sempre que a recomendação for Gasolina, ficando "muda" exatamente quando não há economia real a comunicar — consistente com o Cartão 2 e sem risco de leitura equivocada.

**Alternativa avaliada e descartada:** ponderar a economia pelo fator de rendimento (`Preço Gasolina − (Preço Etanol / 0,70)`), calculando um "ganho real" mesmo acima do limiar. Descartada por gerar valores negativos de interpretação pouco intuitiva em um cartão de KPI ("economia negativa"), preferindo-se a solução mais direta (zerar o cartão).

---

## GRÁFICO: Histórico de Paridade por Semana do Ano

**Tipo:** Gráfico de Colunas (não Linha/Combinado)

**Eixo X:** `Semana do Ano` (`dCalendario`)

**Eixo Y:** `[Razão de Paridade %]`

**Detalhe:** Linha de referência constante em 70%, adicionada via Analytics Pane (Linha Constante) — sem necessidade de medida DAX adicional para desenhá-la.

### Nota Técnica — Escolha de Colunas em vez de Linha
Decisão alinhada com o gráfico equivalente da Página 1, pela mesma causa raiz: gráfico de linha contínua interpolaria entre semanas sem cobertura simultânea de Etanol e Gasolina, sugerindo tendência suave onde há apenas lacuna de coleta.

### Formatação Condicional (opcional, aplicada nas colunas)
Regra de cor: `< 0.70` → verde (Etanol vence); `≥ 0.70` → vermelho/laranja (Gasolina vence). Reforça a leitura instantânea da mensagem do gráfico sem exigir comparação manual com a linha de referência.

---

## TABELA DE APOIO: Matriz de Oportunidades por Município

**Linhas:** Estado (UF), com drill-down para Município

**Colunas:** Razão de Paridade %, Preço Médio Etanol, Preço Médio Gasolina, Total de Amostras, Qtd. Cidades Pesquisadas

**Formatação Condicional:** Verde (Paridade < 70%, Etanol vence) / Vermelho (Paridade ≥ 70%, Gasolina vence)

**Ordenação:** Paridade % (crescente)

### Função
Traduz o histórico temporal (gráfico) em recorte geográfico — "onde" a recomendação se aplica, complementando o "quando" do gráfico.

### Nota Técnica — Células sem par
Municípios com apenas um dos dois produtos pesquisados no período retornam `BLANK()` na coluna de Paridade (via `DIVIDE`), em vez de erro ou valor inventado. Dado o resultado do teste de cobertura (~99% de sobreposição), a incidência esperada dessas células é baixa.

---

## Nota de Arquitetura — Slicers desta página

O campo **Produto não é um slicer** nesta página. A paridade compara dois produtos fixos (`ETANOL HIDRATADO` vs. `GASOLINA COMUM`), cujos contextos são tratados internamente via `CALCULATE` em cada medida — um slicer de produto não teria função de filtro aqui, pois ambos os lados da comparação já são fixados na fórmula.
