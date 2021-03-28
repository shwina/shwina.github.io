---
layout: post
title: "Writing Your First Emacs Package"
---

I decided to write my first Emacs package today.
Here are the steps to do that.
This post is about how to install the package just for yourself.
In a future post, I'll show you how to uplaod the package so other people can use it.

# Set up a local package archive

```elisp
(add-to-list 'package-archives ("local" . "~/emacs-packages"))

(require 'package-x)
```

# Write your first pacakge

```elisp
;;; my-first-package.el --- My First Emacs Package

;; Author: Ashwin Srinath
:; Version: 0.1

(defun my-first-function nil "Prints 'hello!' to the echo area."
    (message "%s" "hello!"))

;;; my-first-package.el ends here
```

# Upload your package to the local archive

```elisp
(setq package-archive-upload-base "~/emacs-packages")
```

Should create "~/emacs-packages/my-first-package-0.1.el".

```
M-x package-upload-file
```

Install the package locally:

```
M-x package-refresh-contents
M-x package-install
```
