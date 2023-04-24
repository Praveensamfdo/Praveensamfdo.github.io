---
layout: post
title: "Using Nginx and Gunicorn to run a Django server"
date: 2023-04-24 11:45:00 +0000
tags: servers
description: "An Nginx front facing server with a Gunicorn backend serving a Django server"
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
