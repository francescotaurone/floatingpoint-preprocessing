# Floating point preprocessing for IoT - Addendum

From the paper, and in the context of the "Addition method" presented there, we know that in floating point bigger numbers are generally less precise than smaller numbers, and that precision is a key element in calculating the recovery error. Therefore, we are led to think that larger A will result in larger recovery errors. This is generally true, but not always: it is only true when x_i + A  2^{E_U^A}. More details for this can be found in the addendum 
We here present one counter intuitive situations that explains a phenomenon mentioned in the paper, related to
Let's consider x_1 = 2.1534481 belongs to 2^1 region. Its precision is therefore 2^{1-23} = 2^{-22}.
Let's select 3 specific A values, in order to investigate their recovery errors by applying the steps in \autoref{fig:additionMethodCompressionDiagram}. The results are summarized in \autoref{tab:counterintuitiveResults}.

| # | x1        | A              | y1      | e                 |
|---|-----------|----------------|---------|-------------------|
| a | 2.1534481 | 6.0            | 2^{-20} | 0.0               |
| b | 2.1534481 | 6.00000047684 | 2^{-20} | 4.7683716*10^{-7} |
| c | 2.1534481 | 16.0000019073  | 2^{-19} | 0.0               |



We notice that although A_c leads to a y_1 region with lower precision, its resulting recovery error is lower than the one using A_b. This interesting result is due to how addition between two FP numbers works in IEEE 754, more specifically with the \textit{normalization} of the result.

Once the mantissas from the two numbers are summed together, we need to approximate anything that is not in the most significant 23 bits of x_1+ A. Supposing that A ≥ x_1, there are 2 possible scenarios:
- x_1 + A belongs to 2^{E_U^A}: therefore no normalization of the result is required (left column on figure);
- x_1 + A belongs to 2^{E_U^A + 1}: in this scenario, the comma of the resulting value needs to be shifted of 1 step, and therefore the LSB of A is going to affect the portion to be approximated out (right column in figure).



![alt text](https://github.com/francescotaurone/floatingpoint-preprocessing/blob/main/figures/floatingpoint-ApproximationCounterintuitive.png?raw=true)

Operation y_1 = x_1 + A might produce non-null approximation error G, R, S stand for guard, round and sticky bits. The red bits are the last bits of each FP number, cyan portion is what needs to be approximated when no shift in A occurs during the sum, whereas the orange portion highlights that a shift occurred as per the IEEE 754 standard, and therefore the approximation error depends also on the A bits. `@' represent any bit value.


By having part of A simplified during the sum, it is as if we were introducing an approximation error not due to numbers in the dataset, but rather on the selected A.
In order to avoid this unwanted behavior, we need to choose A so that this holds:

x_1 + A belongs to 2^{E_U^A}


It should be noted that the bound error ≤ P/2 holds even when the previous equation does not, as shown in this figure. 


![alt text](https://github.com/francescotaurone/floatingpoint-preprocessing/blob/main/figures/errorBoundForIncreasingA.png?raw=true)

Given x = 2.15355810 with varying A, we plot with black line the absolute value of the recovery error, with red line the error bound, with vertical dashed green vertical lines the 2^i numbers, namely the change of exponent regions, and with red vertical lines 2^i - x. The light red areas highlight where x_1 + A belongs to 2^{E_U^A} does not hold, namely where both x and A play a role in determining the recovery error


## On checks after multiplication 

In some cases, the multiplication y = M * x might output a weird result, that we can fix by performing an extra check. Let's see an example.
Consider x_1 = 17.2307701111. We want to reach y = 224.0 by performing the multiplication by 13. However,  y_1 = x_1 * 13 = 224.000015259, with mantissa 11000000000000000000001 having a one at the end. We might think that the x_1 we selected is just a tiny bit too big. Let's consider than the previous representable FP number, namely x_2 = 17.2307682037. With this number, the result is y_2 = x_2 * 13 = 223.9999866481, which has mantissa 10111111111111111111111.  This means that there is no number in 32 bits floating point that multiplied by 13 gives 224.0.

We might also notice that y_1 - y = y_2 - y = P, where P = 2^{7-23} = 2^{-16} is the precision of the number, namely the quantity multiplied with the last bit in the mantissa.

The reason for this behavior lies in the fact that we are using FP with round to nearest approximation approach. Therefore, if we want to keep using it while avoiding the unwanted behavior, we should check after the multiplication by M if we are P away from achieving the desired number of ending zeros, and if so adjust the result with the operations y = y_1 - P or y = y_2 + P.