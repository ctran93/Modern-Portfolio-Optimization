# Modern Portfolio Optimization 

Modern Portfolio Theory is a mathematical method for selecting a variety of investments such as stocks, bonds, or treasury bills in order to optimize the overall returns in a period of time for a given level of risk. This method is also known as the mean-variance analysis since it takes into account the mean (expected return) and variance of each individual security to calculate the overall expected return and level of risk (variance) of the portfolio. The theory considers diversification as a key to portfolio management since it would help investors to reduce risks or/and increase overall return.

Based on the Modern Portfolio Theory, this optimization programming with **R** is going to calculate the minimum risk of the portfolio containing a set of securities for a given desirable overall return. The program is built to optimize a portfolio comprising five (5) different securities, including: 

* Apple Inc. (NASDAQ: AAPL),

* Meta Platforms, Inc. (NASDAQ: META),

* Occidental Petroleum (NYSE: OXY),

* Sea Limited (NYSE: SE),

* Tesla, Inc. (NASDAQ: TSLA).

Raw market data is collected from the first trading day of 2017 (Jan 03, 2017) to the current date. 

## Getting Started 
### Loading Libraries

```r
# Core
library(tidyverse)

# Finance analytics
library(tidyquant)

# Optimization Solving
library(quadprog)

# Visualization 
library(plotly)
```

### Getting Data 

Raw market data was collected from Yahoo! Finance with the function: **tq_get(get = "stock.prices")**. Quantitative data collected with this function include volume, opening, highest, lowest, closing, and adjusted price for every individual security on every trading day.  

Example: Collected data for NYSE: SNAP on Mar 09, 2021. 

```r
SNAP <- tq_get("SNAP", get = "stock.prices", complete_cases = TRUE, from = "2021-03-09", to = "2021-03-10")
SNAP
```

***Output***

| Symbol | Date | Open | High | Low | Close | Volume | Adjusted |
| ------------- | ------------- | ------------- | ------------- | ------------- | ------------- | ------------- | ------------- |
| SNAP | 2021-03-09 | 55.41 | 57.29	| 54.51 | 56.3 | 20761000 | 56.3 |

The following code is used for collecting quantitative market data for the five securities used for this portfolio optimization.

```r
stock <- c("AAPL", "META", "OXY", "SE", "TSLA")
data <- tq_get(stock, get = "stock.prices", complete_cases = TRUE, from = "2017-01-01", to = TODAY())
```

Yet, the previous code only gives the data for every individual stock on every trading day of a given time period. Further steps, then, need to be taken to process and transform market daily data into data that only include the annual return for every security.  

```r
returns_data <- data %>%
    select(symbol, date, adjusted) %>%
    group_by(symbol)%>%
    tq_transmute(adjusted, 
                 mutate_fun = periodReturn, 
                 period = "yearly", 
                 col_rename = "Annual Return")
returns_data %>%
   pivot_wider(names_from = 'symbol',values_from = 'Annual Return')
```

The output shows the **Annual Return** for each individual security from 2017 to the current year. 

***Output** (on 11/23/2022):* 

| Date  | AAPL | META | OXY | SE | TSLA |
| ------------- | ------------- | ------------- | ------------- | ------------- | ------------- |
| 2017-12-29 | 0.48042498 | 0.5100120 | 0.06924442 | -0.1801968 | 0.43485870 |
| 2018-12-31 | -0.05390189 | -0.2571121 | -0.13048936 | -0.1507877 | 0.06889353 |
| 2019-12-31 | 0.88957856 | 0.5657183 | -0.28283133 | 2.5530036 | 0.25700121 |
| 2020-12-31 | 0.82306715 | 0.3308648 | -0.56632817 | 3.9490303 | 7.43437001 |
| 2021-12-31 | 0.34648191 | 0.2313296 | 0.67710942 | 0.1238885 | 0.49755559 |
| 2022-11-23 | -0.14429929 | -0.6663000 | 1.46872210 | -0.7513298 | -0.47992962 |


## Securities Statistics 
### Individual Security

Calculating **Expected Return** and **Standard Deviation** for every individual stock:

```r
stats <- returns_data %>%
  summarise(
    Expected_Annual_Return = mean(`Annual Return`),
    Standard_Deviation = sd(`Annual Return`)
  )
stats
```

The output shows the **Expected Annual Return** and **Standard Deviation** for each individual security.

***Output** (on 11/23/2022):* 

| | Symbol  | Expected_Annual_Return | Standard_Deviation |
| ------------- | ------------- | ------------- | ------------- |
|1| AAPL  | 0.3902252 | 0.4312123 |
|2| META  | 0.1190854 | 0.4833443 |
|3| OXY  | 0.2059046 | 0.7461615 |
|4| SE  | 0.9239347 | 1.8775403 |
|5| TSLA  | 1.3687916 | 2.9921934 |

> The previous table illustrates the Expected Annual Return and Risk (given by the Deviation) of every individual stock. Since the standard deviation (and variance) is used for measuring how the actual return would deviate from the expected value, the highest standard deviation exhibits the greatest level of risk. That said, while TSLA would give a higher expected yearly return (nearly 140 %) than any of the other four securities, a portfolio only contains TSLA would face the highest risk with a standard deviation of 3. 

### Covariance 

The **covariance** is a mathematical quantity that measures the realationship between two variables. 

The mathematical covariance of any pair of securities is given by:

$\sigma_{ij}$ = $E{[(X_{i} - \mu_{i})(X_{j} - \mu_{j})]}$ where: 

* E(x) = expected value of x 

* $\sigma_{ij}$ = the covariance between securities i and j,

* $\mu_{i}$ = expected return of security i,

* $\mu_{i}$ = expected return of security j.


> A high positive covariance means the two stocks tend to move in similar directions. They will increase or decrease together. On the other hand, a large negative covariance means the two stocks tend to move in opposite directions. A covariance of approximately 0 means there is no clear relationship between the two stocks; and it could be concluded that the two stocks are independent. 

Calculating the **Covariance** for each pair of stocks:

```r
returns_matrix <- tidyr::spread(returns_data, 
                                key = "symbol", 
                                value = "Annual Return")
covariance_matrix <- cov(returns_matrix[,-1])
covariance_matrix
```

***Output** (on 11/23/2022):* 

```
           AAPL       META        OXY         SE       TSLA
AAPL  0.1859440  0.1863866 -0.2273701  0.6862112  0.7179273
META  0.1863866  0.2336217 -0.2495221  0.5133700  0.4532178
OXY  -0.2273701 -0.2495221  0.5567571 -1.0224762 -1.2641772
SE    0.6862112  0.5133700 -1.0224762  3.5251577  4.5520224
TSLA  0.7179273  0.4532178 -1.2641772  4.5520224  8.9532214
```

## Optimization Solving 

In the **Modern Portfolio Optimization**, the objective is the **minimum** of the portfolio variance, given by the function:

$\sigma_{p}^2 = \Sigma_{all_i}\Sigma_{all_j }x_{i}x_{j}\sigma_{ij}$

governed by the following constraints:

* $\mu_{p}$ = $\Sigma_{all_i}x_{i}\mu_{i}$ $\geq$ desirable overall expected return

* $\Sigma_{all_i}x_{i} = 1$

* $x_{i} \geq 0$ for all i  

in which:

* $\sigma_{p}^2$ = the variance of the portfolio,

* $\mu_{p}$ = the expected return of the portfolio,

* $x_{i}$ = the relative weights of securities i,

* $\mu_{i}$ = expected return of security i,

* $\sigma_{ij}$ = the covariance between securities i and j. 

In order to find the minimum risk (measured by the portfolio variance) given a desirable overall expected return, an advanced mathematical tool called **quadratic programming** is used.

The general quadratic programming problem:

min $(\frac{1}{2}x^TDx - d^Tx)$ subject to $A^Tx \geq b$

### Quadratic Programming Setup 

The following Quadratic Programming is set with the desirable expected annual return of 0.5 (50%)

```r
# Objective matrix
Dmat <- 2*covariance_matrix
#Objective vector
dvec <- c(0,0,0,0,0)
# Constraint Matrix
Amat <- t(matrix(c(1, as.numeric(stats[stats$symbol == "AAPL", "Expected_Annual_Return"]),1,0,0,0,0,
                   1,as.numeric(stats[stats$symbol == "META", "Expected_Annual_Return"]),0,1,0,0,0,
                   1,as.numeric(stats[stats$symbol == "OXY", "Expected_Annual_Return"]),0,0,1,0,0,
                   1,as.numeric(stats[stats$symbol == "SE", "Expected_Annual_Return"]),0,0,0,1,0,
                   1,as.numeric(stats[stats$symbol == "TSLA", "Expected_Annual_Return"]),0,0,0,0,1),7,5))
# Require Return (Set at 0.5)
goal <- 0.5
# Right hand side
bvec <- c(1,goal,0,0,0,0,0)
```

### Quadratic Programming Solving 

```r
qp <- solve.QP(Dmat, 
               dvec, 
               Amat, 
               bvec, 
               meq = 1)
portfolio_weights <- rbind(stock, round(qp$solution, 4))
portfolio_weights
portfolio_variance <- qp$value
portfolio_variance        
```

***Output** (on 11/23/2022):*

```
      [,1]     [,2]   [,3]    [,4]     [,5]    
stock "AAPL"   "META" "OXY"   "SE"     "TSLA"  
      "0.2043" "0"    "0.502" "0.1912" "0.1024"
[1] 0.2599802
```

> This output means, given the desirable expected annual return at **0.5** (~ 50% after one year), the portfolio would face the lowest risk at the variance of **0.26** if the portfolio consists of **20.4 %** of AAPL, **0 %** of META, **50.2 %** of OXY, **19.1 %** of SE, and **10.2 %** of TSLA.

### Double Checking Result

The following code is used for double-checking the output of the Quadratic Programming. That said, the relative weights of individual securities are going to be put back in functions that calculate the portfolio variance and expected return to make sure they match the portfolio variance calculated by the Quadratic Programming and the desirable return given.

```r
### For checking result only
calc_portfolio_variance <- function(weights) {
    t(weights) %*% (covariance_matrix %*% weights) %>% as.vector()
}
calc_portfolio_variance(qp$solution) #Should equals to portfolio_covariance
calc_portfolio_return <- function(weights) {
    stats_mean <- stats$Expected_Annual_Return
    sum(stats_mean * weights)
}
calc_portfolio_return(qp$solution) #Should equals to the desirable return.
```

***Output** (on 11/23/2022):*

```
[1] 0.2599802
[1] 0.5
```
The results match with the calculated variance of **0.26** and the given desirable return of **50 %**.

## Efficient Frontier Visualization 

**The Efficient Frontier** is all the scatter points, each representing a given portfolio's expected return and its corresponding minimum variance. Any portfolio located on the Efficient Frontier would have a higher expected return than any other portfolio having the same variance. Similarly, any portfolio located on the Efficient Frontier would have a lower risk (lower variance) than any other portfolio having the same expected return. Investors, thus, should only consider portfolios laying on the Efficient Frontier. 

The Efficient Frontier is limited by the minimum and maximum expected returns. While the minimum expected return is determined by the lowest expected return of all securities, the maximum expected return is determined by the highest one. No portfolio has an expected return beyond this bound. 

```r
port_return <- c()
port_variance <-c()
min_return_in_percent <- ceiling(min(stats$Expected_Annual_Return)*100)
max_return_in_percent <- floor(max(stats$Expected_Annual_Return)*100)
for (n in (min_return_in_percent:max_return_in_percent)){
  expected_portfolio_return <- n/100
  new_bvec <- c(1,expected_portfolio_return,0,0,0,0,0)
  qp <- solve.QP(Dmat, dvec, Amat, new_bvec, meq = 1)
  port_return <- c(port_return, expected_portfolio_return)
  port_variance <- c(port_variance, qp$value)
}
efficient_frontier <- plot_ly(x = port_variance, 
                              y = port_return,
                              type = 'scatter',
                              mode = 'lines+markers')%>%
  layout(title = "Efficient Frontier", xaxis = list(title = "Portfolio Variance (Risk)"), yaxis = list(title = "Portfolio Return"))
efficient_frontier
```

***Output** (on 11/23/2022):*

![Efficient Frontier](https://user-images.githubusercontent.com/114312864/204077982-f8e57f02-0653-44ad-8521-76dcd4c92c39.png)

> The graph above illustrates the Efficient Frontier for the portfolio consisting of five securities, including AAPL, META, OXY, SE, and TSLA. There is no portfolio existing above the frontier, while all possible portfolios are laying on or under the line. As mentioned, the line represents the optimal portfolios, each providing the maximum expected annual return given a certain level of risk. The highest possible expected annual return is approximately 1.4, corresponding to 140% capital gain after one year. Yet, this portfolio also faces the highest level of risk, corresponding to a variance of nearly 8.5.  It would be better to accept the overall yearly return of 0.4, or 40%, to reduce the risk with the portfolio variance nearly approaching zero. 

