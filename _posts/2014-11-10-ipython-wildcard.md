---
layout: post
title: "Wildcard completion in IPython"
---

I wanted to look up
all the top-level, grid related functions
available in NumPy.
On the IPython shell,
on a whim,
I typed:

```
import numpy as np
np.*grid*?
```

not knowing what to expect.
It did just what I wanted:

```
np.meshgrid
np.mgrid
np.ogrid
```

That made my day.
Thanks, IPython!
