---
title: "Bayesian Regression in Blavaan (using Jags)"
author: "By [Laurent Smeets](https://www.rensvandeschoot.com/colleagues/laurent-smeets/) and [Rens van de Schoot](https://www.rensvandeschoot.com/about-rens/)"
date: 'Last modified: 21 August 2019'
output:
  html_document:
    keep_md: true
---







This tutorial provides the reader with a basic tutorial how to perform a **Bayesian regression** in [Blavaan](https://faculty.missouri.edu/~merklee/blavaan/). Throughout this tutorial, the reader will be guided through importing data files, exploring summary statistics and regression analyses. Here, we will exclusively focus on [Bayesian](https://www.rensvandeschoot.com/a-gentle-introduction-to-bayesian-analysis-applications-to-developmental-research/) statistics. 

In this tutorial, we start by using the default prior settings of the software.  In a second step, we will apply user-specified priors, and if you really want to use Bayes for your own data, we recommend to follow the [WAMBS-checklist](https://www.rensvandeschoot.com/wambs-checklist/), also available in other software.

## Preparation

This tutorial expects:

- Any installed version of [JAGS](https://sourceforge.net/projects/mcmc-jags/files/latest/download?source=files)
- Installation of R packages `rjags`, `lavaan` and `blavaan`. This tutorial was made using Blavaan version 0.3.6 and Lavaan version 0.6.4 in R version 3.6.1
- Basic knowledge of hypothesis testing
- Basic knowledge of correlation and regression
- Basic knowledge of [Bayesian](https://www.rensvandeschoot.com/a-gentle-introduction-to-bayesian-analysis-applications-to-developmental-research/) inference
- Basic knowledge of coding in R

## Example Data

The data we will be using for this exercise is based on a study about predicting PhD-delays ([Van de Schoot, Yerkes, Mouw and Sonneveld 2013](http://journals.plos.org/plosone/article?id=10.1371/journal.pone.0068839)).The data can be downloaded [here](https://www.rensvandeschoot.com/wp-content/uploads/2018/10/phd-delays.csv). Among many other questions, the researchers asked the Ph.D. recipients how long it took them to finish their Ph.D. thesis (n=333). It appeared that Ph.D. recipients took an average of 59.8 months (five years and four months) to complete their Ph.D. trajectory. The variable B3_difference_extra measures the difference between planned and actual project time in months (mean=9.97, minimum=-31, maximum=91, sd=14.43). For more information on the sample, instruments, methodology and research context we refer the interested reader to the paper.

For the current exercise we are interested in the question whether age (M = 31.7, SD = 6.86) of the Ph.D. recipients is related to a delay in their project.

The relation between completion time and age is expected to be non-linear. This might be due to that at a certain point in your life (i.e., mid thirties), family life takes up more of your time than when you are in your twenties or when you are older.

So, in our model the $gap$ (*B3_difference_extra*) is the dependent variable and $age$ (*E22_Age*) and $age^2$(*E22_Age_Squared *) are the predictors. The data can be found in the file <span style="color:red"> ` phd-delays.csv` </span>.

  <p>&nbsp;</p>


##### _**Question:** Write down the null and alternative hypotheses that represent this question. Which hypothesis do you deem more likely?_

[expand title="Answer" trigclass="noarrow my_button" targclass="my_content" tag="button"]

$H_0:$ _$age$ is not related to a delay in the PhD projects._

$H_1:$ _$age$ is related to a delay in the PhD projects._ 

$H_0:$ _$age^2$ is not related to a delay in the PhD projects._

$H_1:$ _$age^2$ is related to a delay in the PhD projects._ 

[/expand]


  <p>&nbsp;</p>

## Preparation - Importing and Exploring Data



```r
# if you dont have these packages installed yet, please use the install.packages("package_name") command.
library(rjags) 
library(blavaan)
library(psych) #to get some extended summary statistics
library(tidyverse) # needed for data manipulation and plotting
```

You can find the data in the file <span style="color:red"> ` phd-delays.csv` </span>, which contains all variables that you need for this analysis. Although it is a .csv-file, you can directly load it into R using the following syntax:

```r
#read in data
dataPHD <- read.csv2(file="phd-delays.csv")
colnames(dataPHD) <- c("diff", "child", "sex","age","age2")
```


Alternatively, you can directly download them from GitHub into your R work space using the following command:

```r
dataPHD <- read.csv2(file="https://raw.githubusercontent.com/LaurentSmeets/Tutorials/master/Blavaan/phd-delays.csv")
colnames(dataPHD) <- c("diff", "child", "sex","age","age2")
```

GitHub is a platform that allows researchers and developers to share code, software and research and to collaborate on projects (see https://github.com/)

Once you loaded in your data, it is advisable to check whether your data import worked well. Therefore, first have a look at the summary statistics of your data. you can do this by using the  `describe()` function.

##### _**Question:** Have all your data been loaded in correctly? That is, do all data points substantively make sense? If you are unsure, go back to the .csv-file to inspect the raw data._


[expand title="Answer" trigclass="noarrow my_button" targclass="my_content" tag="button"]


```r
describe(dataPHD)
```

```
##       vars   n    mean     sd median trimmed    mad min  max range  skew
## diff     1 333    9.97  14.43      5    6.91   7.41 -31   91   122  2.21
## child    2 333    0.18   0.38      0    0.10   0.00   0    1     1  1.66
## sex      3 333    0.52   0.50      1    0.52   0.00   0    1     1 -0.08
## age      4 333   31.68   6.86     30   30.39   2.97  26   80    54  4.45
## age2     5 333 1050.22 656.39    900  928.29 171.98 676 6400  5724  6.03
##       kurtosis    se
## diff      5.92  0.79
## child     0.75  0.02
## sex      -2.00  0.03
## age      24.99  0.38
## age2     42.21 35.97
```

_The descriptive statistics make sense:_

_diff: Mean (9.97), SE (0.79)_

_age: Mean (31.68), SE (0.38)_

_age2: Mean (1050.22), SE (35.97)_

[/expand]

  <p>&nbsp;</p>

## plot

Before we continue with analyzing the data we can also plot the expected relationship.


```r
dataPHD %>%
  ggplot(aes(x = age,
             y = diff)) +
  geom_point(position = "jitter",
             alpha    = .6)+ #to add some random noise for plotting purposes
  theme_minimal()+
  geom_smooth(method = "lm",  # to add  the linear relationship
              aes(color = "linear"),
              se = FALSE) +
  geom_smooth(method = "lm",
              formula = y ~ x + I(x^2),# to add  the quadratic relationship
              aes(color = "quadratic"),
              se = FALSE) +
  labs(title    = "Delay vs. age",
       subtitle = "There seems to be some quadratic relationship",
       x        = "Age",
       y        = "Delay",
       color    = "Type of relationship" ) +
  theme(legend.position = "bottom")
```

![](Blavaan-Jags-Regression_files/figure-html/unnamed-chunk-5-1.png)<!-- -->


## Regression - Default Priors

In this exercise you will investigate the impact of Ph.D. students&#39; $age$ and $age^2$ on the delay in their project time, which serves as the outcome variable using a regression analysis (note that we ignore assumption checking!).

As you know, Bayesian inference consists of combining a prior distribution with the likelihood obtained from the data. Specifying a prior distribution is one of the most crucial points in Bayesian inference and should be treated with your highest attention (for a quick refresher see e.g.  [Van de Schoot et al. 2017](http://onlinelibrary.wiley.com/doi/10.1111/cdev.12169/abstract)). In this tutorial, we will first rely on the default prior settings, thereby behaving a &#39;naive&#39; Bayesians (which might [not](https://www.rensvandeschoot.com/analyzing-small-data-sets-using-bayesian-estimation-the-case-of-posttraumatic-stress-symptoms-following-mechanical-ventilation-in-burn-survivors/)always be a good idea).

To run a multiple regression with Blavaan, you first specify the model, then fit the model and finally acquire the summary (similar to the frequentist model in Lavaan). The model is specified as follows:

1.  A dependent variable we want to predict.
2.  A "~", that we use to indicate that we now give the other variables of interest.
    (comparable to the '=' of the regression equation).
3.  The different independent variables separated by the summation symbol '+'.
4.  Finally, we insert that the dependent variable has a variance and that we
    want an intercept.

For more information on the basics of (b)lavaan, see their [website](http://lavaan.ugent.be/tutorial/index.html) 

  <p>&nbsp;</p>

The following code is how to specify the regression model:

```r
# 1) specify the model
model.regression <- '#the regression model
                    diff ~ age + age2

                    #show that dependent variable has variance
                    diff ~~ diff

                    #we want to have an intercept
                    diff ~ 1'
```

Then, we need to fit the model by using the following code: We specify `target = "jags"` to tell Blavaan to use the Jags instead of the Stan compiler. 


```r
fit.bayes <- blavaan(model = model.regression, data = dataPHD, convergence = "auto",  target = "jags",  test = "none", seed = c(12,34,56))

# the test="none" input stops the calculations of some posterior checks, we do not need at this moment and speeds up the process. 
# the seed command is simply to guarantee the same exact result when running the sampler multiple times. You do not have to set this. When using Jags you need to set as many seeds as chains (by default)
```

Now we will have a look at the summary by using `summary(fit.bayes)`.

[expand title="Show Output" trigclass="noarrow my_button" targclass="my_content" tag="button"]


```r
summary(fit.bayes)
```

```
## blavaan (0.3-6) results of 37482 samples after 16000 adapt/burnin iterations
## 
##   Number of observations                           333
## 
##   Number of missing patterns                         1
## 
##   Statistic                               
##   Value                                   
## 
## Parameter Estimates:
## 
## 
## Regressions:
##                    Estimate    Post.SD  HPD.025    HPD.975       PSRF
##   diff ~                                                             
##     age                 2.387    0.538      1.301      3.389    1.010
##     age2               -0.023    0.006     -0.034     -0.011    1.009
##     Prior        
##                  
##     dnorm(0,1e-2)
##     dnorm(0,1e-2)
## 
## Intercepts:
##                    Estimate    Post.SD  HPD.025    HPD.975       PSRF
##    .diff              -41.358   11.298    -62.896    -19.001    1.009
##     Prior        
##     dnorm(0,1e-3)
## 
## Variances:
##                    Estimate    Post.SD  HPD.025    HPD.975       PSRF
##    .diff              194.005   14.916    165.491    223.279    1.000
##     Prior        
##  dgamma(1,.5)[sd]
```
[/expand]


  <p>&nbsp;</p> 

The results that stem from a Bayesian analysis are genuinely different from those that are provided by a frequentist model. 

[expand title=Read more on the Bayesian analysis]

The key difference between Bayesian statistical inference and frequentist statistical methods concerns the nature of the unknown parameters that you are trying to estimate. In the frequentist framework, a parameter of interest is assumed to be unknown, but fixed. That is, it is assumed that in the population there is only one true population parameter, for example, one true mean or one true regression coefficient. In the Bayesian view of subjective probability, all unknown parameters are treated as uncertain and therefore are be described by a probability distribution. Every parameter is unknown, and everything unknown receives a distribution.


This is why in frequentist inference, you are primarily provided with a point estimate of the unknown but fixed population parameter. This is the parameter value that, given the data, is most likely in the population. An accompanying confidence interval tries to give you further insight in the uncertainty that is attached to this estimate. It is important to realize that a confidence interval simply constitutes a simulation quantity. Over an infinite number of samples taken from the population, the procedure to construct a (95%) confidence interval will let it contain the true population value 95% of the time. This does not provide you with any information how probable it is that the population parameter lies within the confidence interval boundaries that you observe in your very specific and sole sample that you are analyzing. 

In Bayesian analyses, the key to your inference is the parameter of interest&#39;s posterior distribution. It fulfills every property of a probability distribution and quantifies how probable it is for the population parameter to lie in certain regions. On the one hand, you can characterize the posterior by its mode. This is the parameter value that, given the data and its prior probability, is most probable in the population. Alternatively, you can use the posterior&#39;s mean or median. Using the same distribution, you can construct a 95% credibility interval, the counterpart to the confidence interval in frequentist statistics. Other than the confidence interval, the Bayesian counterpart directly quantifies the probability that the population value lies within certain limits. There is a 95% probability that the parameter value of interest lies within the boundaries of the 95% credibility interval. Unlike the confidence interval, this is not merely a simulation quantity, but a concise and intuitive probability statement. For more on how to interpret Bayesian analysis, check [Van de Schoot et al. 2014](http://onlinelibrary.wiley.com/doi/10.1111/cdev.12169/abstract).

[/expand]

  <p>&nbsp;</p> 

##### _**Question:** Interpret the estimated effect, its interval and the posterior distribution._

[expand title="Answer" trigclass="noarrow my_button" targclass="my_content" tag="button"]





_$Age$ seems to be a relevant predictor of PhD delays, with a posterior mean regression coefficient of 2.387, 95% HPD (Credibility Interval) [1.301 3.389]. Also, $age^2$ seems to be a relevant predictor of PhD delays, with a posterior mean of -0.023, and a 95% Credibility Interval of [-0.034 -0.011]. The 95% HPD shows that there is a 95% probability that these regression coefficients in the population lie within the corresponding intervals, see also the posterior distributions in the figures below. Since 0 is not contained in the Credibility Interval we can be fairly sure there is an effect._





[/expand]

  <p>&nbsp;</p> 

##### _**Question:** Every Bayesian model uses a prior distribution. Describe the shape of the prior distributions of the regression coefficients._
[expand title="Answer" trigclass="noarrow my_button" targclass="my_content" tag="button"]


To check which default priors are being used by Blavaan, you can use the `dpriors()` command. 


```r
dpriors(target = "jags")
```

```
##                 nu              alpha             lambda 
##    "dnorm(0,1e-3)"    "dnorm(0,1e-2)"    "dnorm(0,1e-2)" 
##               beta             itheta               ipsi 
##    "dnorm(0,1e-2)" "dgamma(1,.5)[sd]" "dgamma(1,.5)[sd]" 
##                rho              ibpsi                tau 
##       "dbeta(1,1)"    "dwish(iden,3)"      "dnorm(0,.1)" 
##              delta 
## "dgamma(1,.5)[sd]"
```


_The priors used for the regression coefficients (beta) are flat and allow probability mass across the entire parameter space. Blavaan (Jags) makes use of a very wide normal distribution to come to this uninformative prior. By default the mean is 0, with a standard deviation of 10 (precision of .01). You can decide whether or not this is uninformative enough or not for your situation._


[/expand]

  <p>&nbsp;</p> 


## Regression - User-specified Priors

In Blavaan, you can also manually specify your prior distributions. Be aware that usually, this has to be done BEFORE peeking at the data, otherwise you are double-dipping (!). In theory, you can specify your prior knowledge using any kind of distribution you like. However, if your prior distribution does not follow the same parametric form as your likelihood, calculating the model can be computationally intense. _Conjugate_ priors avoid this issue, as they take on a functional form that is suitable for the model that you are constructing. For your normal linear regression model, conjugacy is reached if the priors for your regression parameters are specified using normal distributions (the residual variance receives an inverse gamma distribution, which is neglected here). In Blavaan, you are quite flexible in the specification of informative priors.

Let&#39;s re-specify the regression model of the exercise above, using conjugate priors. We leave the priors for the intercept and the residual variance untouched for the moment. Regarding your regression parameters, you need to specify the hyperparameters of their normal distribution, which are the mean and the variance. The mean indicates which parameter value you deem most likely. The variance expresses how certain you are about that. We try 4 different prior specifications, for both the $\beta_{age}$ regression coefficient, and the $\beta_{age^2}$ coefficient.

First, we use the following prior specifications:

_$Age$ ~  $\mathcal{N}(3, 0.4)$_

_$Age^2$ ~  $\mathcal{N}(0, 0.1)$_

In Blavaan, the priors are set in the model formulation step. Be careful, Blavaan uses precision instead of variance in the normal distribution. The precision is the reciprocal of the variance, so a variance of 0.1 corresponds to a precision of 10 and a variance of 0.4 corresponds to a precision of 2.5

The priors are presented in code as follows:



```r
model.informative.priors1 <- 
                    '#the regression model with priors
                    diff ~ prior("dnorm(3,2.5)")*age + prior("dnorm(0,10)")*age2

                    #show that dependent variable has variance
                    diff ~~ diff

                    #we want to have an intercept (with normal prior)
                    diff ~ 1'
```

Now fit the model and request for summary statistics. 

[expand title="Answer" trigclass="noarrow my_button" targclass="my_content" tag="button"]


```r
fit.bayes.infprior1 <- blavaan(model = model.informative.priors1, data = dataPHD, convergence = "auto",  target = "jags",  test = "none", seed = c(12,34,56))
```


```r
summary(fit.bayes.infprior1, fit.measures=TRUE, ci = TRUE, rsquare=TRUE)
```

```
## blavaan (0.3-6) results of 33734 samples after 5000 adapt/burnin iterations
## 
##   Number of observations                           333
## 
##   Number of missing patterns                         1
## 
##   Statistic                               
##   Value                                   
## 
## Parameter Estimates:
## 
## 
## Regressions:
##                    Estimate    Post.SD  HPD.025    HPD.975       PSRF
##   diff ~                                                             
##     age                 2.610    0.411      1.818      3.387    1.005
##     age2               -0.025    0.004     -0.033     -0.017    1.004
##     Prior        
##                  
##      dnorm(3,2.5)
##       dnorm(0,10)
## 
## Intercepts:
##                    Estimate    Post.SD  HPD.025    HPD.975       PSRF
##    .diff              -46.022    8.723    -62.406    -29.163    1.005
##     Prior        
##     dnorm(0,1e-3)
## 
## Variances:
##                    Estimate    Post.SD  HPD.025    HPD.975       PSRF
##    .diff              193.797   14.841    165.345    223.411    1.000
##     Prior        
##  dgamma(1,.5)[sd]
## 
## R-Square:
##                    Estimate  
##     diff                0.061
```


[/expand]


#### _**Question** Fill in the results in the table below:_


| $Age$          | Default prior              |$\mathcal{N}(3, .4)$| $\mathcal{N}(3, 1000)$|$\mathcal{N}(20, .4)$|$\mathcal{N}(20, 1000)$|
| ---            | ---                        | ---                    | ---        |---         | ---         |
| Posterior mean |  2.387|                        |            |            |             | 
| Posterior sd   |  0.538|                        |            |            |             |

| $Age^2$        | Default prior             | $\mathcal{N}(0, .1)$   |  $\mathcal{N}(0, 1000)$ |  $\mathcal{N}(20, .1)$|  $\mathcal{N}(20, 1000)$ |
| ---            | ---                       | ---       | ---        | ---        | ---         |
| Posterior mean | -0.023|           |            |            |             |
| Posterior sd   | 0.006|           |            |            |             |


[expand title="Answer" trigclass="noarrow my_button" targclass="my_content" tag="button"]





| $Age$          | Default prior              |$\mathcal{N}(3, .4)$| $\mathcal{N}(3, 1000)$|$\mathcal{N}(20, .4)$|$\mathcal{N}(20, 1000)$|
| ---            | ---                        | ---                      | ---        |---         | ---         |
| Posterior mean |  2.387|2.61|            |            |             | 
| Posterior sd   |  0.538|0.411|            |            |             |

| $Age^2$        | Default prior              | $\mathcal{N}(0, .1)$      |  $\mathcal{N}(0, 1000)$ |  $\mathcal{N}(20, .1)$|  $\mathcal{N}(20, 1000)$ |
| ---            | ---                        | ---                       | ---        | ---        | ---         |
| Posterior mean | -0.023 | -0.025|            |            |             |
| Posterior sd   | 0.006 | 0.004|            |            |             |




[/expand]

Next, try to adapt the code, using the prior specifications of the other columns and then complete the table.


[expand title="Answer" trigclass="noarrow my_button" targclass="my_content" tag="button"]




```r
model.informative.priors2 <- 
                    '#the regression model with priors
                    diff ~ prior("dnorm(3,0.001)")*age + prior("dnorm(0,0.001)")*age2

                    #show that dependent variable has variance
                    diff ~~ diff

                    #we want to have an intercept (with normal prior)
                    diff ~ 1'
```




```r
model.informative.priors3 <- 
                    '#the regression model with priors
                    diff ~ prior("dnorm(20,2.5)")*age + prior("dnorm(20,10)")*age2

                    #show that dependent variable has variance
                    diff ~~ diff

                    #we want to have an intercept (with normal prior)
                    diff ~ 1'
```




```r
model.informative.priors4 <- 
                    '#the regression model with priors
                    diff ~ prior("dnorm(20,0.001)")*age + prior("dnorm(20,0.001)")*age2

                    #show that dependent variable has variance
                    diff ~~ diff

                    #we want to have an intercept (with normal prior)
                    diff ~ 1'
```



```r
fit.bayes.infprior2 <- blavaan(model.informative.priors2, data = dataPHD, convergence = "auto",  target = "jags",  test = "none", seed = c(12,34,56))
fit.bayes.infprior3 <- blavaan(model.informative.priors3, data = dataPHD, convergence = "auto",  target = "jags",  test = "none", seed = c(12,34,56))
fit.bayes.infprior4 <- blavaan(model.informative.priors4, data = dataPHD, convergence = "auto",  target = "jags",  test = "none", seed = c(12,34,56))
```




```r
summary(fit.bayes.infprior2, fit.measures=TRUE, ci = TRUE, rsquare=TRUE)
```

```
## blavaan (0.3-6) results of 41262 samples after 5000 adapt/burnin iterations
## 
##   Number of observations                           333
## 
##   Number of missing patterns                         1
## 
##   Statistic                               
##   Value                                   
## 
## Parameter Estimates:
## 
## 
## Regressions:
##                    Estimate    Post.SD  HPD.025    HPD.975       PSRF
##   diff ~                                                             
##     age                 2.345    0.511      1.304      3.308    1.029
##     age2               -0.023    0.005     -0.033     -0.012    1.028
##     Prior        
##                  
##    dnorm(3,0.001)
##    dnorm(0,0.001)
## 
## Intercepts:
##                    Estimate    Post.SD  HPD.025    HPD.975       PSRF
##    .diff              -40.498   10.760    -61.166    -18.896    1.029
##     Prior        
##     dnorm(0,1e-3)
## 
## Variances:
##                    Estimate    Post.SD  HPD.025    HPD.975       PSRF
##    .diff              194.106   14.959    165.944    224.273    1.000
##     Prior        
##  dgamma(1,.5)[sd]
## 
## R-Square:
##                    Estimate  
##     diff                0.050
```

```r
summary(fit.bayes.infprior3, fit.measures=TRUE, ci = TRUE, rsquare=TRUE)
```

```
## blavaan (0.3-6) results of 21169 samples after 5000 adapt/burnin iterations
## 
##   Number of observations                           333
## 
##   Number of missing patterns                         1
## 
##   Statistic                               
##   Value                                   
## 
## Parameter Estimates:
## 
## 
## Regressions:
##                    Estimate    Post.SD  HPD.025    HPD.975       PSRF
##   diff ~                                                             
##     age                11.105    0.591      9.984      12.26    1.018
##     age2               -0.112    0.006     -0.125     -0.101    1.016
##     Prior        
##                  
##     dnorm(20,2.5)
##      dnorm(20,10)
## 
## Intercepts:
##                    Estimate    Post.SD  HPD.025    HPD.975       PSRF
##    .diff             -223.417   12.443    -247.53   -199.384    1.017
##     Prior        
##     dnorm(0,1e-3)
## 
## Variances:
##                    Estimate    Post.SD  HPD.025    HPD.975       PSRF
##    .diff              314.070   29.547    257.144    372.735    1.004
##     Prior        
##  dgamma(1,.5)[sd]
## 
## R-Square:
##                    Estimate  
##     diff                0.404
```

```r
summary(fit.bayes.infprior4, fit.measures=TRUE, ci = TRUE, rsquare=TRUE)
```

```
## blavaan (0.3-6) results of 24714 samples after 49000 adapt/burnin iterations
## 
##   Number of observations                           333
## 
##   Number of missing patterns                         1
## 
##   Statistic                               
##   Value                                   
## 
## Parameter Estimates:
## 
## 
## Regressions:
##                    Estimate    Post.SD  HPD.025    HPD.975       PSRF
##   diff ~                                                             
##     age                 2.312    0.547      1.125      3.282    1.022
##     age2               -0.022    0.006     -0.033      -0.01    1.021
##     Prior        
##                  
##   dnorm(20,0.001)
##   dnorm(20,0.001)
## 
## Intercepts:
##                    Estimate    Post.SD  HPD.025    HPD.975       PSRF
##    .diff              -39.789   11.508    -60.407    -15.358    1.022
##     Prior        
##     dnorm(0,1e-3)
## 
## Variances:
##                    Estimate    Post.SD  HPD.025    HPD.975       PSRF
##    .diff              194.214   15.076    165.241    223.815    1.000
##     Prior        
##  dgamma(1,.5)[sd]
## 
## R-Square:
##                    Estimate  
##     diff                0.049
```




  
| $Age$          | Default prior              |$\mathcal{N}(3, .4)$      | $\mathcal{N}(3, 1000)$    |$\mathcal{N}(20, .4)$    |$\mathcal{N}(20, 1000)$   |
| ---            | ---                        | ---                      | ---                       |---                       | ---                      |
| Posterior mean |  2.387|2.61| 2.345|11.105|2.312| 
| Posterior sd   |  0.538|0.411| 0.511|0.591|0.547|

| $Age^2$        | Default prior              | $\mathcal{N}(0, .1)$      |  $\mathcal{N}(0, 1000)$ |  $\mathcal{N}(20, .1)$|  $\mathcal{N}(20, 1000)$ |
| ---            | ---                        | ---                       | ---                        | ---                      | ---                      |
| Posterior mean | -0.023 | -0.025|-0.023  |-0.112|-0.022|
| Posterior sd   | 0.006 | 0.004|0.005  |0.006|0.006|

[/expand]



#### _**Question**: Compare the results over the different prior specifications. Are the results comparable with the default model?_


#### _**Question**: Do we end up with similar conclusions, using different prior specifications?_


To answer these questions, proceed as follows:
We can calculate the relative bias to express this difference. ($bias= 100*\frac{(model informative priors-model uninformed priors)}{model uninformative priors}$). In order to preserve clarity we will just calculate the bias of the two regression coefficients and only compare the default (uninformative) model with the model that uses the $\mathcal{N}(20, .4)$ and $\mathcal{N}(20, .1)$ priors. Copy Paste the following code to R:



```r
#1) substract the MCMC chains
mcmc.list <- blavInspect(fit.bayes, what = "mcmc")
mcmc.list.informative <- blavInspect(fit.bayes.infprior3, what = "mcmc")

#2) bind the different chains an calculate the mean (estimate) of the regression coefficients
estimatesuninformative <- colMeans(as.matrix(mcmc.list)[,c("beta[1,2,1]","beta[1,3,1]")])
estimatesinformative   <- colMeans(as.matrix(mcmc.list.informative)[,c("beta[1,2,1]","beta[1,3,1]")])

#3) calculate the bias
round(100*((estimatesinformative-estimatesuninformative)/estimatesuninformative), 2)
```


The `beta[1,2,1]` and `beta[1,3,1]` indices stand for the $\beta_{age}$ and $\beta_{age^2}$ respectively. These somewhat confusing indices are what Blavaan receives from the MCMC sampler in JAGS These are just the labels JAGS gives to the parameters and Blavaan doesn't really translate them back to the input names we gave them. There is some order to it though. Beta[x,x,x] are regression coefficients (in the order we specified them in the model, so first $age$ and then $age^2$), alpha[x,x,x] are intercepts, psi[x,x,x] are variances, and def[x,x,x] are indirect effects (if you have those in your model). They are listed in the same order as they are in the `summary()` output. So first regression coefficients, then intercepts, then (co)variances, and then indirect effects. 

We can also plot these differences by plotting both the posterior and priors for the five different models we ran. In this example we only plot the regression of coefficient of age $\beta_{age}$.

First we extract the MCMC chains of the 5 different models for only this one parameter ($\beta_{age}$=beta[1,2,1]). Copy-past the following code to R: 

[expand title="Show Syntax" trigclass="noarrow my_button" targclass="my_content" tag="button"]



```r
posterior1.2.3.4.5 <- bind_rows("uninformative prior" = enframe(as.matrix(blavInspect(fit.bayes, what="mcmc"))[,'beta[1,2,1]']),
                                "informative prior 1" = enframe(as.matrix(blavInspect(fit.bayes.infprior1, what="mcmc"))[,'beta[1,2,1]']),
                                "informative prior 2" = enframe(as.matrix(blavInspect(fit.bayes.infprior2, what="mcmc"))[,'beta[1,2,1]']),
                                "informative prior 3" = enframe(as.matrix(blavInspect(fit.bayes.infprior3, what="mcmc"))[,'beta[1,2,1]']),
                                "informative prior 4" = enframe(as.matrix(blavInspect(fit.bayes.infprior4, what="mcmc"))[,'beta[1,2,1]']),
                                .id = "id1")

prior1.2.3.4.5 <- bind_rows("uninformative prior" = enframe(rnorm(10000, mean=0, sd=sqrt(1/1e-2))),
                            "informative prior 1" = enframe(rnorm(10000, mean=3, sd=sqrt(0.4))),
                            "informative prior 2" = enframe(rnorm(10000, mean=3, sd=sqrt(1000))),
                            "informative prior 3" = enframe(rnorm(10000, mean=20, sd=sqrt(0.4))),
                            "informative prior 4" = enframe(rnorm(10000, mean=20, sd=sqrt(1000))),
                            .id = "id1")# here we sample a large number of values from the prior distributions to be able to plot them.

priors.posterior <- bind_rows("posterior" = posterior1.2.3.4.5, "prior" = prior1.2.3.4.5,  .id = "id2")
```

[/expand]

Then, we can plot the different posteriors and priors by using the following code:

[expand title="Show Syntax" trigclass="noarrow my_button" targclass="my_content" tag="button"]


```r
ggplot(data    = priors.posterior, 
       mapping = aes(x        = value,
                     fill     = id1,
                     colour   = id2,
                     linetype = id2, 
                     alpha    = id2))+
  geom_density(size = 1)+
  scale_x_continuous(limits  = c(0, 23))+
  scale_colour_manual(name   = 'Posterior/Prior', values = c("black","red"),    labels = c("posterior", "prior"))+
  scale_linetype_manual(name = 'Posterior/Prior', values = c("solid","dotted"), labels = c("posterior", "prior"))+
  scale_alpha_discrete(name  = 'Posterior/Prior', range  = c(.7,.3),            labels = c("posterior", "prior"))+
  scale_fill_manual(name = "Densities", values = c("Yellow","darkred","blue", "green", "pink"))+
  labs(title    = expression("Influence of (Informative) Priors on" ~ beta[Age]), 
       subtitle = "5 different densities of priors and posteriors")
```

![](Blavaan-Jags-Regression_files/figure-html/unnamed-chunk-23-1.png)<!-- -->
[/expand]

Now, with the information from the table, the bias estimates and the plot you can answer the two questions about the influence of the priors on the results. 

[expand title="Answer" trigclass="noarrow my_button" targclass="my_content" tag="button"]



```r
#1) substract the MCMC chains
mcmc.list <- blavInspect(fit.bayes, what = "mcmc")
mcmc.list.informative <- blavInspect(fit.bayes.infprior3, what = "mcmc")

#2) bind the different chains an calculate the mean (estimate) of the regression coefficients
estimatesuninformative <- colMeans(as.matrix(mcmc.list)[,c("beta[1,2,1]","beta[1,3,1]")])
estimatesinformative   <- colMeans(as.matrix(mcmc.list.informative)[,c("beta[1,2,1]","beta[1,3,1]")])

#3) calculate the bias
round(100*((estimatesinformative-estimatesuninformative)/estimatesuninformative), 2)
```

```
## beta[1,2,1] beta[1,3,1] 
##      364.20      385.66
```



_We see that the influence of this highly informative prior is around 364% and 386% on the two regression coefficients respectively. This is a large difference and we thus certainly would not end up with similar conclusions._

_The results change with different prior specifications, but are still comparable. Only using $\mathcal{N}(20, .4)$ for age, results in a really different coefficients, since this prior mean is far from the mean of the data, while its variance is quite certain. However, in general the other results are comparable. Because we use a big dataset the influence of the prior is relatively small. If one would use a smaller dataset the influence of the priors are larger. To check this you can use these lines to sample roughly 20% of all cases and redo the same analysis. The results will of course be different because we use many fewer cases (probably too few!). Use this code._


```r
indices   <- sample.int(333, 60)
smalldata <- data[indices,]
```

_We made a new dataset with randomly chosen 60 of the 333 observations from the original dataset. You can repeat the analyses with the same code and only changing the name of the dataset to see the influence of priors on a smaller dataset. Run the model model.informative.priors2 with this new dataset_ 



```r
fit.bayes.infprior2_small <- blavaan(model = model.informative.priors2, data = smalldata, convergence = "auto",  target = "jags",  test = "none", seed = c(12,34,56))
summary(fit.bayes.infprior2_small, fit.measures=TRUE, ci = TRUE, rsquare=TRUE)
```

[/expand]

If you really want to use Bayes for your own data, we recommend to follow the [WAMBS-checklist](https://www.rensvandeschoot.com/wambs-checklist/), which you are guided through by this exercise.


--- 
### **References**

_Benjamin, D. J., Berger, J., Johannesson, M., Nosek, B. A., Wagenmakers, E.,... Johnson, V. (2017, July 22)._ [Redefine statistical significance](https://psyarxiv.com/mky9j)_. Retrieved from psyarxiv.com/mky9j_

_Greenland, S., Senn, S. J., Rothman, K. J., Carlin, J. B., Poole, C., Goodman, S. N. Altman, D. G. (2016)._ [Statistical tests, P values, confidence intervals, and power: a guide to misinterpretations](https://link.springer.com/article/10.1007/s10654-016-0149-3)_._ _European Journal of Epidemiology 31 (4_). [_https://doi.org/10.1007/s10654-016-0149-3_](https://doi.org/10.1007/s10654-016-0149-3) _   _

_ Rosseel, Y. (2012)._ [lavaan: An R Package for Structural Equation Modeling](https://www.jstatsoft.org/article/view/v048i02). _Journal of Statistical Software, 48(2), 1-36. _

_van de Schoot R, Yerkes MA, Mouw JM, Sonneveld H (2013)_ [What Took Them So Long? Explaining PhD Delays among Doctoral Candidates](http://journals.plos.org/plosone/article?id=10.1371/journal.pone.0068839)_._ _PLoS ONE 8(7): e68839._ [https://doi.org/10.1371/journal.pone.0068839](https://doi.org/10.1371/journal.pone.0068839)

Trafimow D, Amrhein V, Areshenkoff CN, Barrera-Causil C, Beh EJ, Bilgi? Y, Bono R, Bradley MT, Briggs WM, Cepeda-Freyre HA, Chaigneau SE, Ciocca DR, Carlos Correa J, Cousineau D, de Boer MR, Dhar SS, Dolgov I, G?mez-Benito J, Grendar M, Grice J, Guerrero-Gimenez ME, Guti?rrez A, Huedo-Medina TB, Jaffe K, Janyan A, Karimnezhad A, Korner-Nievergelt F, Kosugi K, Lachmair M, Ledesma R, Limongi R, Liuzza MT, Lombardo R, Marks M, Meinlschmidt G, Nalborczyk L, Nguyen HT, Ospina R, Perezgonzalez JD, Pfister R, Rahona JJ, Rodr?guez-Medina DA, Rom?o X, Ruiz-Fern?ndez S, Suarez I, Tegethoff M, Tejo M, ** van de Schoot R** , Vankov I, Velasco-Forero S, Wang T, Yamada Y, Zoppino FC, Marmolejo-Ramos F. (2017)  [Manipulating the alpha level cannot cure significance testing - comments on &quot;Redefine statistical significance&quot;](https://www.rensvandeschoot.com/manipulating-alpha-level-cannot-cure-significance-testing/)_ _PeerJ reprints 5:e3411v1   [https://doi.org/10.7287/peerj.preprints.3411v1](https://doi.org/10.7287/peerj.preprints.3411v1)


