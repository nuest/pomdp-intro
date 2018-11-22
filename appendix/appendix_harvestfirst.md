POMDP comparisons across sigma\_m, sigma\_g
================
Carl Boettiger
2018-01-26

``` r
# devtools::install_github("boettiger-lab/sarsop")  ## install package first if necessary.
library(sarsop)       # the main POMDP package
library(tidyverse)    # for munging and plotting
library(parallel)
library(gridExtra)
```

``` r
library(ggthemes)
library(ggplot2)
library(Cairo)
library(extrafont)
library(hrbrthemes)

extrafont::loadfonts()
knitr::opts_chunk$set( dev="cairo_pdf")
hrbrthemes::import_roboto_condensed() 
ggplot2::theme_set(hrbrthemes::theme_ipsum_rc())
```

``` r
#palette <- ptol_pal()(6)
palette <- c("#D9661F", "#3B7EA1",  "#6C3302", "#FDB515", "#00B0DA",  "#CFDD45") # berkeley colors

colors <- set_names(c(palette[c(1:4,2,5)], "grey", "black"), 
                    c("TAC", "POMDP", "MSY", "CE", 
                      "POMDP: low prior",
                      "POMDP: high prior",
                      "biomass",
                      "catch"))
```

``` r
options(mc.cores=parallel::detectCores())
log_dir <- "../data/observe_harvest_recruit" # Store the computed solution files here
```

Basic deterministic model
-------------------------

``` r
r <- 0.75
K <- 1

## Unlike classic Graham-Schaefer, this assumes harvest occurs right after assessment, before recruitment
f <- function(x, h){ 
    s <- pmax(x-h, 0) 
    s + s * r * (1 - s / K) 
}
```

Utility (reward) function. (Note that setting a smaller discount will take longer to converge to smooth POMDP solution, but result in `S_star` closer to the simple `B_MSY`.)

``` r
reward_fn <- function(x,h) pmin(x,h)
discount <- 0.95
```

Calculating MSY
---------------

This uses a generic optimization routine to find the stock size at which the maximum growth rate is achieved. (Note that this also depends on the discount rate since future profits are worth proportionally less than current profits).

``` r
## A generic routine to find stock size (x) which maximizes 
## growth rate (f(x,0) - x, where x_t+1 = f(x_t))
S_star <- optimize(function(x) -f(x,0) + x / discount, 
                   c(0, 2*K))$minimum
S_star
```

    [1] 0.4649123

Since we `observe` -&gt; `harvest` -&gt; `recruit`, we would observe the stock at its pre-harvest size, *X*<sub>*t*</sub> ∼ *B*<sub>*M**S**Y*</sub> + *H*<sub>*M**S**Y*</sub>.

``` r
#B_MSY <- S_star     # recruit first, as in classic Graham-Schaefer
B_MSY <- f(S_star,0) # harvest first, we observe the population at B_msy + h

#MSY <- f(B_MSY,0) - B_MSY  # recruit first
MSY <- B_MSY - S_star  # harvest first

F_MSY <- MSY / B_MSY  
F_PGY = 0.8 * F_MSY
```

As a basic reference point, simulate these three policies in a purely deterministic world. Unlike later simulations, here we consider all states an actions exactly (that is, within floating point precision). Later, states and actions are limited to a discrete set, so solutions can depend on resolution and extent of that discretization.

``` r
msy_policy <- function(x) F_MSY * x
pgy_policy <- function(x) F_PGY * x

## ASSUMES harvest takes place *before* recruitment, f(x-h), not after.
escapement_policy <- function(x) pmax(x - S_star,0)  

x0 <- K/6
Tmax = 100
do_det_sim <- function(policy, f, x0, Tmax){
    action <- state <- obs <- as.numeric(rep(NA,Tmax))
    state[1] <- x0

    for(t in 1:(Tmax-1)){
      action[t] <- policy(state[t])
      obs[t] <- state[t] - action[t]  # if we observe after harvest but before recruitment
      state[t+1] <- f(state[t], action[t]) 
    }
    data.frame(time = 1:Tmax, state, action, obs)
  }

det_sims <- 
  list(msy = msy_policy, 
       pgy = pgy_policy,
       det = escapement_policy) %>% 
  map_df(do_det_sim, 
         f, x0, Tmax, 
         .id = "method") 

write_csv(det_sims, "../data/observe_harvest_recruit/det_sims.csv")
```

With no stochasticity, MSY leads to the same long-term stock size as under the constant escapement rule, but takes longer to get there. This level is essentially *B*<sub>*M**S**Y*</sub>, though because the model considered here implements events in the order: *observe, harvest, recruit*; rather than *observe, recruit, harvest*, we see the stock at the pre-harvest size of *B*<sub>*M**S**Y*</sub> + *H*<sub>*M**S**Y*</sub> (that is, *K*/2 + *r**K*/4). More conservative rules, such as a harvest set to 80% of MSY result in faster recovery of the stock than under MSY, but slower than under constant escapement. Due to the reduced maximum harvest rate, such rules lead to stock returning to a value higher than $B\_${MSY}$.

``` r
det_sims %>%
  mutate(method = fct_recode(method, 
                             "CE" = "det",
                             "TAC" = "pgy",
                             "MSY" = "msy")) %>%
  ggplot(aes(time, state, col=method)) + 
  geom_line(lwd=1) + 
  scale_color_manual(values = colors) +
  coord_cartesian(ylim = c(0, 1)) + 
  theme(legend.position = "bottom") + 
  ylab("Mean biomass")
```

![](appendix_harvestfirst_files/figure-markdown_github/unnamed-chunk-7-1.png)

------------------------------------------------------------------------

Introduce a discrete grid
-------------------------

``` r
## Discretize space
states <- seq(0,2, length=100)
actions <- states
observations <- states
```

We compute the above policies on this grid for later comparison.

``` r
index <- function(x, grid) map_int(x, ~ which.min(abs(.x - grid)))


policies <- data.frame(
#  det = index(pmax(f(states,0) - S_star,0), actions), # obs,recruit,harv, f(x_t) - h_t
  det = index(escapement_policy(states),     actions), # obs,harv,recruit, f(x_t - h_t)
  msy = index(msy_policy(states),               actions),
  pgy = index(pgy_policy(states),               actions))
```

POMDP Model
===========

We compute POMDP matrices for a range of `sigma_g` and `sigma_m` values:

``` r
meta <- expand.grid(sigma_g = c(0.02, 0.1, 0.15), 
                    sigma_m = c(0, 0.1, 0.15),
                    stringsAsFactors = FALSE) %>%
        mutate(scenario  = as.character(1:length(sigma_m)))
meta
```

      sigma_g sigma_m scenario
    1    0.02    0.00        1
    2    0.10    0.00        2
    3    0.15    0.00        3
    4    0.02    0.10        4
    5    0.10    0.10        5
    6    0.15    0.10        6
    7    0.02    0.15        7
    8    0.10    0.15        8
    9    0.15    0.15        9

``` r
models <- 
  parallel::mclapply(1:dim(meta)[1], 
           function(i){
  fisheries_matrices(
  states = states,
  actions = actions,
  observed_states = observations,
  reward_fn = reward_fn,
  f = f,
  sigma_g = meta[i,"sigma_g"][[1]],
  sigma_m = meta[i,"sigma_m"][[1]],
  noise = "normal")
})
```

POMDP solution
--------------

The POMDP solution is represented by a collection of alpha-vectors and values, returned in a `*.policyx` file. Each scenario (parameter combination of `sigma_g`, `sigma_m`, and so forth) results in a separate solution file.

Because this solution is computationally somewhat intensive, be sure to have ~ 4 GB RAM per core if running the 9 models in parallel. Alternately, readers can skip the evaluation of this code chunk and read the cached solution from the `policyx` file using the `*_from_log` functions that follow:

``` r
dir.create(log_dir)

## POMDP solution 
## (slow, >10,000 seconds per scenario for discount ~ 0.95;
## needs much longer for discount ~ 0.99)
system.time(
  alphas <- 
    parallel::mclapply(1:length(models), 
    function(i){
      log_data <- data.frame(model = "gs", 
                             r = r, 
                             K = K, 
                             sigma_g = meta[i,"sigma_g"][[1]], 
                             sigma_m = meta[i,"sigma_m"][[1]], 
                             noise = "normal",
                             scenario = meta[i, "scenario"][[1]])
      
      sarsop(models[[i]]$transition,
             models[[i]]$observation,
             models[[i]]$reward,
             discount = discount,
             precision = 0.00000002,
             timeout = 15000,
             log_dir = log_dir,
             log_data = log_data)
    })
)
```

We can read the stored solution from the log:

``` r
meta <- meta_from_log(data.frame(model="gs", discount=discount), log_dir) %>% 
  mutate(scenario = as.character(scenario)) %>%
  left_join(meta) %>% 
  arrange(scenario)
alphas <- alphas_from_log(meta, log_dir)
```

Simulating the static policies under uncertainty
------------------------------------------------

``` r
set.seed(12345)

Tmax <- 100
x0 <- which.min(abs(K/6 - states))
reps <- 100
static_sims <- 
 map_dfr(models, function(m){
            do_sim <- function(policy) sim_pomdp(
                        m$transition, m$observation, m$reward, discount, 
                        x0 = x0, Tmax = Tmax, policy = policy, reps = reps)$df
            map_dfr(policies, do_sim, .id = "method")
          }, .id = "scenario") 
```

Simulating the POMDP policies under uncertainty
-----------------------------------------------

``` r
set.seed(12345)

unif_prior <- rep(1, length(states)) / length(states)
pomdp_sims <- 
  map2_dfr(models, alphas, function(.x, .y){
             sim_pomdp(.x$transition, .x$observation, .x$reward, discount, 
                       unif_prior, x0 = x0, Tmax = Tmax, alpha = .y,
                       reps = reps)$df %>% 
              mutate(method = "pomdp") # include a column labeling method
           },
           .id = "scenario")
```

Combine the resulting data frames

``` r
sims <- bind_rows(static_sims, pomdp_sims) %>%
    left_join(meta) %>%  ## include scenario information (sigmas; etc)
    mutate(state = states[state], action = actions[action]) %>%
    select(time, state, rep, method, sigma_m, sigma_g, value)

sim_col_types <- paste0(gsub("^(\\w).*", "\\1", sapply(unname(sims), class)), collapse = "")
write_csv(sims, file.path(log_dir, "sims.csv"))
```

Figure S2
---------

We the results varying over different noise intensities, sigma\_g, and sigma\_m. Figure 1 of the main text considers the case of sigma\_g = 0.05, sigma\_m = 0.1

``` r
sim_col_types <-  "inicnnn"
sims <- read_csv(file.path(log_dir, "sims.csv"), col_types = sim_col_types)

sims %>%
  select(time, state, rep, method, sigma_m, sigma_g) %>%
  group_by(time, method, sigma_m, sigma_g) %>%
  summarise(mean = mean(state), sd = sd(state)) %>%
  ggplot(aes(time, mean, col=method, fill=method)) + 
  geom_line() + 
  geom_ribbon(aes(ymax = mean + sd, ymin = mean-sd), col = NA, alpha = 0.1) +
  facet_grid(sigma_m ~ sigma_g, 
             labeller = label_bquote(sigma[m] == .(sigma_m),
                                     sigma[g] == .(sigma_g))) + 
  coord_cartesian(ylim = c(0, 1)) 
```

![](appendix_harvestfirst_files/figure-markdown_github/unnamed-chunk-15-1.png)

Figure S3
---------

Economic Value

``` r
sims %>%
  select(time, value, rep, method, sigma_m, sigma_g) %>%
  filter(sigma_g %in% c("0.1", "0.15")) %>%
  group_by(rep, method, sigma_m, sigma_g) %>%
  summarise(npv = sum(value)) %>%  
  group_by(method, sigma_m, sigma_g) %>%
  summarise(net_value = mean(npv), se = sd(npv) / mean(npv)) %>%
  ungroup() %>%
  mutate(method = fct_recode(method,
                             "MSY" = "msy",
                             "CE" = "det",
                             "TAC" = "pgy",
                             "POMDP" = "pomdp")) %>%
  
  
  ggplot(aes(method, net_value, ymin=net_value-se, ymax=net_value+se, fill=method)) + 
  geom_bar(position=position_dodge(), stat="identity") + 
  geom_errorbar(size=.3,width=.2,
                position=position_dodge(.9)) +
  facet_grid(sigma_m~sigma_g, 
             labeller = label_bquote(sigma[m] == .(sigma_m),
                                     sigma[g] == .(sigma_g))) +
  theme(legend.position = "none") + 
  scale_color_manual(values = colors) +
  scale_fill_manual(values = colors) +
  ylab("expected net present value")
```

![](appendix_harvestfirst_files/figure-markdown_github/unnamed-chunk-16-1.png)

------------------------------------------------------------------------

POMDP simulations overestimating measurement uncertainty
--------------------------------------------------------

While the POMDP approach requires an estimate of the measurement error, the precise distribution of measurement errors will itself be unknown in most cases. However, the POMDP approach is quite robust to overestimation of the measurement error. As an extreme example of this, we consider the case where the POMDP solution assumes the largest measurement error level considered here, *σ*<sub>*m*</sub> = 0.15, while performing simulations in which measurements occur without error. In such a scenario, the

``` r
set.seed(12345)
true <- 3 # sigma_g = 0.15, sigma_m = 0
source("../appendix/pomdp_overestimate.R")
pomdp_overest_sims <- 
  map2_dfr(models, alphas, function(.x, .y){
    pomdp_overestimates(transition = .x$transition, 
            model_observation = .x$observation, 
            reward = .x$reward, 
            discount = discount, 
            true_observation = models[[true]]$observation,
            x0 = x0,
            Tmax = Tmax,
            alpha = .y,
            reps = reps)$df %>% 
              mutate(method = "pomdp") # include a column labeling method
           },
           .id = "scenario"
  )
```

Combine the resulting data frames

``` r
overest <-
bind_rows(static_sims, pomdp_overest_sims) %>%
  left_join(meta, by = "scenario") %>%  ## include scenario information (sigmas; etc)
  mutate(state = states[state], action = actions[action]) %>%
  select(time, state, rep, method, sigma_m, sigma_g, value) %>%
  group_by(time, method, sigma_m, sigma_g) %>%
  summarise(mean = mean(state), sd = sd(state)) %>%
  filter(sigma_g == "0.15") %>% 
  ungroup()

overest %>% filter(method != "pomdp", sigma_m == "0")  %>%
  bind_rows(
  overest %>% filter(method == "pomdp", sigma_m == "0.15")) %>%
  select(-sigma_m, -sigma_g) %>%
  mutate(method = fct_recode(method,
                             "MSY" = "msy",
                             "CE" = "det",
                             "TAC" = "pgy",
                             "POMDP" = "pomdp")) %>%
  write_csv(file.path(log_dir, "overest_sims.csv"))
```

``` r
read_csv(file.path(log_dir, "overest_sims.csv")) %>%
  ggplot(aes(time, mean, col=method, fill=method)) + 
  geom_line(lwd = 1) + 
  geom_ribbon(aes(ymax = mean + sd, ymin = mean-sd), col = NA, alpha = 0.1) +
  coord_cartesian(ylim = c(0, 1)) 
```

![](appendix_harvestfirst_files/figure-markdown_github/unnamed-chunk-18-1.png)

Policy plots
------------

``` r
policy_table <- tibble(state = 1:length(states)) %>%
  bind_cols(policies) %>%
  gather(policy, harvest, -state) %>%
  mutate(harvest = actions[harvest], state = states[state]) %>%
  mutate(escapement = state - harvest)
```

Note that when recruitment occurs before harvest, *f*(*x*)−*h*, to get to *X*<sub>*t*</sub> = *B*<sub>*M**S**Y*</sub> as fast as possible, we actually want to harvest a little bit when X is below *B*<sub>*M**S**Y*</sub>, so that rather than over-shooting *B*<sub>*M**S**Y*</sub> (deterministic) recruitment would land us right at it *B*<sub>*M**S**Y*</sub>. This corresponds to constant escapement. The discrete grid makes these appear slightly stepped.

Note that the deterministic solution crosses the MSY solution at an observed value of *B*<sub>*M**S**Y*</sub> (i.e. *K*/2 = 0.5). PGY harvests are always smaller than MSY harvests, but unlike the deterministic optimal solution, PGY and MSY solutions never go to zero.

``` r
policy_table %>% 
  ggplot(aes(state, harvest, col=policy)) + 
  geom_line()  + 
  coord_cartesian(xlim = c(0,K), ylim = c(0, .8))
```

![](appendix_harvestfirst_files/figure-markdown_github/unnamed-chunk-20-1.png)

Note that when recruitment happens before harvest, escapement is *f*(*x*<sub>*t*</sub>)−*h*<sub>*t*</sub>, not *x*<sub>*t*</sub> − *h*<sub>*t*</sub>. The effect of a continuous function map *f* on a discrete grid is also visible as slight wiggles when we plot in terms of escapement instead of harvest (as is common in the optimal control literature in fisheries, e.g. @Sethi2005).

Note that under the optimal solution, escapement is effectively constant at *B*<sub>*M**S**Y*</sub> = 0.5: for all states above a certain size the population is harvested back down to that size. Note that even stocks observed at states slightly below *B*<sub>*M**S**Y*</sub> = 0.5 achieve this target escapement, since we are following the classic Graham Shaeffer formulation here where we observe first, then recruitment happens before harvest, and thus we see the population at smaller size than we harvest it. In classical escapement analysis, observations are usually indexed instead to occur immediately before harvests, and this inflection point occurs right at *B*<sub>*M**S**Y*</sub>.

``` r
policy_table   %>% 
  ggplot(aes(state, escapement, col=policy)) + 
  geom_line() + 
  coord_cartesian(xlim = c(0,K), ylim = c(0, .8))
```

![](appendix_harvestfirst_files/figure-markdown_github/unnamed-chunk-21-1.png)

### Policies under uncertainty

In strategies whose policies are shown in the above plots all ignore both stochasticity and measurement error. If want to compare these to an MDP or POMDP policy, we must specify the level of uncertainty.

In the absence of measurement uncertainty this is straight forward. @Reed1979 essentially tells us that for small growth noise (satisfying or approximately satisfying Reed's self-sustaining condition) that the stochastic optimal policy is equal to the deterministic optimal policy. We can confirm this numerically as follows.

First we grab the transition matrix we have already defined for small `sigma_g`:

``` r
i <- meta %>% filter(sigma_g == 0.02, sigma_m ==0) %>% pull(scenario) %>% as.integer()
m <- models[[i]]
```

With no observation uncertainty, we can solve numerically for the optimal policy with stochastic dynamic programming

``` r
mdp <- MDPtoolbox::mdp_policy_iteration(m$transition, m$reward, discount) 
```

Adding this to the plot we see the result is identical to the deterministic case:

``` r
bind_rows(policy_table,
  data.frame(state = states, 
             policy = "sdp 0.02",
             harvest = actions[mdp$policy],
             stringsAsFactors = FALSE) %>%
  mutate(escapement = state- harvest)) %>% 
  ggplot(aes(state, harvest, col=policy)) + 
  geom_line()  + 
  coord_cartesian(xlim = c(0,K), ylim = c(0, .8))
```

![](appendix_harvestfirst_files/figure-markdown_github/unnamed-chunk-24-1.png)

Repeating this for larger stochasticity, we get a slightly more conservative result:

``` r
i <- meta %>% filter(sigma_g == 0.15, sigma_m ==0) %>% pull(scenario) %>% as.integer()
m <- models[[i]]
mdp <- MDPtoolbox::mdp_policy_iteration(m$transition, m$reward, discount) 
```

``` r
bind_rows(policy_table,
  data.frame(state = states, 
             policy = "sdp 0.15",
             harvest = actions[mdp$policy],
             stringsAsFactors = FALSE)) %>% 
  ggplot(aes(state, harvest, col=policy)) + 
  geom_line()  + 
  coord_cartesian(xlim = c(0,K), ylim = c(0, .8))
```

![](appendix_harvestfirst_files/figure-markdown_github/unnamed-chunk-26-1.png)

### Comparing POMDP Policies

The comparison of POMDP policy is yet more complicated, but the POMDP policy cannot be expressed merely in terms of a target harvest (or escapement) level given an estimation of the stock size (state). The optimal solution for the partially observed system must also reflect all prior observations of the system, not merely the most recent observation, as the system is not Markovian in the observed state variable. We summarize this history as a prior "belief"" about the state, which is updated according to Bayes rule after each observation. (Note that @Sethi2005 fails to realize this and plots solutions with measurement uncertainty without reference to the prior, which explains their counter-intuitive finding that increased uncertainty should result in increased harvest rates).

Let us look at the POMDP solutions under various priors focusing on the case of moderate uncertainty, *σ*<sub>*g*</sub> = *σ*<sub>*m*</sub> = 0.1. (Recall we have already solved the POMDP solution for this model in the simulations above, as defined by the `alpha` vectors, so we can quickly load that solution now.)

``` r
i <- meta %>% filter(sigma_g == 0.15, sigma_m ==0.1) %>% pull(scenario) %>% as.integer()
m <- models[[i]]
alpha <- alphas[[i]] # we need the corresponding alpha vectors  
```

We will consider what the POMDP solution looks like under a few different prior beliefs. A uniform prior sounds like a conservative assumption, but it is not: it puts significantly more weight on improbably large stock values than other priors. (Loading the *α* vectors from our POMDP solution computed earlier, we can then compute a POMDP given these *α*, the matrices for transition, observation, and reward, and the prior we are using)

``` r
unif_prior = rep(1, length(states)) / length(states) # initial belief
unif <- compute_policy(alpha, m$transition, m$observation, m$reward,  unif_prior)
```

For more realistic set of priors, we will consider priors centered at the target *B*<sub>*M**S**Y*</sub> size (or *S*<sup>\*</sup> in the language of Reed), at half *B*<sub>*M**S**Y*</sub>, and at 1.5 times *B*<sub>*M**S**Y*</sub>, each with a standard deviation of *σ*<sub>*m*</sub> = 0.1 (i.e. the uncertainty around a single observation of a stock at that size.)

``` r
i_star <- which.min(abs(states - S_star))
i_low <- which.min(abs(states - 0.25 * K))
i_high <- which.min(abs(states - 0.75 * K))

prior_star <- m$observation[,i_star,1]
prior_low <- m$observation[,i_low,1]
prior_high <- m$observation[,i_high,1] 

star <- compute_policy(alpha, m$transition, m$observation, m$reward,  prior_star)
low <- compute_policy(alpha, m$transition, m$observation, m$reward,  prior_low)
high <- compute_policy(alpha, m$transition, m$observation, m$reward,  prior_high)
```

We gather these solutions into a single data frame and convert from grid indices to continuous values

``` r
df <- unif
df$medium <- star$policy
df$low <- low$policy
df$high <- high$policy

pomdp_policies <- df %>% 
  select(state, uniform = policy, low, medium, high) %>% 
  gather(policy, harvest, -state) %>%
  mutate(state = states[state], 
         harvest = actions[harvest]) %>%
  mutate(escapement = state-harvest) %>%
  select(state, policy, harvest, escapement)

all_policies <- bind_rows(policy_table, pomdp_policies)

write_csv(all_policies, file.path(log_dir, "all_policies.csv"))
```

POMDP policies depend on the prior belief.

``` r
priors <- 
  data_frame(states, 
             prior_low,
             prior_star,
             unif_prior,
             prior_high) %>%
  select(state = states, 
         "POMDP: low prior" = prior_low, 
         "POMDP: high prior" = prior_high) %>%
  gather(policy, prior, -state) %>%
  mutate(prior = prior*10)

policy_df <- all_policies %>% 
  filter(policy %in% c("low", "high", "det", "pgy")) %>%
  mutate(policy = fct_recode(policy,
                             "CE" = "det",
                             "TAC" = "pgy",
                             "POMDP: low prior" = "low",
                             "POMDP: high prior" = "high"
                             )) 


policies_w_priors <- right_join(priors, policy_df, by = c("state", "policy"))
```

    Warning: Column `policy` joining character vector and factor, coercing into
    character vector

``` r
write_csv(policies_w_priors, file.path(log_dir, "policies_w_priors.csv"))
```

``` r
policies_w_priors %>% 
  gather(panel, value, -state, -policy) %>%
  ggplot(aes(state, value, col=policy)) + 
  geom_line(lwd=1)  +
  facet_wrap(~panel, ncol=1, scales = "free_y") +
  scale_color_manual(values = colors) +
  coord_cartesian(xlim = c(0,1), ylim = c(0,.8)) + 
  theme(legend.position = "bottom")
```

    Warning: Removed 200 rows containing missing values (geom_path).

![](appendix_harvestfirst_files/figure-markdown_github/unnamed-chunk-32-1.png)