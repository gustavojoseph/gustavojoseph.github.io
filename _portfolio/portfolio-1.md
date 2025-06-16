---
title: "Datação de Ciclos Econômicos na economia brasileira com R"
excerpt: "Identificação de fases de expansão e recessão na economia brasileira por meio de técnicas de datação cíclica<br/><img src='/images/ciclos_capa.png'>"
collection: portfolio
---

A análise de ciclos econômicos é uma das práticas mais relevantes para compreender os movimentos recorrentes de expansão e contração da atividade econômica. Esses ciclos, embora não ocorram de maneira regular, afetam diretamente o comportamento de consumidores, empresas e governos. Para as empresas, identificar corretamente o estágio do ciclo econômico pode ser crucial para a definição de estratégias de investimento, marketing, precificação e expansão.

Durante períodos de hiato do produto negativo, isto é, quando o PIB observado está abaixo de seu potencial, é comum que empresas adotem posturas mais conservadoras, focando em contenção de custos e preservação de capital. Já em cenários de hiato positivo, a maior demanda e o otimismo com a economia incentivam decisões mais arrojadas, como aumento da capacidade produtiva ou ampliação da participação de mercado.

Além disso, a datação precisa do ciclo econômico é uma ferramenta fundamental para a formulação de políticas econômicas. A teoria dos ciclos econômicos sustenta que políticas anticíclicas, especialmente a política monetária, podem suavizar os efeitos de choques e promover maior estabilidade macroeconômica. Para tanto, os Bancos Centrais e instituições governamentais dependem de métodos confiáveis para identificar, em tempo oportuno, os momentos de pico e vale da atividade econômica. 

Embora existam diversas abordagens para a identificação e a datação dos ciclos econômicos, este artigo explora o método desenvolvido por **Harding e Pagan (2002)**. Esse procedimento segue a tradição inaugurada por **Burns e Mitchell (1946)**, que define o ciclo econômico como “flutuações recorrentes, mas não periódicas, na atividade econômica agregada”. Harding e Pagan operacionalizam essa definição com um algoritmo robusto, denominado **BBQ (Bry-Boschan Quarterly)**, que permite detectar fases de recessão e expansão com base em regras pré-definidas sobre a duração mínima das fases e dos ciclos.

A principal vantagem do método é a entrega de pontos de reversão claros e gráficos, tornando-o amplamente utilizado por instituições como a OCDE e o Banco Central Europeu. No Brasil, o método é utilizado em estudos complementares ao trabalho do **CODACE (Comitê de Datação de Ciclos Econômicos)**, coordenado pela FGV/IBRE.

## Aplicação em R: PIB Brasileiro

Para exemplificar a aplicação do algoritmo de Harding e Pagan, foi utilizada a série trimestral do PIB com ajuste sazonal, disponibilizada pelo **SIDRA/IBGE**. A seguir, apresenta-se o código completo utilizado na extração, transformação, datação e visualização dos ciclos econômicos.

### 1. Carregamento dos Pacotes
```R
library(tidyverse)  # Tratamento e visualização dos dados
library(BCDating)   # Algoritmo BBQ
library(sidrar)     # Acesso ao SIDRA/IBGE
library(tsibble)    # Manipulação tidy de séries temporais
```

### 2. Extração e Tratamento dos Dados
A série utilizada é a Tabela 1621 do SIDRA, que fornece o índice encadeado do volume do PIB trimestral com ajuste sazonal.

```R
dados <- sidrar::get_sidra(api = "/t/1621/n1/all/v/all/p/all/c11255/90707/d/v584%202") |>
  dplyr::select(11,5) |>
  dplyr::mutate(
    Tri  = as.integer(str_extract(Trimestre, "\\d+")),
    Ano  = as.integer(str_extract(Trimestre, "\\d+$")),
    Mes  = Tri * 3 - 2,
    Data = as.Date(paste(Mês, "/1/", Ano, sep = ""), format = "%m/%d/%Y")
  ) |>
  dplyr::select(Data, Valor) |>
  as_tsibble() |>
  dplyr::rename(data = Data, pib = Valor) |>
  dplyr::mutate(ln_pib = log(pib))
```

### 3. Conversão para Série Temporal
Para aplicar o algoritmo de datação, a série é transformada em um objeto do tipo `ts` com frequência trimestral:

```R
dados_ts <- tsibble::ts(
  data = dados$ln_pib,
  start = c(year(min(dados$data)), quarter(min(dados$data))),
  frequency = 4
)
```

### 4. Datação dos Ciclos Econômicos
Aplicamos o algoritmo BBQ, com parâmetros que impõem uma duração mínima para fases e ciclos:

```R
ciclo <- BCDating::BBQ(
  y        = dados_ts,
  minphase = 2,   # Fase mínima: 2 trimestres
  mincycle = 5    # Ciclo mínimo: 5 trimestres
)
```

O resultado apresenta os trimestres identificados como **picos** (início de recessão) e **vales** (fim da recessão).

| #  | Phase      | Start  | End    | Duration | LevStart | LevEnd | Amplitude |
|----|------------|--------|--------|----------|----------|--------|-----------|
| 1  | Expansion  | <NA>   | 2001Q1 | NA       | NA       | 5      | NA        |
| 2  | Recession  | 2001Q1 | 2001Q4 | 3        | 5        | 5      | 0.0       |
| 3  | Expansion  | 2001Q4 | 2002Q4 | 4        | 5        | 5      | 0.1       |
| 4  | Recession  | 2002Q4 | 2003Q2 | 2        | 5        | 5      | 0.0       |
| 5  | Expansion  | 2003Q2 | 2008Q3 | 21       | 5        | 5      | 0.3       |
| 6  | Recession  | 2008Q3 | 2009Q1 | 2        | 5        | 5      | 0.1       |
| 7  | Expansion  | 2009Q1 | 2014Q1 | 20       | 5        | 5      | 0.2       |
| 8  | Recession  | 2014Q1 | 2016Q4 | 11       | 5        | 5      | 0.1       |
| 9  | Expansion  | 2016Q4 | 2019Q4 | 12       | 5        | 5      | 0.1       |
| 10 | Recession  | 2019Q4 | 2020Q2 | 2        | 5        | 5      | 0.1       |
| 11 | Expansion  | 2020Q2 | <NA>   | NA       | 5        | NA     | NA        |

| Tipo        | Amplitude | Duração Média |
|-------------|-----------|----------------|
| Expansão    | 0.1       | 14.2 trimestres |
| Recessão    | 0.1       | 4.0 trimestres  |

A tabela apresenta uma cronologia dos ciclos econômicos no Brasil desde o início dos anos 2000 até o período pós-pandemia, distinguindo fases de expansão e recessão com base na duração e na amplitude das variações do PIB. Essa datação cíclica fornece uma visão clara dos momentos de crescimento sustentado e de retração econômica, sendo fundamental para a análise macroeconômica.

As expansões predominam na série, tanto em número quanto em duração. Foram registradas seis fases de crescimento, com destaque para o período entre 2003Q2 e 2008Q3, que durou 21 trimestres, um intervalo marcado pelo boom das commodities, estabilidade fiscal e maior inserção do Brasil nos mercados globais. Apesar da extensão temporal, a amplitude média das expansões foi modesta (0,1), o que sugere uma trajetória de crescimento contínuo, porém de intensidade limitada. A fase de crescimento iniciada em 2020Q2, após o choque da pandemia, ainda está em curso, caracterizando o momento atual da economia brasileira como um período de expansão.

No que diz respeito às recessões, foram identificados cinco episódios, com duração média de quatro trimestres. Esses momentos de contração tendem a ser mais curtos do que os períodos de expansão, mas ocorrem com relativa frequência. A recessão mais severa ocorreu entre 2014Q1 e 2016Q4, prolongando-se por 11 trimestres ao refletir uma combinação de crise política, colapso fiscal e perda de confiança. Ainda assim, a amplitude média também foi de 0,1, indicando que as retrações foram, em termos quantitativos, relativamente rasas, o que pode indicar uma economia que ajusta lentamente, sem grandes oscilações abruptas no PIB agregado.

Essa assimetria, com expansões longas e recessões curtas, é típica de economias com certa resiliência macroeconômica. No entanto, a baixa amplitude observada tanto nas altas quanto nas quedas aponta para um ciclo econômico com volatilidade reduzida, pelo menos no agregado. Isso não significa, contudo, que os efeitos distributivos e setoriais sejam igualmente suaves: a média esconde as disparidades regionais e os impactos desiguais entre setores da economia. Por isso, a leitura dos ciclos deve ser sempre complementada por análises qualitativas e estruturais.

### 5. Visualização dos Resultados
O gráfico abaixo apresenta o PIB brasileiro com sombreamento dos períodos de recessão identificados pelo algoritmo:

```R
# Organizando datas de pico e vale
pico <- ciclo@peaks
vale <- ciclo@troughs

data <- dados |>
  dplyr::mutate(tempo = row_number(),
         inicio = ifelse(tempo %in% pico, "PICO", NA),
         fim    = ifelse(tempo %in% vale, "VALE", NA)) |>
  dplyr::select(data, inicio, fim)

inicio <- data |> dplyr::filter(inicio == "PICO") |> dplyr::select(data) |> dplyr::rename("inicio" = "data")
fim    <- data |> dplyr::filter(fim    == "VALE") |> dplyr::select(data) |> dplyr::rename("fim"    = "data")

data_g <- data.frame(
  inicio = c(inicio$inicio, rep(NA, sum(is.na(data$inicio)))),
  fim    = c(fim$fim, rep(NA, sum(is.na(data$fim))))
)

# Gráfico com ggplot
dados |>
  ggplot2::ggplot(aes(x = data, y = pib)) +
  
  # Sombreamento das recessões
  ggplot2::geom_rect(
    data = data_g,
    inherit.aes = FALSE,
    aes(xmin = inicio, xmax = fim, ymin = -Inf, ymax = Inf, fill = "Recessão"),
    alpha = 0.4,
    color = NA
  ) +

  # Linha do PIB
  ggplot2::geom_line(color = "#000080", size = 1.25) +

  # Eixos e escalas
  ggplot2::scale_x_date(breaks = "2 years", date_labels = "%Y") +
  ggplot2::scale_y_continuous(n.breaks = 8) +

  # Legenda personalizada
  ggplot2::scale_fill_manual(name = NULL, values = c("Recessão" = "gray")) +

  # Tema e layout
  ggplot2::theme_minimal() +
  ggplot2::theme(
    plot.title = element_text(face = "bold", size = 14),
    plot.subtitle = element_text(size = 12),
    plot.caption = element_text(hjust = 0, face = "bold", size = 9),
    legend.position = "top"
  ) +

  # Títulos e rótulos
  ggplot2::labs(
    title    = "Datação dos Ciclos Econômicos - Brasil",
    subtitle = "Recessão datada pelo algoritmo de Harding-Pagan (2002)",
    x        = NULL,
    y        = "Índice",
    caption  = "Fonte: Sistema IBGE de Recuperação Automática - SIDRA"
  )
```

  <img src="/images/ciclos.png" alt="descrição da imagem">
  
A identificação dos ciclos econômicos é essencial não apenas para estudos empíricos da macroeconomia, mas também para decisões estratégicas públicas e privadas. Ao aplicar o método de **Harding e Pagan (2002)** sobre a série trimestral do PIB brasileiro, é possível obter uma representação clara dos momentos de recessão da economia. Empresas, formuladores de política e analistas econômicos podem utilizar essas informações para avaliar riscos, antecipar tendências e alinhar estratégias a contextos mais ou menos favoráveis do ciclo econômico.

### Referências

1. Burns, Arthur F. & Mitchell, Wesley C., (1946). Measuring Business Cycles. National Bureau of Economic Research, Inc, <https://EconPapers.repec.org/RePEc:nbr:nberbk:burn46-1>.

2. Harding, D., & Pagan, A. (2002). Dissecting the cycle: a methodological investigation. Journal of monetary economics, 49(2), 365-381.
