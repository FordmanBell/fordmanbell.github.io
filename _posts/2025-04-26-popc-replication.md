---
layout: post
title: Principals of Principle Components Paper Replication
---

Principle component analysis (PCA) is an important tool in yield curve analysis. It helps traders interpret yield curve movements by breaking them down into three main components: level, slope, and curvature. For those who are unfamiliar with PCA, a great introduction can be found [here](https://gregorygundersen.com/blog/2022/09/17/pca/){:target="_blank"}. In this post, I will attempt to replicate the results in "Principles of Priciple Components", a classic paper by Salomon Brothers that discusses in detail the application of PCA in hedging treasury portfolios and constructing butterfly trades. 

The code for this post can be found [here](https://github.com/FordmanBell/Rates-Projects){:target="_blank"}. I’d also like to credit [Analytic Musings](https://analytic-musings.com/2023/12/31/principles-of-principal-components/){:target="_blank"} (who also happens to use the same blog template, what a coincidence) and [Goutham Balaraman](https://gouthamanbalaraman.com/blog/quantlib-term-structure-bootstrap-yield-curve.html){:target="_blank"} for articles that helped guide parts of this post.


The Salomon paper is divided into two parts. In the first part, the author performs PCA on yield curve data in the 90s and uses it to construct hedges that correspond to each principle component. To replicate this, we first gather US Treasury weekly yield curve data from 01/01/1990 to 02/28/1998. Note that while the paper uses data starting in January 1989, we begin in 1990 due to availability. 

The next step is to interpolate the yield curve to obtain yields at more granular intervals. Specifically, we generate par yields at every 3-month interval from 3 months to 30 years, resulting in 120 yield curve points for each date, just like the paper. However, note that there will be small discrepencies between our yield curves, which are interpolated from a few par yields (3m, 6m, 1y, 2y, 5y, 10y, 30y) and the yield curves in the paper, which are based on many more treasury securities.

We can now fit a PCA model to the yield curve data. Following the paper, we will fit the PCA to the change in yields, although level is also acceptable as input. The resulting explained variances of each component closely resembles those in the paper.

<br>
![Alt text](/images/pca_replication/repl/explained_variance.png)
![Alt text](/images/pca_replication/orig/explained_variance.png)
<br>

The first 3 components of the PCA, when scaled to a 1-month timeframe, also closely follows the results in the paper, with only PC2 having flipped signs.

{% include pca_replication/repl/pca_components.html %}
![Alt text](/images/pca_replication/orig/pca_components.png)
<br>

With the fitted PCA, we can now construct trades that can hedge out movements in each of the 3 components. For each trade, we select the 2y, 10y, and 30y par bonds as our securities of choice. We then calculate two types of weights for each trade: the duration weights and the market values of the securities to purchase for the trade.

The duration weights represent how much duration is assigned to each security to create a trade that isolates exposure to a single principal component. To calculate these weights, take the cross product of the loadings of the unwanted components, then normalize the resulting vector. For example, suppose M is a 3×3 matrix where M[i,j] represents the loading of the i-th principal component on the j-th security. To create a trade that is exposed only to curvature (the third component), take the cross product of the first and second rows of M, and normalize the result.

The market values of the securities represent the nominal amounts of each security to purchace/sell for a trade. To calculate the market values:

1. Compute the 3x3 matrix D, where D[i, j] represents the i-th security's delta (in %) with regard to a 1sd move in the j-th component.
2. Construct the 3x1 column vector y, where y[i] is the desired dollar exposure to a 1sd move in the ith component
3. Solve Dx=y, and x will be the market values of each security for the trade

Following the paper, we assume each trade will take on an exposure of $1 million for a 1-month, 1sd move in each component. Using the computation methods outlined above, we arrive at the following weights:

<br>
![Alt text](/images/pca_replication/repl/trade_weights.png)
![Alt text](/images/pca_replication/orig/trade_weights.png)
<br>

The tables show that our weights for the first and second principal components closely match those in the original paper, aside from a sign flip in the second component, which is due to the earlier inversion of the PCA result. The weights for the third trade differ more noticeably, most likely because of the discrepancies in the yield curve data. This makes sense, since the third PC is the most sensitive to variations in the input data.

Moving on to part 2 of the paper, we now analyze the 4y-10y-27y butterfly using daily data from June 6, 1997 to June 10, 1998. We begin by plotting the long spreads and the short spreads of the fly position during the data period. 

{% include pca_replication/repl/long_vs_short.html %}
![Alt text](/images/pca_replication/orig/long_vs_short.png)
<br>

The charts show that unlike part 1, our spreads of the long leg and the short leg differ more noticebly from that of the paper. Barring from programming erros, a possible explaination for this could be that our 4y and 27y yields are interpolated from nearby points (5y, 30y), where as the yields in the paper are extrapolated from market securities with closer maturities. This discrepency will also influence the other charts that involve the buterfly spread.

{% include pca_replication/repl/dv01_fly_spread.html %}
{% include pca_replication/repl/yield_and_slope.html %}
![Alt text](/images/pca_replication/orig/spread_yield_slope.png)
{% include pca_replication/repl/fly_vs_yield.html %}
![Alt text](/images/pca_replication/orig/fly_vs_yield.png)
{% include pca_replication/repl/fly_vs_slope.html %}
![Alt text](/images/pca_replication/orig/fly_vs_slope.png)
<br>

Apart from the yield and slope chart, the rest of the charts show notable differences from the paper. Therefore, future attempts should interpolate the yield curve using a broader set of market securities to improve accuracy.

This concludes my attempt at replicating the "Principles of Principal Components" paper. While the part 1 results mirrors the original, results from part 2 leaves one hoping for better. Further directions of exploration could involve performing PCA on more recent data (security price data since 2009 is publicly available on [TreasuryDirect](https://treasurydirect.gov/GA-FI/FedInvest/todaySecurityPriceDetail){:target="_blank"}, so a more recent analysis can bootstrap the yield curve from actual security prices), or writing code to backtest a particular trading strategy involving butterfly spreads, as done in the paper. 

