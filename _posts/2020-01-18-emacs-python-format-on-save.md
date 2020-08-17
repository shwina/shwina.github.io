---
layout: post
title: "Format-on-save with elpy in Emacs"
date: 2020-01-18
comments: false
---

This post is about configuring [Elpy](https://elpy.readthedocs.io/en/latest/)
and Emacs to auto-format Python buffers using [Black](https://github.com/psf/black)
for a specific project (i.e., using [Directory Variables][directory-variables]).


*Note: for a while, I used [firestarter][firestarter] to accomplish this,
but that approach seemed clunky for what I needed.
I mention it here as another useful tool for run-on-save in Emacs*

## Configuring the Elpy RPC virtualenv

When using Elpy, there are two virtualenvs (or alternatively, conda environments)
involved:

- The virtualenv containing Python packages needed by your project
  (we'll refer to this as the *working environment*).
- The Elpy [RPC virtualenv](https://elpy.readthedocs.io/en/latest/concepts.html#the-rpc-process), where packages used by Elpy (such as `jedi` and `black`) are installed.

Personally, I find it better practice to just use the working environment
as the RPC environment. This allows me to control the versions of formatters
like `black` on a per-project basis. But, it also means having to install
them in each of my working environments.

```elisp
;; init.el

;; Use the working environment as the RPC environment
(setq elpy-rpc-virtualenv-path 'current)
```

If you do this, you *must* activate the working environment in Emacs
at the start of each session.
You can do this with `M-x pyvenv-activate` and pointing to the appropriate directory.

At this point, you should run `M-x elpy-config` to make sure
Elpy has found the right virtualenv,
and also is able to find the `black` formatter.

## Create (or update) `.dir-locals.el`

The [`.dir-locals.el`][directory-variables] file allows you to define variables
local to a directory and its sub-directories.
This makes it possible to specialize your Emacs configuration for a project,
while keeping a more general configuration across projects.

Place the `.dir-locals.el` file in your project's top-level directory,
and add the following entry:

```elisp
((python-mode . ((eval . (add-hook 'before-save-hook #'elpy-black-fix-code nil t)))))
```

## Restart Emacs and auto-format away!

After a restart of Emacs, run `M-x pyvenv-activate` again,
and navigate to your project directory.
You may be warned by Emacs about the directory local variable
being possibly [unsafe][safe-file-variables].
Type `!` to silence this warning.

That's it! Python buffers in this project should now be formatted
when you save them:

Before save:

```python
import os

a = [1,   2, 3]
def func(x,y = 3):
    z= x+ y
    return z
```

After save:

```python
import os

a = [1, 2, 3]


def func(x, y=3):
    z = x + y
    return z
```

Note that Elpy will respect the Black configuration provided in
a `pyproject.toml` placed in the project root directory.


[directory-variables]: https://www.gnu.org/software/emacs/manual/html_node/emacs/Directory-Variables.html
[safe-file-variables]: https://www.gnu.org/software/emacs/manual/html_node/emacs/Safe-File-Variables.html#Safe-File-Variables
[firestarter]: https://github.com/wasamasa/firestarter
