# Buchwald–Hartwig HTE Yield Prediction

使用公开高通量实验数据，比较不同分子表示与机器学习模型的产率预测表现。

## 研究问题

1. 模型能否预测已知组分构成的新条件组合？
2. 模型能否预测训练中未出现过的添加剂？
3. Morgan 指纹是否比组分身份编码更有帮助？

## 获取数据

本仓库通过原始来源提供数据下载入口。

1. 打开下面的工作簿链接，点击 **Download raw file**。
2. 在项目根目录创建 `data/raw/` 文件夹。
3. 将下载的文件放到以下位置，保留文件名：

   `data/raw/Dreher_and_Doyle_input_data.xlsx`

分析使用工作簿中的 `FullCV_01` 工作表，共 3,955 条记录。

- [下载 Excel 工作簿](https://github.com/rxn4chemistry/rxn_yields/blob/master/data/Buchwald-Hartwig/Dreher_and_Doyle_input_data.xlsx)
- [原论文](https://doi.org/10.1126/science.aar5169)

完成数据下载和环境安装后，选择项目的 `.venv` 内核，
按 `01` 至 `07` 的 Notebook 顺序运行，每个文件从顶部执行。

## 数据

使用 `Dreher_and_Doyle_input_data.xlsx` 的 `FullCV_01` 工作表，共 3,955 条记录。

输入角色为芳基卤化物、配体、碱和添加剂，目标列为 `Output`，产率采用 0–100 的尺度。

四类组分分别有 15、4、3、22 种。按原始组分字符串检查，无缺失值和重复条件组合。当前表覆盖 3,960 种可能组合中的 3,955 种。

数据与研究来源见文末链接。

## 环境与运行

Python 版本见 `python_version.txt`，依赖版本见 `requirements.txt`。

在项目根目录的 PowerShell 终端执行：

```powershell
py -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
```

在 VS Code 中选择 `.venv` 作为 Notebook 内核，按以下文件顺序运行，每个文件从顶部开始执行。

| Notebook | 内容 |
|---|---|
| `01_data_exploration.ipynb` | 数据审计与产率分布 |
| `02_feature_engineering.ipynb` | 拆分与特征构建 |
| `03_model_training.ipynb` | 模型比较、RF 选参与保存 |
| `04_model_evaluation.ipynb` | 指标、误差图与失败案例 |
| `05_shap_analysis.ipynb` | SHAP 全局与局部解释 |
| `06_improvement_experiments.ipynb` | 案例诊断与指纹半径对照 |
| `07_project_review.ipynb` | 文件与环境检查 |

运行期间会更新对应的结果文件。原始数据保存在 `data/raw/`。

## 方法

- One-hot 分别编码四个角色，编码器仅在各自训练集上拟合。
- Morgan 指纹采用半径 2、每角色 2,048 位、启用手性，按四角色顺序拼接为 8,192 列。
- 比较均值基线、线性回归、Random Forest 和 XGBoost。
- 随机拆分固定 `random_state=42`，训练与测试各 2,768、1,187 条。
- 添加剂留出将按 SMILES 排序后的第一种添加剂全部用于测试，训练与测试各 3,775、180 条。
- 拆分清单见 `data/splits/split_manifest.csv`。
- 小范围选参针对 Morgan Random Forest，随机场景使用三折 KFold，添加剂场景使用三折 GroupKFold。

## 初始模型结果

MAE 单位为产率百分点。

| 模型 | 随机拆分 | 添加剂留出 |
|---|---:|---:|
| 均值基线 | 23.65 | 23.36 |
| One-hot + 线性回归 | 12.25 | 13.73 |
| One-hot + Random Forest | 6.03 | 12.20 |
| One-hot + XGBoost | 8.87 | 11.95 |
| Morgan + 线性回归 | 12.66 | 13.77 |
| Morgan + Random Forest | 4.69 | 8.37 |
| Morgan + XGBoost | 6.86 | 9.57 |

## 选定模型的评估

| 场景 | MAE | RMSE | R² |
|---|---:|---:|---:|
| 随机拆分 | 4.689 | 7.141 | 0.933 |
| 添加剂留出 | 8.398 | 12.130 | 0.811 |

随机场景选择不限树深度、叶节点最少样本数为 1 的配置。添加剂场景根据内部验证选择深度 15、叶节点最少样本数为 1 的配置，测试 MAE 为 8.40。

模型在高产率区间有低估倾向。添加剂留出中，实测至少 80% 的 21 条反应平均被低估约 17.97 个百分点。

## SHAP 与改进实验

SHAP 解释选定的添加剂留出 Random Forest，使用 `tree_path_dependent` 设置。全部贡献通过预测还原检查。

案例 2293 实测产率为 86.42%，预测为 42.23%。已查看同条件下其他添加剂的产率与结构，目前未确定其化学原因。

固定模型参数与验证划分，Morgan 半径 2 和 3 的平均分组验证 MAE 分别为 9.983、10.194，因此继续采用半径 2。

## 结果位置

- `reports/morgan_random_forest/` 保存两个选定模型、配置、交叉验证表和测试预测。
- `reports/evaluation/` 保存指标、误差分析表与图。
- `reports/shap/` 保存 SHAP 数值、图、案例和解释配置。
- `reports/improvement/` 保存案例匹配数据与半径对照实验。

## 适用范围与待完成检查

数据来自特定 HTE 体系，模型的预测范围受其组分与实验条件覆盖限制。外部未知添加剂评估目前只包含一种添加剂。

测试结果在探索过程中已被多次查看。后续应扩大添加剂留出评估，检查结论是否稳定。SHAP 提供模型归因，机理解释需要进一步证据。

文件整理已完成，01–07 已完成重启内核后的顺序复跑与结果核对。

## 来源

- Ahneman et al., *Science* (2018), *Predicting reaction performance in C–N cross-coupling using machine learning*. [论文](https://doi.org/10.1126/science.aar5169)
- [公开 Excel 工作簿](https://github.com/rxn4chemistry/rxn_yields/blob/master/data/Buchwald-Hartwig/Dreher_and_Doyle_input_data.xlsx)
- [作者代码与数据仓库](https://github.com/doylelab/rxnpredict)