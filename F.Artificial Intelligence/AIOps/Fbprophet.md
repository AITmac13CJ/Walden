# 时间序列模型Prophet

> Prophet遵循sklearn库建模的API。

**输入和输出**

1. 输入已知的时间序列的时间戳和相应的值。
2. 输入需要预测的时间序列的长度。
3. 输出未来的时间序列走势。
4. 输出结果可以提供必要的统计指标，包括拟合曲线，上界和下界。

**参数**

​	ds：时间戳, pandas日期格式，YYYY-MM-DD或者YYYY-MM-DD HH:MM:SS

​	y：时间序列的取值，必须是数值。

通过Prophet可以计算出：

​	yhat:	表示时间序列的预测值。

​	yhat_lower：表示预测值的下界。

​	yhat_upper：表示预测值的上界。

**算法模型**
$$
y(t)=g(t)+s(t)+h(t)+\epsilon(t)
$$
​	g(t)：growth(增长趋势)，表示趋势项，它表示时间序列在非周期上面的变化趋势。

​	s(t)：seasonality(季节趋势)，表示周期项，或者成为季节项，一般是以周或者年为单位。

​	h(t)：holidays(节假日对季节的)，表示时间序列中那些潜在的具有非固定周期的节假日对预测值造成的影响。

​	&epsilon;(t)：即误差项或者称为剩余项，表示模型未预测到的波动， &epsilon;(t)服从高斯分布。

> Prophet 算法就是通过拟合这几项，然后最后把它们累加起来就得到了时间序列的预测值。

**适用场景**

	1.	有至少几个月(至少是一年)的每小时、每天或每周观察的历史数据。
	2.	有多种人类规模级别的较强的季节性趋势：每周的一些天或者每年的一些天。
	3.	有事先知道的以不定期的间隔发生的重要节假日。
	4.	缺失的历史数据或者较大的异常数据的数量在合理范围内。
	5.	有历史趋势的变化。
	6.	对于数据的非线性增长的趋势有一个自然极限或饱和状态。

**趋势项模型g(t)**

> 趋势项有两个重要的函数，一个是基于逻辑回归函数的(非线性增长)，一个是基于分段线性函数的(线性增长)。

1. 基于逻辑回归的趋势项
   $$
   g(t)=\frac{C(t)}{1+exp(-(k+\alpha(t)^T\delta))·(t-(m+(\alpha(t)^T\gamma)))},
   \\
   a(t)=(a_1(t),...,a_s(t))^T,\delta=(\delta_1,...,\delta_S)^T,\gamma=(\gamma_1,...,\gamma_S)^T
   $$
   C(t)表示承载量：

   - 它是一个随时间变化的函数，限定了所能增长的最大值。
   - 在使用Prophet的`growth='logistic'`的时候，需要提前设置好C(t)的取值才行。

   k表示增长率：

   - 在现实的时间序列中，趋势的走势肯定不会一直保持不变，在某些特定时候或者有着某种潜在的周期曲线会发生变化，模型定义了增长率k发生变化时对应的点，将其称作changepoints。
   - 在Prophet中是需要设置变点的位置的，而每一段的趋势和走势也是会根据变点的情况而改变的。可以通过人工指定的方式指定变点的位置，也可以通过算法来自动选择。在默认的函数中，Prophet会选择n_changpoints=25个变点，然后设置变点的范围是前80%，也就是时间序列的前80%的区间内会设置变点。之后还要看一些边界条件是否合理，例如时间序列的点数是否少于n_changepoint等。如果边界条件符合，那么变点的位置就是均匀分布的。

   m表示偏移量：

   - 当增长率k调整后，每个changpoints点对应的偏移量m也应相应调整以连接每个分段的最后一个时间点，表达式如下
     $$
     \gamma_j=(s_j-m-\sum_{e<j}\gamma e)·(1-\frac{k+\sum_{e<j}\delta e}{k+\sum_{e\leqslant j}\delta e})
     $$
     

2. 基于分段线性函数的趋势项
   $$
   g(t)=(k+\alpha(t)^T\delta)·t+(m+\alpha(t)^T\gamma)
   $$
   k表示增长率。

   &delta;表示增长率的变化量。

   m表示偏移量。

   分段线性函数与逻辑回归函数的最大区别就是&gamma;的设置不一样。在分段线性函数中：
   $$
   \gamma=(\gamma_1,...,\gamma_S)^T,\gamma_j=-S_j\delta_j
   $$
   在分段线性函数中，不需要capacity这个指标，因此`m=Prophet()`这个函数默认使用`growth='linear'`这个增长函数，也可以写作`m=Prophet(growth='linear');`。

3. 变点的选择

   在Prophet算法中最重要的三个指标：

   - changepoint_range：变点的位置。指的是百分比，默认的函数中设置为`changpoint_range=0.8`。
   - n_changpoint：变点的个数。默认函数中是`n_changpoint=25`。
   - Changpoint_prior_scale：增长的变化率。

   > 默认的场景下，变点的选择是基于时间序列的前80%的历史数据，然后通过等分的方法找到25个变点，而变点的增长率是满足Laplace分布&delta;~j~~Laplace(0,0.05)的。因此当changpoint_prior_scale趋近于0的时候， &delta;~j~也是趋向于0的，此时的增长函数将变成全段的逻辑回归函数或者线性函数。

4. 对未来的预估
   - 从历史上长度为T的数据中，可以选择出 S 个变点，它们所对应的增长率的变化量是&delta;~j~~Laplace(0,&tau;)。
   - 此时我们需要预测未来，因此也需要设置相应的变点的位置,此时通过 Poisson 分布等概率分布方法找到新增的changepoint_ts_new 的位置。
   - 然后与changepoint_t 拼接在一起就得到了整段序列的 changepoint_ts。

**季节性趋势s(t)**

> 由于时间序列中有可能包含多种天，周，月，年等周期类型的季节性趋势，因此，傅里叶级数可以用来近似表达这个周期属性。	

- 使用傅立叶级数来模拟时间序列的周期性：假设P表示时间序列的周期，P=365.25 表示以年为周期，P=7 表示以周为周期。
- 它的傅立叶级数的形式都是：$s(t)=\sum_{n=1}^{N}(a_ncos(\frac{2\pi nt}{P}) + b_nsin(\frac{2\pi nt}{P}))$

- N表示希望在模型中使用的这种周期的个数，较大的N值可以拟合出更复杂的季节性函数，然而也会带来更多的过拟合问题。
- 按照经验值，对于以年为周期的序列(P=365.25)而言，N=10；对于以周为周期的序列(P=7)而言，N = 3。
- 当 N = 10 时，$X(t)=[cos(\frac{2\pi(1)t}{365.25}),...,sin(\frac{2\pi(10)t}{365.25})]$​。
- 当 N = 3 时，$X(t)=[cos(\frac{2\pi(1)t}{7}),...,sin(\frac{2\pi(3)t}{7})]$​​​。

因此时间序列的季节项就是：$s(t)=X(t)\beta$

其中，

- &beta;的初始化是&beta;~Normal(0,&sigma;^2^)。
- 这里的&sigma;是通过seasonality_prior_scale 来控制的，也就是说&sigma;=seasonality_prior_scale。这个值越大，表示季节的效应越明显；这个值越小，表示季节的效应越不明显。
- 在代码里面，seasonality_mode 也对应着两种模式，分别是加法和乘法，默认是加法的形式。

**节假日效应h(t)**

- 在现实环境中，节假日或者是一些大事件都会对时间序列造成很大影响，而且这些时间点往往不存在周期性，对这些点的分析是极其必要的。
- 在 Prophet 里面，收集了各个国家的特殊节假日。除了节假日之外，用户还可以根据自身的情况来设置必要的假期，例如双十一。
- 由于每个节假日或者某个已知的大事件对时间序列的影响程度不一样，例如春节，国庆节则是七天的假期，对于劳动节等假期来说则假日较短。
- 因此，节假日模型将不同节假日在不同时间点下的影响视作独立的模型，并且可以为不同的节假日设置不同的前后窗口值，表示该节假日会影响前后一段时间的时间序列。
- 对于第 i 个节假日来说，D~i~表示该节假日的前后一段时间。
- 为了表示节假日效应，需要一个相应的指示函数，同时需要一个参数来表示节假日的影响范围。
- 假设有 L 个节假日，那么节假日效应模型就是：$h(t)=Z(t)\kappa=\sum_{i=1}^{L}\kappa_i·1_{t\epsilon D_i}$​​，其中$Z(t)=(1_{t\epsilon D_1}),...,(1_{t\epsilon D_L)}$​​和$\kappa=(\kappa_1,...,\kappa_L)^T$​。

其中，

- &kappa;~Normal(0,&mu;^2^)并且该正态分布是受到 v = holidays_prior_scale 这个指标影响的。
- 默认值是 10，当值越大时，表示节假日对模型的影响越大；当值越小时，表示节假日对模型的效果越小。
- 该参数可自行调整。

**Prophet 使用时可设置的参数**

- growth：增长趋势模型。分为两种：”linear”与”logistic”，分别代表线性与非线性的增长，默认值：”linear”。
- Capacity：在增量函数是逻辑回归函数的时候，需要设置的容量值,表示非线性增长趋势中限定的最大值，预测值将在该点达到饱和。
- Change Points：可以通过 n_changepoints 和 changepoint_range 来进行等距的变点设置，也可以通过人工设置的方式来指定时间序列的变点，默认值：“None”。
- n_changepoints：用户指定潜在的”changepoint”的个数，默认值：25。
- changepoint_prior_scale：增长趋势模型的灵活度。调节”changepoint”选择的灵活度，值越大，选择的”changepoint”越多，从而使模型对历史数据的拟合程度变强，然而也增加了过拟合的风险。默认值：0.05。
- seasonality_prior_scale（seasonality模型中的）：调节季节性组件的强度。值越大，模型将适应更强的季节性波动，值越小，越抑制季节性波动，默认值：10.0。
- holidays_prior_scale（holidays模型中的）：调节节假日模型组件的强度。值越大，该节假日对模型的影响越大，值越小，节假日的影响越小，默认值：10.0。
- freq：数据中时间的统计单位（频率），默认为”D”，按天统计。
- periods：需要预测的未来时间的个数。例如按天统计的数据，想要预测未来一年时间内的情况，则需填写365。
- mcmc_samples：mcmc采样，用于获得预测未来的不确定性。若大于0，将做mcmc样本的全贝叶斯推理，如果为0，将做最大后验估计，默认值：0。
- interval_width：衡量未来时间内趋势改变的程度。表示预测未来时使用的趋势间隔出现的频率和幅度与历史数据的相似度，值越大越相似，默认值：0.80。当mcmc_samples = 0时，该参数仅用于增长趋势模型的改变程度，当mcmc_samples > 0时，该参数也包括了季节性趋势改变的程度。
- uncertainty_samples：用于估计未来时间的增长趋势间隔的仿真绘制数，默认值：1000。
