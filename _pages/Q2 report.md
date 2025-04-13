# Q2 报告

## 1. 引言

​	本pyspark实现的生存分析基于 Telco 客户流失数据，旨在探究客户在订阅服务期间的存活概率及影响流失的关键因素。分析主要采用了三种方法：

* **Kaplan-Meier 生存曲线**：用于估计群体水平的生存概率，并进行单变量分组对比。
* **Cox 比例风险模型**：通过回归方法评估各协变量对客户流失风险的影响，提供风险比（Hazard Ratio）的量化结果，并检验模型的比例风险假设。
* **加速失效时间模型（AFT）**：采用对数-对数逻辑模型进一步解释客户生存时间的变化，并计算中位生存时间。

## 2. 数据载入与预处理

* **数据来源**：从 Telco 客户数据文件（CSV格式）加载数据。
* **数据结构**：包括客户基本信息、服务类型、合同类型、账单信息及流失标记。
* **预处理步骤**：
  * 读取数据后，利用 Spark DataFrame 进行数据清洗与转换。
  * 对重要变量（例如 `Churn` 列）进行类型转换（如布尔型或数值型），并根据业务需求构造衍生变量（如 `churnFlag`）。
  * 对某些分类变量进行 One-Hot 编码，以便后续模型能正确处理类别信息。

## 3. Kaplan-Meier 生存分析

### 3.1 模型构建

* 使用 Lifelines 库中的 `KaplanMeierFitter` 构建模型，传入客户“服务时长（tenure）”作为生存时间变量，`churnFlag` 作为事件指标（1 表示发生流失，0 表示未流失）。
* 模型拟合后的结果包括：
  * 生存函数估计值
  * 中位生存时间

### 3.2 结果展示

* 绘制整体生存曲线，并用浅蓝色区域表示置信区间，直观展示模型在不同时间点的生存概率变化。

> ![image-20250413210842257](C:\Users\86132\Desktop\Courses\Big Data Analysis Software and Application\Projects\KM Curve.png)

### 3.3 分组对比与 Log-Rank 检验

* 为不同组别（如性别、OnlineSecurity 等）绘制 Kaplan-Meier 曲线，观察各组别生存曲线的分离情况。
* 应用 Log-rank 检验评估各组别间是否存在统计显著的生存差异。
  * 如对于性别组别，若 p 值大于 0.05，则说明不同性别之间的生存曲线无显著差异；反之，对于 OnlineSecurity 变量，若 p 值远小于 0.05，则说明该变量对客户流失具有显著影响。

> ![image-20250413211005850](C:\Users\86132\Desktop\Courses\Big Data Analysis Software and Application\Projects\KM Curve of gender.png)

## 4. Cox 比例风险模型分析

### 4.1 模型拟合

* 利用 `CoxPHFitter` 对预处理后的数据（经过 One-Hot 编码）进行拟合，输入变量包括：Dependents、InternetService、OnlineBackup、TechSupport 等。
* 模型输出提供了各协变量的系数、标准误、风险比（exp(coef)）以及 p 值，帮助判断每个变量对流失风险的影响大小和显著性。

### 4.2 模型结果解释

* **风险比解读**：例如，对于变量 `InternetService_DSL`，若 coef 为 -0.22 对应 exp(coef)=0.80，说明相较于基准组，订阅 DSL 的客户流失风险降低约20%。
* 输出中还包含模型的对数似然、AIC 以及检验统计量（如 log-rank 检验结果），用于模型优度评估。

> **![image-20250413211314777](C:\Users\86132\AppData\Roaming\Typora\typora-user-images\image-20250413211314777.png)**

## 5. Cox 模型假设检验

### 5.1 检查比例风险假设

* 通过 `check_assumptions` 方法对 Cox 模型进行比例风险假设检验。
* 同时利用 Schoenfeld 残差图和 Log-log 生存曲线检验各变量是否满足比例风险假设。
  * 如果发现某些变量（如 InternetService_DSL, OnlineBackup_Yes, TechSupport_Yes） p 值较低，则说明这些变量可能不满足比例风险假设，需要采取 stratification 或其他调整方法。

> ![image-20250413211423402](C:\Users\86132\Desktop\Courses\Big Data Analysis Software and Application\Projects\Scaled S resi.png)

> ![image-20250413211519587](C:\Users\86132\Desktop\Courses\Big Data Analysis Software and Application\Projects\loglog KM Curve of dependents.png)

## 6. 加速失效时间（AFT）模型分析

### 6.1 模型构建

* 采用 `LogLogisticAFTFitter` 构建加速失效时间模型，对客户“服务时长”与流失事件进行拟合。
* 模型通过对数-对数逻辑分布描述客户生存时间，输出中包含各变量系数、中位生存时间（如 135.51 个月）等结果。

### 6.2 模型解读

* **中位生存时间**：模型计算得出的中位生存时间较 Kaplan-Meier 模型有所不同，反映出模型参数选择对生存估计的影响。
* 同样，输出结果中的风险因子（exp(coef)）与系数提供了各协变量对流失时间加速或延迟的影响程度。

> ![image-20250413211622865](C:\Users\86132\Desktop\Courses\Big Data Analysis Software and Application\Projects\aft plot.png)

## 7. 客户终身价值（CLV）与 Dashboard 展示

### 7.1 基于生存概率的 CLV 计算

* 利用 Cox 模型预测获得每个时间点的生存概率，结合每月固定利润计算平均预期利润。
* 运用净现值（NPV）方法，折现未来每月利润后得到累积 NPV，用于评估客户获取的最大可接受成本。

### 7.2 Dashboard 可视化

* 构建包含“合同月份”、“生存概率”、“月利润”、“平均预期月利润”、“NPV”及“累计 NPV”在内的数据表。
* 绘制累积 NPV 柱状图和生存概率曲线，为决策提供直观依据。

> ![image-20250413211853961](C:\Users\86132\Desktop\Courses\Big Data Analysis Software and Application\Projects\NPV barplot.png)

> ![image-20250413211929700](C:\Users\86132\Desktop\Courses\Big Data Analysis Software and Application\Projects\Survival Prob Curve.png)

## 8. 总结与思考

* **主要发现**：
  * Kaplan-Meier 生存曲线为我们提供了整体生存概率的直观展示，其中中位生存时间反映了客户流失的基本规律。
  * Cox 比例风险模型揭示了各关键变量对客户流失的显著影响，但需注意部分变量不满足比例风险假设，可能需要进一步调整模型或采用分层分析。
  * AFT 模型作为一种全参数模型，可以为预测客户生存时间提供另一种视角，其输出结果与 Cox 模型互为补充。
* **局限性与改进方向**：
  * 数据中某些变量类别较少，可能导致模型假设检验的敏感性较高；
  * 不同模型对中位生存时间和风险比的估计存在差异，后续应结合交叉验证和业务背景调整模型设置；
  * 建议未来整合更多客户行为数据，完善客户细分模型，以提高 CLV 计算的准确性。