# Radar de Preços de Combustível — Inteligência para Decisão Logística ⛽📊

Painel de Business Intelligence desenvolvido em Power BI para monitoramento de preços de combustíveis no Brasil, com foco em benchmarking regional e decisão de abastecimento de frota (Etanol × Gasolina). Dados extraídos via engenharia reversa de API pública, tratados em Power Query, modelados em Star Schema e apresentados em dois painéis interativos.

🔗 **[Acessar Dashboard Interativo](LINK_PUBLICAR_NA_WEB_AQUI)**

---

## 📌 Contexto e Problema de Negócio

Uma equipe de Logística/Compras de uma empresa com frota de distribuição precisa decidir em quais praças abastecer e quando negociar contratos de combustível, mas não tem visibilidade estruturada de como o preço varia por região, por tipo de combustível e ao longo do tempo — hoje a decisão é feita sem dado.

Este projeto simula essa ferramenta de decisão, respondendo:
- Onde o combustível está mais barato ou mais caro agora?
- O mercado está subindo, caindo ou estável?
- Vale mais a pena abastecer com Etanol ou Gasolina, e quanto se economiza fazendo essa escolha?
- Qual a confiabilidade estatística de cada leitura de preço?

---

## 🏗️ Arquitetura da Solução

```
Extração (API reversa) → Power Query (tratamento) → Star Schema (modelagem) → DAX (métricas) → Power BI (visualização)
```

---

## 🔍 Extração de Dados — Engenharia Reversa de API

A fonte pública (site que exibe ranking de preços por cidade) renderiza sua tabela via JavaScript (SPA), o que impede a raspagem direta de HTML pelo conector Web padrão do Power Query. A solução adotada:

1. Inspeção via DevTools (aba Network/Fetch-XHR) para identificar a chamada de API interna feita pelo front-end ao navegar entre páginas.
2. Identificação do endpoint JSON e do parâmetro `perPage`, permitindo trazer o dataset completo de um estado em uma única chamada (sem paginação manual).
3. Repetição do processo para os 5 produtos pesquisados pela ANP disponíveis na fonte (Gasolina Comum, Gasolina Aditivada, Etanol Hidratado, Óleo Diesel, GNV), consumidos via Power BI → Obter Dados → Web, com conversão automática de JSON para tabela.

Essa abordagem é mais robusta que raspagem de HTML (estrutura de API é mais estável que classes CSS) e mais performática (JSON é mais leve que HTML completo).

---

## 🧹 Modelagem & Tratamento (Power Query)

### Star Schema
| Tabela | Tipo | Conteúdo |
|---|---|---|
| `fCombustiveis` | Fato | Preço médio, mínimo, máximo, amostras, por município × produto × semana |
| `dLocalidade` | Dimensão | Município, UF, chave composta (`id_localidade`) |
| `dProdutos` | Dimensão | Os 5 combustíveis pesquisados |
| `dCalendario` | Dimensão | Calendário contínuo gerado via `List.Dates` (sem lacunas), com Ano, Mês, Semana do Ano |

### Chave composta na dimensão de localidade
Como municípios homônimos existem em UFs diferentes (ex: mais de uma cidade com o mesmo nome no Brasil), o relacionamento entre fato e dimensão não pode ser feito por `município` isoladamente. Solução: criação de uma coluna-chave concatenada (`município & "-" & UF`) em ambas as tabelas, replicando o padrão de chave composta via coluna única — limitação nativa do motor de relacionamentos do Power BI, que não suporta chaves compostas multi-coluna diretamente.

### Qualidade de dado — deduplicação
Verificação de inconsistências de nome de cidade (ex.: variações de acentuação) via ordenação alfabética antes da criação das dimensões, evitando duplicidade artificial de linhas na `dLocalidade`.

---

## 📐 Metodologia da Fonte de Dados (ANP)

Os dados têm origem na pesquisa semanal de preços de combustíveis realizada pela **ANP (Agência Nacional do Petróleo, Gás Natural e Biocombustíveis)**, consumidos via a API pública do agregador utilizado como fonte.

### Como a coleta funciona
A ANP não pesquisa todos os postos do país — define uma amostra representativa por município, priorizando cidades de maior consumo e **distribuindo a coleta ao longo da semana**. Municípios grandes recebem amostras mais robustas; municípios pequenos podem se limitar a um ou dois postos.

### Impacto direto nas decisões deste projeto
Essa metodologia de coleta rotativa é a causa raiz de duas características tratadas explicitamente na modelagem:

1. **Cobertura irregular ao longo do tempo** — nem toda cidade é pesquisada toda semana. Motivou a troca de **gráficos de linha por gráficos de colunas** em todos os visuais de série temporal: colunas não interpolam entre pontos distantes, evitando sugerir tendências suaves onde há apenas ausência de coleta.
2. **Amostra variável por município** — motivou a inclusão da coluna `Total de Amostras` (postos pesquisados) e `Qtd. Cidades Pesquisadas` como camada de confiabilidade em todas as tabelas de apoio. Um preço médio sustentado por 2 postos carrega peso estatístico diferente de um sustentado por 200.

### Definições oficiais
- **Preço médio:** média aritmética de todos os preços coletados na amostra do município.
- **Preço mínimo/máximo:** extremos encontrados entre os postos pesquisados. A ANP orienta usar o mínimo como referência/meta de economia — base conceitual da métrica **Spread de Preço** deste projeto.

### Validação empírica de hipótese (Etanol × Gasolina)
Antes de assumir que a comparação de paridade entre Etanol e Gasolina poderia estar enviesada por cobertura geográfica assimétrica entre os dois produtos, essa hipótese foi testada (`DISTINCTCOUNT` de municípios por produto, cruzado por semana e por UF). Resultado: sobreposição de ~99% (414 municípios com Etanol vs. 416 com Gasolina) — a ANP aparenta coletar múltiplos produtos na mesma visita ao posto. Hipótese refutada; a paridade calculada em nível agregado é estatisticamente confiável.

---

## 🧮 Principais Medidas DAX

```dax
Preco Medio =
AVERAGE(fCombustiveis[preco_medio])

Spread de Preco =
[Maior Preco] - [Menor Preco]

Razao de Paridade % =
DIVIDE([Preco Medio Etanol], [Preco Medio Gasolina])

Status Recomendacao =
IF([Razao de Paridade %] < 0.70, "ABASTECER ETANOL", "ABASTECER GASOLINA")

Economia Potencial por Litro =
IF([Razao de Paridade %] < 0.70,
   [Preco Medio Gasolina] - [Preco Medio Etanol],
   0)
```

> A condicional na `Economia Potencial por Litro` evita a inconsistência de reportar "economia" quando a Gasolina já é a opção mais vantajosa — o cartão só exibe valor quando há ganho real a comunicar.

### Racional de Negócio: Paridade Etanol × Gasolina
O Etanol tem poder calorífico inferior ao da Gasolina — um veículo flex consome entre 25% e 30% a mais de etanol para percorrer a mesma distância. Por isso, o preço na bomba não conta a história toda: o etanol precisa ser proporcionalmente mais barato para compensar seu menor rendimento. A **regra dos 70%** (se o etanol custa até 70% do preço da gasolina, compensa trocar) é a referência de mercado adotada como premissa deste projeto.

**Limitação assumida:** 70% é uma média nacional. Veículos modernos calibrados para etanol podem compensar até ~75%; motores mais antigos, apenas até ~65%. O dashboard não diferencia por modelo de veículo.

---

## 💡 Decisões Técnicas de Destaque

| Característica do Dado (ANP) | Desafio Analítico | Solução Aplicada |
|---|---|---|
| Coleta amostral rotativa | Lacunas de cobertura em cidades pequenas | Gráficos de **colunas** em vez de linha (evita interpolação falsa) |
| Amostra desigual por UF/cidade | Disparidade de confiabilidade estatística | Colunas `Total de Amostras` e `Qtd. Cidades Pesquisadas` em todas as matrizes |
| Amplitude entre mínimo e máximo | Identificar ineficiência de mercado | Métrica `Spread de Preço` — mede oportunidade de negociação |
| Preços em R$/litro vs. R$/m³ (GNV) | Distorção em agregações diretas (MIN/MAX/Cards) | Slicer de Produto com **Gasolina Comum pré-selecionada**; Página 2 não sofre do problema (compara apenas produtos líquidos) |
| Possível viés de cobertura Etanol×Gasolina | Paridade poderia comparar regiões diferentes | Hipótese testada e refutada via `DISTINCTCOUNT` (~99% de sobreposição) antes de validar a métrica |

---

## 📊 Estrutura do Dashboard

### Página 1 — Panorama de Mercado & Precificação Regional
*Onde está o mercado?*
- **Cards:** Preço Médio, Menor Preço, Maior Preço, Spread de Preço
- **Gráfico de Colunas:** Evolução do preço médio por Semana do Ano
- **Ranking Top 10:** Municípios mais baratos / mais caros
- **Matriz operacional:** Preço médio, mínimo, máximo, spread, ponderado e amostras por UF

### Página 2 — Análise de Decisão & Paridade de Combustíveis
*O que comprar?*
- **Cards:** Razão de Paridade, Indicador de Recomendação, Economia Potencial por Litro
- **Gráfico de Colunas:** Histórico da paridade Etanol/Gasolina, com linha de referência em 70%
- **Matriz de Oportunidades:** Paridade por UF, com formatação condicional (verde = Etanol vence / vermelho = Gasolina vence)

---

## 🔑 Principais Insights

- A cobertura de GNV é substancialmente menor que a de combustíveis líquidos (148 vs. ~417 registros), refletindo a barreira de infraestrutura de gasodutos/compressão do produto.
- A paridade Etanol/Gasolina varia semana a semana dentro do próprio período analisado — em algumas semanas a Gasolina foi a opção recomendada, em outras o Etanol, evidenciando que a decisão de abastecimento não é estática e exige monitoramento contínuo.
- Estados com poucas amostras (ex.: Amapá, com apenas 2 leituras de posto para determinado produto) apresentam preços médios estatisticamente menos confiáveis que estados com amostras robustas (ex.: Bahia, 200+ leituras) — reforçando a importância de nunca ler preço médio isoladamente, sem considerar o tamanho da amostra que o sustenta.

---

## 📁 Estrutura do Repositório

```
├── imagens/                        # Prints e GIF de demonstração do dashboard
├── docs/                           # Documentação técnica detalhada por métrica/decisão
├── radar_combustivel.pbix          # Arquivo do Power BI Desktop
└── README.md                       # Este documento
```

---

## 🛠️ Ferramentas Utilizadas
Power BI Desktop · Power Query (linguagem M) · DAX · API REST (engenharia reversa)

---

## 👤 Autor
[SEU NOME] — [LINK DO PORTFÓLIO] · [LINK DO LINKEDIN]
