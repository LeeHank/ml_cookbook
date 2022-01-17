# Tree  

## 理論  

## 實作  

* 這篇文章摘自 Julia Silge 在 2021/7/13 寫的 blog [連結](https://juliasilge.com/blog/scooby-doo/)  
* 主要目的，是用 tree，去預測 `Scooby Doo` 這個美國暢銷影集的某一集，是否會出現 real monster？  


```r
library(tidyverse)
```

### Explore data  


```r
# scooby_raw <- read_csv("https://raw.githubusercontent.com/rfordatascience/tidytuesday/master/data/2021/2021-07-13/scoobydoo.csv")
# saveRDS(scooby_raw, "./data/scooby_raw.rds")
scooby_raw = readRDS("./data/scooby_raw.rds")
scooby_raw
#> # A tibble: 603 × 75
#>    index series_name  network season title  imdb  engagement
#>    <dbl> <chr>        <chr>   <chr>  <chr>  <chr> <chr>     
#>  1     1 Scooby Doo,… CBS     1      What … 8.1   556       
#>  2     2 Scooby Doo,… CBS     1      A Clu… 8.1   479       
#>  3     3 Scooby Doo,… CBS     1      Hassl… 8     455       
#>  4     4 Scooby Doo,… CBS     1      Mine … 7.8   426       
#>  5     5 Scooby Doo,… CBS     1      Decoy… 7.5   391       
#>  6     6 Scooby Doo,… CBS     1      What … 8.4   384       
#>  7     7 Scooby Doo,… CBS     1      Never… 7.6   358       
#>  8     8 Scooby Doo,… CBS     1      Foul … 8.2   358       
#>  9     9 Scooby Doo,… CBS     1      The B… 8.1   371       
#> 10    10 Scooby Doo,… CBS     1      Bedla… 8     346       
#> # … with 593 more rows, and 68 more variables:
#> #   date_aired <date>, run_time <dbl>, format <chr>,
#> #   monster_name <chr>, monster_gender <chr>,
#> #   monster_type <chr>, monster_subtype <chr>,
#> #   monster_species <chr>, monster_real <chr>,
#> #   monster_amount <dbl>, caught_fred <chr>,
#> #   caught_daphnie <chr>, caught_velma <chr>, …
```

* 可以看到，每一列，就是一集。  
* y 是 欄位 `monster_real`，有三種值(TRUE, FALSE, NULL)，意思是，這一集出現的monster們，是真的怪物，還是假的怪物，還是沒有怪物。但我們是想預測出現的怪物是真是假，所以等等要把 NULL 的拔掉    
* x 作者頗偷懶，只用了兩個變數：  
  * `imdb`：這一集的觀眾評分。(裡面有 "NULL" 這個key入的字串，導致欄位類別變成字串，需要把NULL的列拔掉)  
  * `date_aired`： 這一集的播出時間。(作者想把它轉成 10 年一個單位，變成類別變數這樣。)  
* 所以，作者大概是猜測，這兩個變數是預測是否出現怪物的重要因子  
* 先做簡單的前處理：  


```r
scooby = scooby_raw %>%
  mutate(
    # readr 的函數，我把imdb這一欄當number來parse，所以會把非數值的cell補成NA
    imdb = parse_number(imdb),
    # 第一集是1969，所以乾脆算他1970年代比較好處理
    year_aired = 10 * ((lubridate::year(date_aired) + 1) %/% 10),
    # 把 T
    monster_real = case_when(
      monster_real == "FALSE" ~ "fake",
      monster_real == "TRUE" ~ "real"
    ),
    monster_real = factor(monster_real)
  ) %>%
  # 不見得每一集都有 monster，所以只篩有 monster 的集數
  filter(monster_amount > 0, !is.na(imdb)) %>%
  select(monster_real, imdb, year_aired)
scooby
#> # A tibble: 501 × 3
#>    monster_real  imdb year_aired
#>    <fct>        <dbl>      <dbl>
#>  1 fake           8.1       1970
#>  2 fake           8.1       1970
#>  3 fake           8         1970
#>  4 fake           7.8       1970
#>  5 fake           7.5       1970
#>  6 fake           8.4       1970
#>  7 fake           7.6       1970
#>  8 fake           8.2       1970
#>  9 fake           8.1       1970
#> 10 fake           8         1970
#> # … with 491 more rows
```


```r
dim(scooby)
#> [1] 501   3
```


* 看一下 y 的分佈比例：  


```r
scooby %>%
  count(monster_real) %>%
  mutate(p = n/sum(n))
#> # A tibble: 2 × 3
#>   monster_real     n     p
#>   <fct>        <int> <dbl>
#> 1 fake           389 0.776
#> 2 real           112 0.224
```

* 可以發現，出現的怪獸，幾乎都不是 real monster。
* 而且如果我全部都猜 FALSE 的話，我的 accuracy 也有 78%。這是等等的比較基準。  
* 接著看一下 x 對 y 的影響：  


```r
scooby %>%
  group_by(year_aired) %>%
  summarise(real_p = sum(monster_real == "real")/n()) %>%
  ungroup() %>%
  ggplot(aes(x = year_aired, y = real_p)) +
  geom_bar(stat = "identity", fill = "steelblue") +
  theme_bw()
```

<img src="ch2_3_tree_files/figure-html/unnamed-chunk-6-1.png" width="672" />

* 不同年份，果然有差，很明顯在 1980 和 2000 的真怪獸比例超過 40%，其他年份出現的怪獸幾乎都是假怪獸  
* 再來看看 imdb 的影響  


```r
scooby %>%
  ggplot(aes(imdb, after_stat(density), fill = monster_real)) +
  geom_histogram(position = "identity", alpha = 0.5) +
  labs(x = "IMDB rating", y = "Density", fill = "Real monster?")
```

<img src="ch2_3_tree_files/figure-html/unnamed-chunk-7-1.png" width="672" />

* 可以看到，IMDB rating 如果小於 7 分，幾乎都是真怪獸; IMDB > 7 分，那一集幾乎都出現假怪獸  
* 所以，這兩個變數看來真的有點用處。  

### Train a model  

#### split data  


```r
library(rsample)
set.seed(1234)
scooby_split <- initial_split(scooby, strata = monster_real)
scooby_train <- training(scooby_split)
scooby_test <- testing(scooby_split)

# 資料太少，不用 cv ，改用 bootstrap
set.seed(1234)
scooby_folds <- bootstraps(scooby_train, strata = monster_real)
scooby_folds
#> # Bootstrap sampling using stratification 
#> # A tibble: 25 × 2
#>    splits            id         
#>    <list>            <chr>      
#>  1 <split [375/139]> Bootstrap01
#>  2 <split [375/144]> Bootstrap02
#>  3 <split [375/142]> Bootstrap03
#>  4 <split [375/136]> Bootstrap04
#>  5 <split [375/138]> Bootstrap05
#>  6 <split [375/124]> Bootstrap06
#>  7 <split [375/138]> Bootstrap07
#>  8 <split [375/138]> Bootstrap08
#>  9 <split [375/136]> Bootstrap09
#> 10 <split [375/134]> Bootstrap10
#> # … with 15 more rows
```

#### preprocessing  

* 做 tree 的好處，就是不太需要 preprocessing，送啦！  


```r
library(recipes)
tree_rec <- recipe(monster_real ~ ., data = scooby_train)
tree_rec
#> Recipe
#> 
#> Inputs:
#> 
#>       role #variables
#>    outcome          1
#>  predictor          2
```
 
#### specify model  

* tree 的 三個 tuning parameter 我都要自己 tune  


```r
library(parsnip)
library(tune)
tree_spec <-
  decision_tree(
    cost_complexity = tune(),
    tree_depth = tune(),
    min_n = tune()
  ) %>%
  set_mode("classification") %>%
  set_engine("rpart")
```


#### workflow setting  


```r
library(workflows)
tree_wf = workflow() %>%
  add_recipe(tree_rec) %>%
  add_model(tree_spec)

tree_wf
#> ══ Workflow ════════════════════════════════════════════════
#> Preprocessor: Recipe
#> Model: decision_tree()
#> 
#> ── Preprocessor ────────────────────────────────────────────
#> 0 Recipe Steps
#> 
#> ── Model ───────────────────────────────────────────────────
#> Decision Tree Model Specification (classification)
#> 
#> Main Arguments:
#>   cost_complexity = tune()
#>   tree_depth = tune()
#>   min_n = tune()
#> 
#> Computational engine: rpart
```

#### hyper parameter setting  


```r
library(dials)
hyper_param_meta = tree_spec %>%
  parameters() %>%
  finalize(scooby_train)

hyper_param_meta
#> Collection of 3 parameters for tuning
#> 
#>       identifier            type    object
#>  cost_complexity cost_complexity nparam[+]
#>       tree_depth      tree_depth nparam[+]
#>            min_n           min_n nparam[+]
```

* 從 hyper parameter 的 meta data table，可看出：  
  * identifier: 每個超參數的 id  
  * object: nparam[+] 的意思是，他是 numeric parameter，`+` 表示已有設定 range 在裡面  
* 我們可以這樣看他的範圍


```r
hyper_param_meta %>%
  pull_dials_object("cost_complexity")
#> Cost-Complexity Parameter (quantitative)
#> Transformer:  log-10 
#> Range (transformed scale): [-10, -1]
```

```r
hyper_param_meta %>%
  pull_dials_object("tree_depth")
#> Tree Depth (quantitative)
#> Range: [1, 15]
```


```r
hyper_param_meta %>%
  pull_dials_object("min_n")
#> Minimal Node Size (quantitative)
#> Range: [2, 40]
```

* 我想用 latin_hypercube，幫他在 feature space 中均勻撒 50 個點  


```r
set.seed(17)
my_grid = grid_latin_hypercube(hyper_param_meta, size = 50)
my_grid
#> # A tibble: 50 × 3
#>    cost_complexity tree_depth min_n
#>              <dbl>      <int> <int>
#>  1   0.00000000301          9     5
#>  2   0.0906                 2    18
#>  3   0.00000431             6     8
#>  4   0.00340               14     7
#>  5   0.000116               2     4
#>  6   0.0000190             11    14
#>  7   0.0496                 3    16
#>  8   0.0000000422          10    16
#>  9   0.0000000215          12    13
#> 10   0.000277              12    19
#> # … with 40 more rows
```

* 看一下這 50 個點撒得怎麼樣：  


```r
my_grid %>%
  # 因為 cost_complexity 是在 log scale 上均勻撒點
  # 所以為了畫圖，先讓他回到 log scale
  mutate(cost_complexity = log(cost_complexity)) %>%
  ggplot(aes(x = .panel_x, y = .panel_y)) + 
  geom_point() +
  geom_blank() +
  ggforce::facet_matrix(vars(cost_complexity, tree_depth, min_n), layer.diag = 2) + 
  labs(title = "Latin Hypercube design with 50 candidates")
```

<img src="ch2_3_tree_files/figure-html/unnamed-chunk-17-1.png" width="672" />

* nice~ 果然撒得很均勻～  

#### model fitting  

##### tune hyper parameter  


```r
# 開平行運算
library(doParallel)
cl <- makePSOCKcluster(4) # Create a cluster object
registerDoParallel(cl) # register

library(yardstick)
# fitting
set.seed(130)
tree_tune <- 
  tune::tune_grid(
    object = tree_wf,
    resamples = scooby_folds,
    metrics = metric_set(accuracy, roc_auc, sensitivity, specificity),
    grid = my_grid,
    control = control_resamples(save_pred = TRUE, save_workflow = TRUE)
)

# 關平行運算
stopCluster(cl)
```

* 看一下 tuning 過程：  


```r
autoplot(tree_tune, type = "marginals")
```

<img src="ch2_3_tree_files/figure-html/unnamed-chunk-19-1.png" width="672" />


```r
show_best(tree_tune)
#> # A tibble: 5 × 9
#>   cost_complexity tree_depth min_n .metric  .estimator  mean
#>             <dbl>      <int> <int> <chr>    <chr>      <dbl>
#> 1   0.00719                8    31 accuracy binary     0.836
#> 2   0.0190                14    40 accuracy binary     0.835
#> 3   0.00000000627          6    39 accuracy binary     0.835
#> 4   0.0000000626           5    31 accuracy binary     0.835
#> 5   0.000000167           14    30 accuracy binary     0.835
#> # … with 3 more variables: n <int>, std_err <dbl>,
#> #   .config <chr>
```












