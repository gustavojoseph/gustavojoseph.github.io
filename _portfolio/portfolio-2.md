---
title: "Política Monetária Brasileira: Uma Análise dos Períodos Contracionistas e Expansionistas"
excerpt: "Estimando o Juro Neutro e o Juro Real com Dados do Boletim Focus <br/><img src='/images/capa_pom.monetaria.png'>"
collection: portfolio
---

A condução da política monetária é uma das tarefas mais sensíveis e estratégicas do Banco Central. Ao definir a taxa básica de juros (Selic), a autoridade monetária busca equilibrar o binômio inflação e atividade econômica.

No Brasil, as decisões sobre a taxa básica de juros são responsabilidade do **Comitê de Política Monetária (Copom)**, órgão do Banco Central encarregado de definir a estratégia monetária do país. Quando o Copom opta por reduzir os juros, o crédito se torna mais acessível, estimulando o consumo das famílias e os investimentos das empresas, trata-se de uma política monetária expansionista, voltada a impulsionar a atividade econômica. Por outro lado, ao elevar os juros, o comitê busca conter a demanda agregada e reduzir as pressões inflacionárias, o que caracteriza uma política contracionista.

**Mas como identificar, na prática, se a política vigente está de fato estimulando ou restringindo a atividade econômica?**

O **juro neutro**, também conhecido como taxa de juros natural, é um conceito macroeconômico fundamental. Trata-se de uma referência teórica que indica o nível da taxa de juros real capaz de equilibrar a economia no longo prazo, mantendo a inflação estável e o produto próximo ao seu potencial. Quando a taxa Selic real se encontra acima desse nível, a política monetária tende a exercer um efeito contracionista, ao restringir a demanda agregada. Por sua vez, quando está abaixo, o estímulo à atividade econômica caracteriza uma política expansionista. 

Por ser um parâmetro não observável diretamente, o juro neutro precisa ser estimado a partir de aproximações. No contexto brasileiro, uma proxy simplificada da taxa neutra pode ser construída a partir da taxa Selic esperada para três anos à frente (t+3), conforme projeções do Boletim Focus, deflacionada pela expectativa de inflação (IPCA) para o mesmo horizonte. Essa escolha metodológica busca captar uma visão de equilíbrio intertemporal do mercado, minimizando os ruídos conjunturais de curto prazo e se aproximando do componente estrutural da taxa de juros. 

$$
\text{Juro Neutro} = \left( \frac{1 + \text{Selic t+3}}{1 + \text{IPCA t+3}} - 1 \right) \times 100
$$

O **juro real** representa o retorno ajustado pela inflação, em outras palavras, indica quanto os juros efetivamente remuneram em termos de poder de compra. Sua mensuração pode ser feita de duas formas: ex-post ou ex-ante. O juro real ex-post utiliza a inflação realizada no período, sendo adequada para análises retrospectivas. Já o juro real ex-ante é uma estimativa prospectiva, construída com base na inflação esperada, e é especialmente útil para avaliar decisões de política monetária no momento em que são tomadas, uma vez que os formuladores de política operam, em grande medida, com base em expectativas futuras, e não apenas em dados já observados.

Neste estudo, será adotado o juro real ex-ante como referência para mensurar a orientação da política monetária. Para isso, será utilizada a taxa swap DI com vencimento em 360 dias como proxy da taxa de juros nominal de mercado, representando o custo de oportunidade em um horizonte anual. Essa taxa será deflacionada pela expectativa suavizada de inflação para os 12 meses seguintes, conforme as projeções do IPCA divulgadas no Boletim Focus. 

$$
\text{Juro Real Ex-Ante} = \left( \frac{1 + \text{Swap}}{1 + \text{IPCA}_{12m}} - 1 \right) \times 100
$$

## Aplicação Prática: Estimando Juros Neutro e Real

A seguir, será realizada a aplicação prática da metodologia de estimativa dos juros neutro e real, utilizando a `linguagem R`, dados do Boletim Focus (Banco Central do Brasil) e da série histórica da taxa de swap DI (IpeaData). O objetivo é construir, de forma reprodutível, uma base que permita inferir se a política monetária está em um regime contracionista ou expansionista, a partir da comparação entre essas duas taxas.

### 1. Carregamento dos Pacotes
```R
library(rbcb)       # Acesso às séries temporais do BCB
library(ipeadatar)  # Acesso às séries temporais do Ipea
library(tsibble)    # Manipulação de séries temporais
library(tidyverse)  # Manipulação e visualização de dados
```

### 2. Coleta das Expectativas do Boletim Focus

O comando abaixo coleta as expectativas anuais para os principais indicadores macroeconômicos dos últimos 10 anos:

```R
dados_focus <- rbcb::get_market_expectations(
  type       = "annual",
  indic      = c("IPCA", "PIB Total", "Selic", "Câmbio"),
  start_date = Sys.Date() - 10*365
)
```

### 3. Tratamento das Expectativas

As expectativas diárias são agregadas para periodicidade mensal com `yearmonth()`, e em seguida calcula-se a média da mediana das projeções:

```R
dados <- dados_focus |>
# Considera apenas a base dos últimos 30 dias
  dplyr::filter(baseCalculo == 0) |>                            
  dplyr::group_by(Indicador, data = yearmonth(Data), DataReferencia) |>
  dplyr::summarise(expectativa_media = mean(Mediana, na.rm = TRUE), .groups = "drop") |>
  dplyr::mutate(
    indicador = Indicador,
    data      = as_date(data),
    data_ref  = as.numeric(DataReferencia),
    .keep     = "none"
  )
```

### 4. Coleta da Taxa Swap e do IPCA Suavizado

A série da taxa swap DI 360 dias representa o juro nominal de mercado, enquanto o IPCA 12m suavizado serve como proxy para a inflação esperada. Ambos são ajustados para frequência mensal:

```R
swaps <- ipeadata("BMF12_SWAPDI36012") |> 
  dplyr::select(data = date, swap = value)

ipca_12m <- rbcb::get_market_expectations(
  type  = "inflation-12-months",
  indic = "IPCA"
) |> 
  filter(Suavizada == "S", baseCalculo == 0) |> 
  dplyr::group_by(data = yearmonth(Data)) |> 
  dplyr::summarise(ipca = mean(Mediana, na.rm = TRUE), .groups = "drop") |> 
  dplyr::mutate(data = as_date(data))
```

### 5. Cálculo do Juro Neutro (t+3)

Utiliza-se a projeção da Selic e do IPCA três anos à frente (t+3) como aproximação para o cálculo da taxa neutra:

```R
juro_neutro <- dados |>
  dplyr::filter(indicador %in% c("Selic", "IPCA"), 
         data_ref == year(data) + 3) |> 
  tidyr::pivot_wider(names_from = indicador, values_from = expectativa_media) |> 
  dplyr::mutate(
    neutro = ((1 + Selic / 100) / (1 + IPCA / 100) - 1) * 100
  )
```

### 6. Cálculo do Juro Real Ex-Ante

Aqui, é estimado o juro real com base no swap nominal deflacionado pela expectativa suavizada de inflação para os próximos 12 meses:

```R
juro_real <- dplyr::left_join(ipca_12m, swaps, by = "data") |>
  dplyr::mutate(
    ex_ante = ((1 + swap / 100) / (1 + ipca / 100) - 1) * 100
  )
```

### 7. Classificação da Política Monetária

Nesta etapa, a política monetária é classificada a cada ponto da série temporal como expansionista ou contracionista, de acordo com a relação entre os juros estimados:

```R
pol_monetaria <- dplyr::left_join(
  x  = juro_neutro, 
  y  = juro_real, 
  by = "data") |>
  dplyr::mutate(
    data = data,
    `Juro Neutro` = neutro,
    `Juro Real`   = ex_ante,
    politica      = ifelse(neutro < ex_ante, "Contracionista", "Expansionista"),
    group = cumsum(c(0, diff(ifelse(neutro < ex_ante, 1, 0)) != 0)),
    .keep         = "none"
  )
```

Abaixo a saída das dez primeiras linhas da tabela.

| Data       | Juro Neutro (%) | Juro Real (%) | Política         | Group |
|------------|------------------|----------------|------------------|--------|
| 2015-06-01 | 5.26             | 7.68           | Contracionista   | 0      |
| 2015-07-01 | 5.26             | 7.77           | Contracionista   | 0      |
| 2015-08-01 | 5.26             | 8.11           | Contracionista   | 0      |
| 2015-09-01 | 5.26             | 8.90           | Contracionista   | 0      |
| 2015-10-01 | 5.04             | 8.48           | Contracionista   | 0      |
| 2015-11-01 | 5.16             | 7.94           | Contracionista   | 0      |
| 2015-12-01 | 5.61             | 8.25           | Contracionista   | 0      |
| 2016-01-01 | 6.04             | 7.89           | Contracionista   | 0      |
| 2016-02-01 | 5.72             | 7.15           | Contracionista   | 0      |
| 2016-03-01 | 5.71             | 6.74           | Contracionista   | 0      |

### 8. Visualização dos Regimes de Política Monetária

O gráfico final compara as duas taxas ao longo do tempo. A área sombreada destaca visualmente os momentos em que a política monetária operou acima ou abaixo da taxa neutra, facilitando a leitura do regime em cada período.

```R
pol_monetaria |>
  ggplot2::ggplot() +
  ggplot2::aes(x = data) +
  ggplot2::geom_hline(yintercept = 0, 
                      size       = 0.75,
                      linetype   = "dashed",
                      colour     = "gray55") +
  ggplot2::geom_ribbon(
    mapping = ggplot2::aes(ymin  = `Juro Neutro`, 
                           ymax  = `Juro Real`, 
                           fill  = politica,
                           group = group), 
    alpha = 0.3
  ) +
  ggplot2::geom_line(
    mapping = ggplot2::aes(y     = `Juro Neutro`,
                           color = "Juro Neutro"),
    size = 1.5
  ) +
  ggplot2::geom_line(
    mapping = ggplot2::aes(y     = `Juro Real`,
                           color = "Juro Real"),
    size = 1.5
  ) +
  ggplot2::scale_color_manual(values = c("darkblue", "darkred")) +
  ggplot2::scale_fill_manual(values = c("darkred", "darkblue"),
                             guide  = "none") +
  ggplot2::scale_y_continuous(n.breaks = 9) +                         
  ggplot2::scale_x_date(breaks      = "1 years",                        
                        date_labels = "%Y") +
  ggplot2::theme_minimal() +
  ggplot2::theme(
    plot.title            = element_text(size = 15,      
                                         face = "bold"),             
    plot.title.position   = "plot",                
    plot.caption          = element_text(hjust = 0,   
                                         face  = "bold"),            
    plot.caption.position = "plot",              
    legend.position       = "bottom",                 
    legend.text           = element_text(face   = "bold"),  
    strip.background      = ggplot2::element_blank(),         
    strip.text            = ggplot2::element_text(size   = 9.5,   
                                                  colour = "black", 
                                                  face   = "bold"),
    axis.title.y          = element_text(size   = 9,       
                                         colour = "black", 
                                         face   = "bold")
  ) +
  ggplot2::labs(
    title    = "Condução da Política Monetária",
    subtitle = "Juro Neutro: Selic esperada t+3 deflacionada pela inflação t+3",
    color    = NULL,
    y        = "% a.a.",
    x        = NULL,
    caption  = "Dados: Sistema Expectativas de Mercado | Banco Central do Brasil (BCB) & Ipeadata"
  )
```

  <img src="/images/pol_monetaria.png" alt="descrição da imagem">

O gráfico final sintetiza toda a lógica construída ao longo da análise. A **área sombreada em vermelho** representa os períodos em que o juro real superou o juro neutro, caracterizando um regime de **política contracionista**. Já a **área sombreada em azul** corresponde aos períodos em que o juro real ficou abaixo do juro neutro, sinalizando um ambiente de **política expansionista**.

A aplicação prática descrita acima demonstra como a união de conceitos econômicos bem fundamentados e ferramentas de análise de dados pode oferecer interpretações robustas sobre o regime da política monetária brasileira. Ao operacionalizar as definições de juro neutro e juro real ex-ante, conseguimos observar como o Banco Central tem atuado nos últimos anos, com fases distintas de estímulo e restrição à atividade econômica.

### Referências

1. A. S. Blinder. Bancos Centrais: teoria e prática. São Paulo: Editora 34, 1999.

2. F. J. C. Carvalho, F. E. P. Souza, J. Sicsu, L. F. R. Paula, and R. Studart. Economia Monetária e Financeira - Teoria e Política. Elsevier, Rio de Janeiro, sétima edition, 2000.
