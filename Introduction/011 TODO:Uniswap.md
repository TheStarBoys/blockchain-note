# Uniswap

## 公式

**乘积常量 K**：
$$
K = reserve0 * reserve1
$$
**当 swap 发生时，给定 amountIn，计算 amountOut**：
$$
(reserveIn + amountIn) * (reserveOut - amountOut) = K
$$
将此等式与上面的等式结合，最终得到（不考虑任何服务费的情况）：
$$
amountOut = \frac{reserveOut}{\frac{reserveIn}{amountIn} + 1}
$$
Uniswap 协议将收取 0.3% 作为 LPs 的服务费（近似值）：
$$
amountOut = \frac{reserveOut}{\frac{reserveIn}{amountIn} * \frac{1000}{997} + 1}
$$
mintFee，如果费用开关打开，当 pair 发生 mint 或 burn 时，会产生的费用：
$$
mintFee = \frac{totalLiquidity * (K - KLast)}{5 * K + KLast}
$$