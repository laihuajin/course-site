# 使用SDK完成离线评估

![](assets/langfuse-dataset-experiment-evaluation-workflow.png)

先删除UI评估器

```python
from ai_eval.benchmarks.experiment import run_experiment

run_experiment("主Agent回归实验", "pre-build")
```
