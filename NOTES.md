# Aave V3 Notes  
## Terminology  
- Reserve Size: Total amount of that token that is currently in Aave's pool  
- Available Liquidity: Amount of that token that is currently available for borrowing  
- Utilization Rate: (Reserve Size - Available Liquidity / Reserve Size) * 100  
- APR: Annual Percentage Return is the annual interest rate without compound  
- APY: Annual Percentage Yield is the annual interest rate with compound  
$$
  APR = R\times{N} (Periodic\ interest\ rate\ \times\ Number\ of\ periods\ in\ a\ year)  
$$

$$
  APY = (1 + R)^{N} - 1  
$$
- Max LTV: LTV stands for Loan-To-Value. It's the max percentage of the lent token's value that you can borrow. For example if Max LTV is 63%, and you lent 100 DAI, you can borrow upto 63 dollars worth of assets.  
- Liquidation Threshold: Percentage at which your collateral can be liquidated.  
- Liquidation Penalty: The penalty that is paid from your collateral if your collateral is liquidated.  
- Reserve Factor: Basically a percentage of the interest you've paid will go to the protocol.  
- Isolation Mode: If a user deposits an isolated asset as collateral, they cannot deposit any other asset as collateral (but they can deposit to generate yield). Also, the user can only borrow some of the assets approved by Aave Governance for that isolated asset.  
- E-mode: Also known as efficiency mode allows the users to maximize their borrowing power if the collateral and the borrowed asset are correlated in price.  
- Index: In the contracts, the `index` term is used for cumulative rates from time of contract deployment  

## Market Forces  
### Supply and Demand  
$$
  {Utilization\ rate} = \frac{debt}{supply + debt}
$$
where `debt` is the amount borrowed plus the interest that needs to be paid and `supply` is the amount of tokens currently in Aave. It is always between 0 and 1.    
$$
{Supply\ IR} = {Borrow\ IR}\times{Utilization\ Rate}\times{(1 - Protocol\ Fee\ Rate)}
$$  
So if protocol fee rate = 0 and utilization rate = 1, then supply IR = borrow IR  
  
### Example:  
UR = 50% = 0.5  
Borrow IR = 7% = 0.07  
Protocol fee rate = 2% = 0.02  
Therefore, 
$$
  {Supply\ IR} = {0.07}\times{0.5}\times{(1 - 0.02)} = 0.0343 = 3.43\%  
$$
  
Each reserve has `aTokenAddress` and `variableDebtTokenAddress`  
- aTokens are the tokens you receive when you supply tokens to the reserve  
- variableDebtTokens are tokens that you receive when you borrow tokens from the reserve  
Both tokens are rebase tokens, their balances change even if you don't do anything. aTokens increase as you receive interest, debt tokens increase as the amount of interest you have to repay increases.  
  
### How to calculate interest on debt  
If user borrows `x` from timestamp `k` to `n`:  
$$
	{Debt} = x \prod_{i=k}^{n-1} (1 + r_i)
  \\
  {R(n)} = \prod_{i=0}^{n} (1 + r_i)
  \\
  {Debt} = x \frac{\prod_{i=0}^{n-1} (1 + r_i)}{\prod_{i=0}^{k-1} (1 + r_i)} = \frac{R(n-1)}{R(k-1)}
$$
So if user doesn't repay at `n` and borrows more at `l` till `m`, to handle that scenario in a contract efficiently, we do:  
$$
  {Debt}\ +=\ \frac{{amount}\times{1e18}}{R(l)}  
  \\
  {calculateDebtWithInterest}\ =\ \frac{{Debt}\times{R(m)}}{1e18}
$$  
Similarly, if user partially repaid his debt at `l` and wants to calculate debt with interest at `m`, we do:  
$$
  {Debt}\ -=\ \frac{{amount}\times{1e18}}{R(l)}  
  \\
  {calculateDebtWithInterest}\ =\ \frac{{Debt}\times{R(m)}}{1e18}
$$  
  
### Linear Interest Rate on Supply  
Assume user deposits 100 tokens at `time=0`, the interest stays the same at time 1 and 2, so the interest grows linearly, the interest updates at 3 and stays the same through 4 and user withdraws at 5. So the formula looks something like:  
$$
  100(1\ +\ r_1\ +\ r_2)(1\ +\ r_3\ +\ r_4)
  \\
  =\ 100(1\ +\ s_1)(1\ +\ s_2)\ =\ 100\frac{S(2)}{S(0)}
$$
Looks a lot like the equation for debt. We just need to modify the rate accumulator function to multiply the sum of the linear interests instead of multiplying the rate per second like we do in debt  
$$
  {S(n)} = \prod_{i=0}^{n} (1 + s_i)
$$
Each of s<sub>i</sub> here is the cumulation of the linear interests between updates  
