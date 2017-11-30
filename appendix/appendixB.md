
Section 1: Model estimates from data
====================================

Computes model parameter estimates on each `stockid` in RAM (after normalizing data) using nimble. Then, estimate common stock measurement (Bmsy, MSY, PGY).

``` r
# devtools::install_github("boettiger-lab/sarsop")  ## install package first if necessary.
library(tidyverse)
library(sarsop)
library(nimble)
library(parallel)
```

``` r
download.file("https://depts.washington.edu/ramlegac/wordpress/databaseVersions/RLSADB_v3.0_(assessment_data_only)_excel.zip",
              "ramlegacy.zip")
```

``` r
path <- unzip("ramlegacy.zip")
sheets <- readxl::excel_sheets(path)
ram <- lapply(sheets, readxl::read_excel, path = path)
names(ram) <- sheets
unlink("ramlegacy.zip")
unlink("RLSADB v3.0 (assessment data only).xlsx")
ramlegacy <- 
  ram$timeseries_values_views %>%
  select(assessid, stockid, stocklong, year, SSB, TC) %>%
  left_join(ram$stock) %>%
  left_join(ram$area) %>%
  select(assessid, stockid, scientificname, 
         commonname, areaname, country, year, 
         SSB, TC) %>%
  left_join(ram$timeseries_units_views %>%
              rename(TC_units = TC, SSB_units = SSB)) %>%
  select(scientificname, commonname, 
         stockid, areaname, country, year, 
         SSB, TC, SSB_units, TC_units)
```

Let's filter out missing data, non-matching units, and obvious reporting errors (catch exceeding total spawning biomass), then we rescale each series into the 0,1 by appropriate choice of units:

``` r
df2 <- ramlegacy %>% 
  filter(!is.na(SSB), !is.na(TC)) %>%
  filter(SSB_units == "MT", TC_units=="MT") %>% 
  filter(SSB > TC) %>%
  select(-SSB_units, -TC_units) %>% 
  group_by(stockid) %>%
  mutate(scaled_catch = TC / max(SSB),
         scaled_biomass = SSB / max(SSB)) 
```

``` r
stock_ids <- c("PLAICNS", "ARGHAKENARG")
examples <- df2 %>% 
  filter(stockid %in% stock_ids) %>% 
  ungroup() %>% 
  group_by(commonname)
```

``` r
## Model does not estimate sigma_m; data is insufficient to do so.
## Note that RAM stock estimates are not always direct measurements, can be artifically smooth, underestimating sigma_g

gs_code  <- nimble::nimbleCode({
    
  r ~ dunif(0, 2)
  K ~ dunif(0, 2)
  sigma ~ dunif(0, 1)
  
  x[1] <- x0
  for(t in 1:(N-1)){
    mu[t] <- x[t] + x[t] * r * (1 - x[t] / K) - min(a[t],x[t])
    x[t+1] ~ dnorm(mu[t], sd = sigma)
  }

  
})

fit_models <- function(fish, code){
  # fish <- examples %>% filter(stockid == stock_ids[1])
  
  ## Rescale data
  N <- dim(fish)[1]
  scaled_data <- data.frame(t = 1:N, y = fish$scaled_biomass, a = fish$scaled_catch)
  data = data.frame(x = scaled_data$y)

  ## Compile  model
  constants <- list(N = N, a = scaled_data$a)
  inits <- list(r = 0.5, K = 0.5, sigma = 0.02, x0 = scaled_data$y[1])
  model <- nimbleModel(code, constants, data, inits)
  C_model <- compileNimble(model)
  
  mcmcspec <- configureMCMC(model, thin = 1e2)
  mcmc <- buildMCMC(mcmcspec)
  Cmcmc <- compileNimble(mcmc, project = model)
  Cmcmc$run(1e6)
  
  
  samples <- as.data.frame(as.matrix(Cmcmc$mvSamples))
  burnin <- 1:(0.05 * dim(samples)[1]) # drop first 5%
  samples <- samples[-burnin,1:(length(inits) - 1)] # drop raised vars, burnin
  
  #gather(samples) %>% ggplot() + geom_density(aes(value)) + facet_wrap(~key, scale='free')
   
  ## Return fit
  data.frame(stockid = fish$stockid[1],
             commonname = fish$commonname[1],
             r = mean(samples$r),
             K = mean(samples$K),
             sigma_g = mean(samples$sigma),
             r_sd = sd(samples$r),
             K_sd = sd(samples$K),
             sigma_g_sd = sd(samples$sigma),
             stringsAsFactors = FALSE)
  
}
```

``` r
set.seed(123)
fits <- examples %>% do(fit_models(., code=gs_code))
```

    |-------------|-------------|-------------|-------------|
    |-------------------------------------------------------|
    |-------------|-------------|-------------|-------------|
    |-------------------------------------------------------|

``` r
fits 
```

    # A tibble: 2 x 8
    # Groups:   commonname [2]
          stockid      commonname         r        K   sigma_g       r_sd
            <chr>           <chr>     <dbl>    <dbl>     <dbl>      <dbl>
    1 ARGHAKENARG  Argentine hake 1.0387783 1.196112 0.1118370 0.17650455
    2     PLAICNS European Plaice 0.9055933 1.778186 0.1275455 0.07339706
    # ... with 2 more variables: K_sd <dbl>, sigma_g_sd <dbl>

<!--
Do the estimates make sense?  Consider simulation from models, under this historical harvests, compared to historical trajectories


```r
examples %>% ungroup() %>%
  select(year, scaled_biomass, scaled_catch, commonname) %>%
  gather(stock, biomass, -year, -commonname) %>%
  ggplot(aes(year, biomass, col=stock)) + 
  geom_line() + facet_wrap(~commonname, scales = "free")
```

![](appendixB_files/figure-markdown_github-ascii_identifiers/unnamed-chunk-6-1.png)


-->
``` r
pars <- fits %>% ungroup() %>% select(commonname, r, K, sigma_g) 
pars
```

    # A tibble: 2 x 4
           commonname         r        K   sigma_g
                <chr>     <dbl>    <dbl>     <dbl>
    1  Argentine hake 1.0387783 1.196112 0.1118370
    2 European Plaice 0.9055933 1.778186 0.1275455

------------------------------------------------------------------------

Decision Policies
=================

``` r
options(mc.cores = 4) # Reserve ~ 10 GB per core
log_dir <- "pomdp_historical"
```

``` r
## Classic Graham-Schaefer. Note that recruitment occurs *before* harvest
gs <- function(r,K){
  function(x, h){ 
    x + x * r * (1 - x / K) - pmin(x,h)
  }
}
reward_fn <- function(x,h) pmin(x,h)
discount <- .99
```

Discretize space
----------------

Note that the large values of *K* require we carry the numerical grid out further.

``` r
states <- seq(0,4, length=150)
actions <- states
observations <- states
```

All parameter values combinations for which we want solutions
-------------------------------------------------------------

``` r
meta <- expand.grid(commonname = pars$commonname, 
                    sigma_m = c(0, 0.1, 0.2),
                    stringsAsFactors = FALSE) %>%
  left_join(pars) %>%
  mutate(scenario  = as.character(1:length(sigma_m)))

meta
```

           commonname sigma_m         r        K   sigma_g scenario
    1  Argentine hake     0.0 1.0387783 1.196112 0.1118370        1
    2 European Plaice     0.0 0.9055933 1.778186 0.1275455        2
    3  Argentine hake     0.1 1.0387783 1.196112 0.1118370        3
    4 European Plaice     0.1 0.9055933 1.778186 0.1275455        4
    5  Argentine hake     0.2 1.0387783 1.196112 0.1118370        5
    6 European Plaice     0.2 0.9055933 1.778186 0.1275455        6

Create the models
-----------------

``` r
models <- 
  parallel::mclapply(1:dim(meta)[1], 
                     function(i){
                       fisheries_matrices(
                         states = states,
                         actions = actions,
                         observed_states = observations,
                         reward_fn = reward_fn,
                         f = gs(meta[i, "r"][[1]], meta[i, "K"][[1]]),
                         sigma_g = meta[i,"sigma_g"][[1]],
                         sigma_m = meta[i,"sigma_m"][[1]],
                         noise = "normal")
                     })
```

``` r
dir.create(log_dir)

## POMDP solution (slow, >20,000 seconds per loop memory intensive)
system.time(
  alphas <- 
    parallel::mclapply(1:length(models), 
    function(i){
      log_data <- data.frame(model = "gs", 
                             r = meta[i, "r"][[1]], 
                             K = meta[i, "K"][[1]], 
                             sigma_g = meta[i,"sigma_g"][[1]], 
                             sigma_m = meta[i,"sigma_m"][[1]], 
                             noise = "normal",
                             commonname = meta[i, "commonname"][[1]],
                             scenario = meta[i, "scenario"][[1]])
      
      sarsop(models[[i]]$transition,
             models[[i]]$observation,
             models[[i]]$reward,
             discount = discount,
             precision = 2e-6,
             timeout = 20000,
             log_dir = log_dir,
             log_data = log_data)
    })
)
```

``` r
meta <- meta_from_log(data.frame(model="gs"), log_dir)  %>% 
  left_join(
    ## use the estimated sigma_m, r, K pars recorded in the logged meta.csv.  Technically
    ## these should be the same as the above, but without random seed re-running the nimble
    ## code but not re-running these can create mis-matches.  
    select(meta, sigma_m, commonname, scenario),   
    by = c("sigma_m", "commonname")) %>% 
  arrange(scenario)
```

    Warning: Column `commonname` joining factor and character vector, coercing
    into character vector

``` r
alphas <- alphas_from_log(meta, log_dir)
#models <- models_from_log(meta)
```

Comparison to the static models
-------------------------------

``` r
pars <- examples %>% 
  group_by(commonname) %>% 
  summarise(N = max(SSB)) %>% 
  right_join(
    meta %>% 
      select(commonname, r, K) %>% 
      distinct())
```

Add corresponding static policy levels on:

``` r
statics <- function(P){
  f <- gs(P$r, P$K)
  S_star <- optimize(function(x) -f(x,0) + x / discount, c(0, 2* P$K))$minimum
  B_MSY <- S_star
  MSY <- f(B_MSY,0) - B_MSY
  
  tibble(S_star, F_MSY = MSY / B_MSY, F_PGY = 0.8 * F_MSY, 
         commonname = P$commonname, N = P$N)
}

policy_pars <- 
  pars %>% 
  rowwise() %>% 
  do(statics(.))
```

Convert example data into discrete index space.

``` r
index <- function(x, grid) map_int(x, ~ which.min(abs(.x - grid)))
## repeats each series for each static model
ex <- examples %>%
  mutate(biomass = index(scaled_biomass, states),          
         catch = index(scaled_catch, actions))  %>% 
  left_join(policy_pars) %>% left_join(pars) %>%
  ungroup() 
ex
```

    # A tibble: 69 x 18
          scientificname     commonname     stockid           areaname
                   <chr>          <chr>       <chr>              <chr>
     1 Merluccius hubbsi Argentine hake ARGHAKENARG Northern Argentina
     2 Merluccius hubbsi Argentine hake ARGHAKENARG Northern Argentina
     3 Merluccius hubbsi Argentine hake ARGHAKENARG Northern Argentina
     4 Merluccius hubbsi Argentine hake ARGHAKENARG Northern Argentina
     5 Merluccius hubbsi Argentine hake ARGHAKENARG Northern Argentina
     6 Merluccius hubbsi Argentine hake ARGHAKENARG Northern Argentina
     7 Merluccius hubbsi Argentine hake ARGHAKENARG Northern Argentina
     8 Merluccius hubbsi Argentine hake ARGHAKENARG Northern Argentina
     9 Merluccius hubbsi Argentine hake ARGHAKENARG Northern Argentina
    10 Merluccius hubbsi Argentine hake ARGHAKENARG Northern Argentina
    # ... with 59 more rows, and 14 more variables: country <chr>, year <dbl>,
    #   SSB <dbl>, TC <dbl>, scaled_catch <dbl>, scaled_biomass <dbl>,
    #   biomass <int>, catch <int>, S_star <dbl>, F_MSY <dbl>, F_PGY <dbl>,
    #   N <dbl>, r <dbl>, K <dbl>

``` r
det_f <- function(S_star, r, K, i) index(pmax(gs(r[[1]],K[[1]])(states,0) - S_star[[1]],0), actions)[i]
msy_f <- function(F_MSY, i) index(states * F_MSY[[1]], actions)[i]
pgy_f <- function(F_PGY, i) index(states * F_PGY[[1]], actions)[i]

rescale <- function(x, N) states[x]*N

historical <- ex %>% 
  group_by(commonname) %>% 
  mutate(det =  det_f(S_star, r, K, biomass),
         msy =  msy_f(F_MSY, biomass),
         pgy =  pgy_f(F_PGY, biomass)) %>%
  select(year, biomass, catch, det, msy, pgy, commonname, N) %>%
  gather(model, stock, -year, -commonname, -N) %>%
  mutate(stock = states[stock] * N) %>% 
  select(-N)
```

Plots
-----

``` r
historical %>%
  ggplot(aes(year, stock, col=model)) +
  geom_line(lwd=1) +
  scale_color_manual(values = c("black", "grey", "#D9661F", "#00B0DA", "#CFDD45")) +
  facet_wrap(~commonname, scales = "free", ncol=1) + theme_bw()  
```

![](appendixB_files/figure-markdown_github-ascii_identifiers/unnamed-chunk-19-1.png)

POMDP historical
----------------

``` r
pomdp_sims <- 
  pmap_dfr(list(models, alphas, 1:dim(meta)[[1]]), 
           function(.x, .y, .z){
             
             ## avoid NSE
             who <- (ex$commonname == meta[.z,"commonname"]) 
             df <- ex[who,]
             
             hindcast_pomdp(.x$transition, .x$observation, .x$reward, discount, 
                            obs = index(df$scaled_biomass, states), 
                            action = index(df$scaled_catch,states),
                            alpha = .y)$df %>% 
              mutate(method = "pomdp") %>% # include a column labeling method
              mutate(year = ex[who, "year"][[1]]) 
           },
           .id = "scenario") 
```

``` r
pomdp_sims <- 
  meta %>% 
  select(scenario, commonname,sigma_m) %>% 
  left_join(pars) %>%
  right_join(pomdp_sims)
```

``` r
sims <- pomdp_sims %>% 
  mutate(optimal = states[optimal] * N) %>% # original scale
  select(year, optimal, commonname, sigma_m) %>%
  rename(stock = optimal) %>% 
  ## treat each sigma_m value as separate 'model'
  mutate(sigma_m = as.factor(sigma_m)) %>%
  mutate(model = recode(sigma_m, 
                        "0" = "reed",
                        "0.1" = "pomdp_0.1", 
                        "0.2" = "pomdp_0.2")) %>%
  select(-sigma_m) %>%
  bind_rows(historical)
```

    Warning in bind_rows_(x, .id): binding factor and character vector,
    coercing into character vector

    Warning in bind_rows_(x, .id): binding character and factor vector,
    coercing into character vector

``` r
sims %>%
  filter(model %in% c("biomass", "catch", "pomdp_0.1", "reed", "pgy")) %>%
  ggplot(aes(year, stock, col=model)) +
  geom_line(lwd=1) +
  scale_color_manual(values = c("black", "grey", "#D9661F", "#3B7EA1", "#FDB515", "#6C3302",  "#00B0DA",  "#CFDD45")) +
  facet_wrap(~commonname, scales = "free", ncol=1) + theme_bw()  
```

    Warning: Removed 2 rows containing missing values (geom_path).

![](appendixB_files/figure-markdown_github-ascii_identifiers/unnamed-chunk-23-1.png)