---
title: mobstr (Mechanical Options Backtesting w/ Tidy R)
author: Jason Taylor
date: '2019-02-03'
slug: mobstr-mechanical-options-backtesting-w-tidy-r
draft: TRUE
categories:
  - Original
tags:
  - AWS
  - backtest study
  - options
  - Original Content
  - RSI
  - Short Put
  - rstats
topics: []
description: ''
editor_options: 
  chunk_output_type: console
---


#### Study:

Traders often look for momentum stocks to get long. Our trading group recently discussed if using common technical analysis metrics like SMA (Simple Moving Average), could be used to find these opportune entries using options?

<!--more-->

* Calculate the historical closing 20 and 50 day SMA
* Identify when the SMA 20 crosses above the SMA 50
* Open a short put closest to 30 delta and 45 DTE and hold to expiration
* Analyze the results and compare to random entry groups
+ calculate the number of trades entered in the period Jan 2012 - Dec 2018 
- sample that same number of trades 1000 times
- compare results of these buckets against SMA entry criteria trades

#### Results:

TBD

#### Setup global options, load libraries:


```r
knitr::opts_chunk$set(message = FALSE, tidy.opts = list(width.cutoff = 60))
suppressWarnings(suppressMessages(suppressPackageStartupMessages({
  library_list <- c("tidyverse", "mobstr", "here", "TTR", "DBI",
                    "formattable", "kableExtra", "knitr", "scales")
  lapply(library_list, require, character.only = TRUE)})))
args <- expand.grid(symbol = c("SPY", "IWM", "GLD", "QQQ", "DIA",
                               "TLT", "XLE", "EEM", "EWZ", "FXE"),
                    tar_dte = 45,
                    tar_delta_put = -0.30,
                    stringsAsFactors = FALSE)
```

#### Summary of study function:

* Connect to the dataset in AWS Athena
* Open short puts every day given the target DTE and Delta
* Calculate the 20 and 50 day SMA and 1 day lags to determine crossovers
* Collect a subset of options data that are possible expiration exits
* Close all trades at expiration and join together results


```r
# athena <- mobstr::athena_connect("default")
# 
# odbc::dbListTables(athena)
# 
# furrr::future_pmap(list(list(athena), "default", args$symbol), mobstr::athena_load)
# 
# odbc::dbListTables(athena)
# args <- args[1, ]
# stock <- args$symbol[1]
# tar_dte <- args$tar_dte[1]
# tar_delta_put <- args$tar_delta_put[1]

study <- function(stock, tar_dte, tar_delta_put) {
  athena <- mobstr::athena_connect("default")
  
  sma <- athena %>% 
    dplyr::tbl(stock) %>%
    dplyr::distinct(symbol, quotedate, close) %>%
    dplyr::collect() %>%
    dplyr::mutate(sma_20 = TTR::SMA(close, n = 20),
                  sma_50 = TTR::SMA(close, n = 50),
                  sma_20_lag = lag(sma_20, n = 1),
                  sma_50_lag = lag(sma_50, n = 1))
  
  opened_puts <- mobstr::open_leg(conn = athena, stock = stock,
                                  put_call = "put", direction = "short", 
                                  tar_delta = tar_delta_put, tar_dte = tar_dte)
  
  sub_options <- athena %>%
    dplyr::tbl(stock) %>%
    dplyr::filter(quotedate == expiration,
                  strike %in% opened_puts$put_strike) %>%
    dplyr::collect()
  
  purrr::pmap_dfr(list(df = list(sub_options), 
                       entry_date = opened_puts$quotedate,
                       exp = opened_puts$expiration,
                       typ = list("put"),
                       stk = opened_puts$put_strike,
                       entry_mid = opened_puts$put_open_short,
                       entry_delta = opened_puts$delta,
                       stk_price = opened_puts$close,
                       direction = list("short")),
                  mobstr::close_leg) %>%
    dplyr::left_join(sma, by = c("symbol", "entry_date" = "quotedate",
                                 "entry_stock_price" = "close"))
}
```

#### Run Study


```r
results <- furrr::future_pmap_dfr(list(args$symbol, args$tar_dte, args$tar_delta_put), study)
```

#### Resample 1000 times  

* Count # of trades for each symbol
* Create arguments list including (symbol, count, 1:1000)
+ SPY had ????? days when the SMA crossed above
+ create 1000 random groups with ????? trades in them


```r
set.seed(42)
grp_args <- results %>%
  group_by(symbol) %>%
  filter(sma_20_lag < sma_50_lag,
         sma_20 > sma_50) %>%
  summarise(count = n()) %>%
  ungroup() %>% 
  slice(rep(row_number(), 1000)) %>%
  group_by(symbol) %>%
  mutate(run = row_number()) %>%
  ungroup()

result_sample <- function(df, s, n, x) {
  df %>%
    filter(symbol == s, 
           complete.cases(.)) %>%
    sample_n(size = n) %>%
    mutate(run = x)
}

resampled <- purrr::pmap_dfr(list(list(results), grp_args$symbol, grp_args$count,
                                  grp_args$run), result_sample)
```

#### Calculate metrics for all runs so we can see the distribution of random entry selections  

* Trade Count
* Win Rate
* Mean Credit
* Mean Profit
* Total Profit


```r
metrics <- resampled %>%
  group_by(symbol, run) %>%
  mutate(trade_count = n(),
         profitable = ifelse(put_profit > 0, 1, 0),
         win_rate = sum(profitable) / trade_count,
         mean_credit = mean(put_entry_mid * 100),
         mean_profit = mean(put_profit * 100),
         total_profit = sum(put_profit * 100)) %>%
  distinct(symbol, trade_count, win_rate, mean_credit, 
           mean_profit, total_profit) %>%
  ungroup() %>%
  arrange(desc(win_rate))
```

#### Boxplot of Win Rates for random samples  


```r
metrics %>%
  select(symbol, win_rate) %>%
  gather(., metric, value, -symbol) %>%
  ggplot(., aes(metric, value, fill = symbol)) +
  geom_boxplot() +
  theme_minimal() +
  theme(plot.title = element_text(hjust = 0.5)) + 
  scale_x_discrete(labels = "Win Rate") +
  scale_y_continuous(labels = scales::percent) +
  xlab("") +
  ylab("Win Rate %") +
  ggtitle("Win rate distributions for each symbol")
```

<img src="2019-02-03-mobstr-mechanical-options-backtesting-w-tidy-r_files/figure-html/unnamed-chunk-3-1.png" width="672" />

#### Density of Win Rates for random samples


```r
metrics %>%
  select(symbol, win_rate) %>%
  gather(., metric, value, -symbol) %>%
  ggplot(., aes(value, fill = symbol)) +
  geom_density() + 
  theme_minimal() +
  theme(plot.title = element_text(hjust = 0.5)) + 
  scale_x_continuous(labels = scales::percent) +
  xlab("Win Rate %") +
  ylab("Count") +
  ggtitle("Win rate distributions for each symbol")
```

<img src="2019-02-03-mobstr-mechanical-options-backtesting-w-tidy-r_files/figure-html/unnamed-chunk-4-1.png" width="672" />

#### Boxplot of Mean Profit at expiration for random samples  


```r
metrics %>%
  select(symbol, mean_profit) %>%
  gather(., metric, value, -symbol) %>%
  ggplot(., aes(metric, value, fill = symbol)) +
  geom_boxplot() +
  theme_minimal() +
  theme(plot.title = element_text(hjust = 0.5)) + 
  scale_x_discrete(labels = "Mean Profit") +
  scale_y_continuous(labels = scales::dollar) +
  xlab("") +
  ylab("Mean Profit") +
  ggtitle("Mean Profit distributions for each symbol")
```

<img src="2019-02-03-mobstr-mechanical-options-backtesting-w-tidy-r_files/figure-html/unnamed-chunk-5-1.png" width="672" />

#### Boxplot of Total Profit at expiration for random samples  


```r
metrics %>%
  select(symbol, total_profit) %>%
  gather(., metric, value, -symbol) %>%
  ggplot(., aes(metric, value, fill = symbol)) +
  geom_boxplot() +
  theme_minimal() +
  theme(plot.title = element_text(hjust = 0.5)) + 
  scale_x_discrete(labels = "Total Profit") +
  scale_y_continuous(labels = scales::dollar) +
  xlab("") +
  ylab("Total Profit") +
  ggtitle("Total Profit distributions for each symbol")
```

<img src="2019-02-03-mobstr-mechanical-options-backtesting-w-tidy-r_files/figure-html/unnamed-chunk-6-1.png" width="672" />

#### Calculate metrics for RSI cross-over entries  

* Trade Count
* Win Rate
* Mean Credit
* Mean Profit
* Total Profit  


```r
sma_results <- results %>%
  group_by(symbol) %>%
    filter(sma_20_lag < sma_50_lag,
         sma_20 > sma_50) %>%
  ungroup()

sma_metrics <- sma_results %>%
  group_by(symbol) %>%
  mutate(trade_count = n(),
         profitable = ifelse(put_profit > 0, 1, 0),
         win_rate = sum(profitable) / trade_count,
         mean_credit = dollar(mean(put_entry_mid * 100)),
         mean_profit = mean(put_profit * 100),
         mean_profit = ifelse(mean_profit < 0, 
                              cell_spec(dollar(mean_profit), 
                                        color = "red", italic = TRUE),
                              dollar(mean_profit)),
         total_profit = sum(put_profit * 100),
         total_profit = ifelse(total_profit < 0, 
                               cell_spec(dollar(total_profit), 
                                         color = "red", italic = TRUE),
                               dollar(total_profit))) %>%
  distinct(symbol, trade_count, win_rate, mean_credit, 
           mean_profit, total_profit) %>%
  ungroup() %>%
  arrange(desc(win_rate)) %>%
  mutate(win_rate = percent(win_rate))
```

#### Metrics table for RSI cross-over entries  







