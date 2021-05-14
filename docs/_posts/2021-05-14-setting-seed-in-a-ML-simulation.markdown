---
layout: post
title: "Setting seed in a ML simulation"
date: 2021-05-14 01:12:00 +0000
tags: security encryption
description: "A code snippet for setting seed in a ML python script"
---
Use the following code snippet to set the seed in a python machine learning script.

```python
seed = 0       
torch.manual_seed(seed)
torch.cuda.manual_seed_all(seed)
torch.cuda.manual_seed(seed)
np.random.seed(seed)
random.seed(seed)
torch.backends.cudnn.deterministic=True
torch.backends.cudnn.benchmarks=False
os.environ['PYTHONHASHSEED'] = str(seed)
```

# Tips to make a code deterministic
- If *try-except* blocks behave differently for the same seed in different simulation runs, then the results may tend to differ (e.g. instances where different simulation runs with the same seed raise different errors in *try*).
