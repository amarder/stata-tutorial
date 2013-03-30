---
layout: post
title: [xtreg fe] Clustered Standard Errors Too Small!
---

I have been implementing a fixed-effects estimator in Python so I can work with data that is too large to hold in memor
> y.  To make sure I was calculating my coefficients and standard errors correctly I have been comparing the calculatio
> ns of my Python code to results from Stata.  This lead me to find a surprising inconsistency in Stata's calculation o
> f standard errors.  I illustrate the issue by comparing standard errors computed by Stata's `xtreg fe` command to tho
> se computed by the standard `regress`.  To start, I use the first hundred observations of the `nlswork` dataset:

    . webuse nlswork, clear
    (National Longitudinal Survey.  Young Women 14-26 years of age in 1968)
    
    . keep if _n <= 100
    (28434 observations deleted)

I then estimate a fixed-effects model using regress:

    . regress ln_w age tenure i.idcode, vce(cluster idcode)
    
    Linear regression                                      Number of obs =      99
                                                           F(  1,     8) =       .
                                                           Prob > F      =       .
                                                           R-squared     =  0.5172
                                                           Root MSE      =  .26493
    
                                     (Std. Err. adjusted for 9 clusters in idcode)
    ------------------------------------------------------------------------------
                 |               Robust
         ln_wage |      Coef.   Std. Err.      t    P>|t|     [95% Conf. Interval]
    -------------+----------------------------------------------------------------
             age |   .0025964   .0159834     0.16   0.875    -.0342613    .0394541
          tenure |   .0356595      .0165     2.16   0.063    -.0023896    .0737086
                 |
          idcode |
              2  |  -.4282753   .0207949   -20.60   0.000    -.4762285   -.3803222
              3  |  -.5313624   .0531705    -9.99   0.000    -.6539738    -.408751
              4  |  -.1054791   .0862957    -1.22   0.256    -.3044774    .0935192
              5  |  -.1976049   .0275788    -7.17   0.000    -.2612016   -.1340081
              6  |  -.4590234   .0484582    -9.47   0.000    -.5707681   -.3472787
              7  |  -.7201447   .0362701   -19.86   0.000    -.8037837   -.6365058
              9  |  -.2538001   .1237821    -2.05   0.074    -.5392421     .031642
             10  |  -.5417859   .1337093    -4.05   0.004    -.8501201   -.2334518
                 |
           _cons |   1.922783   .4005497     4.80   0.001     .9991134    2.846452
    ------------------------------------------------------------------------------
    
    . matrix regress = e(V)

For comparison, I then estimate the same model using `xtreg fe`:

    . xtset idcode
           panel variable:  idcode (unbalanced)
    
    . xtreg ln_w age tenure, fe vce(cluster idcode)
    
    Fixed-effects (within) regression               Number of obs      =        99
    Group variable: idcode                          Number of groups   =         9
    
    R-sq:  within  = 0.2358                         Obs per group: min =         3
           between = 0.1373                                        avg =      11.0
           overall = 0.1740                                        max =        15
    
                                                    F(2,8)             =     14.98
    corr(u_i, Xb)  = -0.0963                        Prob > F           =    0.0020
    
                                     (Std. Err. adjusted for 9 clusters in idcode)
    ------------------------------------------------------------------------------
                 |               Robust
         ln_wage |      Coef.   Std. Err.      t    P>|t|     [95% Conf. Interval]
    -------------+----------------------------------------------------------------
             age |   .0025964   .0153029     0.17   0.869    -.0326922    .0378849
          tenure |   .0356595   .0157976     2.26   0.054    -.0007698    .0720888
           _cons |   1.583834   .3793026     4.18   0.003     .7091607    2.458508
    -------------+----------------------------------------------------------------
         sigma_u |  .23415096
         sigma_e |  .26492811
             rho |  .43856575   (fraction of variance due to u_i)
    ------------------------------------------------------------------------------
    
    . matrix xtreg = e(V)

Notice that the coefficients on age and tenure match perfectly across the two regressions, but the standard errors calc
> ulated by xtreg fe are smaller.  Experimenting with my own code, I believe this discrepancy is due to a degrees of fr
> eedom correction.  As Kevin Goulding explains [here](http://thetarzan.wordpress.com/2011/06/11/clustered-standard-err
> ors-in-r/), clustered standard errors are generally computed by multiplying the estimated asymptotic variance by (M /
>  (M - 1)) ((N - 1) / (N - K)).  M is the number of individuals, N is the number of observations, and K is the number 
> of parameters estimated.  The standard regress command correctly sets K = 12, xtreg fe sets K = 3.  Making the asympt
> otic variance (99 - 12) / (99 - 3) = 0.90625 times the correct value.

    . display ((99 - 12) / (99 - 3)) * regress[1, 1] - xtreg[1, 1]
    -2.661e-06
          name:  <unnamed>
           log:  /Users/amarder/Desktop/stata-tutorial/clustered-standard-errors.md1
      log type:  text
     closed on:  29 Mar 2013, 22:20:59
    -----------------------------------------------------------------------------------------------------------------------
