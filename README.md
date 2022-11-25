# Modern Portfolio Optimization 

Modern Portfolio Theory is a mathematical method for selecting a variety of investments such as stocks, bonds, or treasury bills in order to optimize the overall returns in a period of time for a given level of risk. This method is also known as the mean-variance analysis since it shall take into account the mean (expected return) and variance of each individual security to calculate the overall expected return and level of risk (variance) of the portfolio. The theory considers diversification as a key to portfolio management since it would help investors to reduce risks or/and increase overall return. 

Based on the Modern Portfolio Theory, this optimization programming with R shall calculate the minimum risk of the portfolio containing a set of securities for a given desirable overall return. The program is built to optimize a portfolio comprising five (5) different securities, including 
1. Apple Inc. (NASDAQ: AAPL), 
2. Meta Platforms, Inc. (NASDAQ: META),
3. Occidental Petroleum (NYSE: OXY),
4. Sea Limited (NYSE: SE),
5. Tesla, Inc. (NASDAQ: TSLA).

Raw market data is collected from the start of 2017 to the current date (as of 11/24/2022). 

<h2>Getting Started </h2>
<h3>Loading libraries </h3>

```
# Core
library(tidyverse)

# Finance analytics
library(tidyquant)

# Optimization Solving
library(quadprog)

# Visualization 
library(plotly)
```

<h3> Getting data </h3>

```
stock <- c("AAPL", "META", "OXY", "SE", "TSLA")
data <- tq_get(stock, get = "stock.prices", complete_cases = TRUE, from = "2017-01-01", to = TODAY())
returns_data <- data %>%
    select(symbol, date, adjusted) %>%
    group_by(symbol)%>%
    tq_transmute(adjusted, 
                 mutate_fun = periodReturn, 
                 period = "yearly", 
                 col_rename = "Yearly Return")
return_data
```

<h2> Securities Statistics </h2>
<h3> Individual security </h3>

Calculating Expected Return and Standard Deviation for every individual stock:

```
stats <- returns_data %>%
  summarise(
    Yearly_Expected_Return = mean(`Yearly Return`),
    Standard_Deviation = sd(`Yearly Return`)
  )
stats
```

Output (as of 11/24/2022): 

| | Symbol  | Yearly_Expected_Return | Standard_Deviation |
| ------------- | ------------- | ------------- | ------------- |
|1| AAPL  | 0.3902252 | 0.4312123 |
|2| META  | 0.1190854 | 0.4833443 |
|3| OXY  | 0.2059046 | 0.7461615 |
|4| SE  | 0.9239347 | 1.8775403 |
|5| TSLA  | 1.3687916 | 2.9921934 |

<h3> Covariance </h3>

The mathematical covariance of any pair of securities is given by:
$\sigma_{ij}$ = expected value of $[(X_{i} - \mu_{i})((X_{j} - \mu_{j})]$ where: 

* $\sigma_{ij}$ = the covariance between securities i and j,

* $\mu_{i}$ = expected return of security i,

* $\mu_{i}$ = expected return of security j.

Calculating Covariance for each pair of stocks:

```
returns_matrix <- tidyr::spread(returns_data, 
                                key = "symbol", 
                                value = "Yearly Return")
covariance_matrix <- cov(returns_matrix[,-1])
covariance_matrix
```

Output (as of 11/24/2022):

```
           AAPL       META        OXY         SE       TSLA
AAPL  0.1859441  0.1863867 -0.2273701  0.6862111  0.7179275
META  0.1863867  0.2336217 -0.2495221  0.5133700  0.4532178
OXY  -0.2273701 -0.2495221  0.5567570 -1.0224761 -1.2641770
SE    0.6862111  0.5133700 -1.0224761  3.5251577  4.5520224
TSLA  0.7179275  0.4532178 -1.2641770  4.5520224  8.9532214
```

<h2> Optimization Solving </h2>

In the Modern Portfolio Optimization, the objective is the minimum of the portfolio variance, given by the function:

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

In order to find the minimum risk (measured by the portfolio variance) given a desirable overall expected return, an advanced mathematical tool called quadratic programming is used.

The general quadratic programming problem:

min $(\frac{1}{2}x^TDx - d^Tx)$ subject to $A^Tx \geq b$

<h3> Quadratic Programming Setup </h3>

```
# Objective matrix
Dmat <- 2*covariance_matrix
#Objective vector
dvec <- c(0,0,0,0,0)
# Constraint Matrix
Amat <- t(matrix(c(1, as.numeric(stats[stats$symbol == "AAPL", "Yearly_Expected_Return"]),1,0,0,0,0,
                   1,as.numeric(stats[stats$symbol == "META", "Yearly_Expected_Return"]),0,1,0,0,0,
                   1,as.numeric(stats[stats$symbol == "OXY", "Yearly_Expected_Return"]),0,0,1,0,0,
                   1,as.numeric(stats[stats$symbol == "SE", "Yearly_Expected_Return"]),0,0,0,1,0,
                   1,as.numeric(stats[stats$symbol == "TSLA", "Yearly_Expected_Return"]),0,0,0,0,1),7,5))
# Require Return (Set at 0.5)
goal <- 0.5
# Right hand side
bvec <- c(1,goal,0,0,0,0,0)
```

<h3> Quadratic Programming Solving </h3>

```
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

Output (as of 11/24/2022):

```
      [,1]     [,2]   [,3]    [,4]     [,5]    
stock "AAPL"   "META" "OXY"   "SE"     "TSLA"  
      "0.2043" "0"    "0.502" "0.1912" "0.1024"
[1] 0.2599801
```

This output means, given the desirable overall expected return at 0.5 (~ 50% after one year), the portfolio would face the lowest risk at the variance of 0.26 if the portfolio consists of 20.4 % of AAPL, 0 % of META, 50.2 % of OXY, 19.1 % of SE, and 10.2 % of TSLA

<h2> Efficient Frontier Visualization </h2>

The Efficient Frontier is all the scatter points, each representing a given portfolio's expected return and its corresponding minimum variance. Any portfolio located on the Efficient Frontier would have a higher expected return than any other portfolio having the same variance. Similarly, any portfolio located on the Efficient Frontier would have a lower risk (lower variance) than any other portfolio having the same expected return. Investors, thus, should only consider portfolios laying on the Efficient Frontier. 

The Efficient Frontier is limited by the minimum and maximum expected returns. While the minimum expected return is determined by the lowest expected return of all securities, the maximum expected return is determined by the highest one. No portfolio has an expected return beyond this bound. 

```
port_return <- c()
port_variance <-c()
min_return_in_percent <- ceiling(min(stats$Yearly_Expected_Return)*100)
max_return_in_percent <- floor(max(stats$Yearly_Expected_Return)*100)
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

<h3> Efficient Frontier </h3>
