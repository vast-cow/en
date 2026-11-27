---
title: "Operate Emacs’s smerge-mode with a single key"
description: "An Emacs configuration that assigns conflict resolution operations to a single key and displays help in the mode line."
pubDatetime: 2026-11-27T17:58:00+09:00
---
Emacs comes with `smerge-mode` built in as a standard feature for resolving Git conflicts.

Since the operations for resolving conflicts are somewhat limited, I configured it so that it can be operated with single keys without using modifier keys.

## Settings

```elisp
(with-eval-after-load 'smerge-mode
  ;; Automatically exit after conflict resolution
  (setq smerge-auto-leave t)

  ;; Display of conflict markers
  (set-face-attribute 'smerge-markers nil
                      :foreground 'unspecified
                      :background 'unspecified
                      :inherit 'default
                      :weight 'bold
                      :extend t)

  ;; Key bindings
  (define-key smerge-mode-map (kbd "n") #'smerge-next)
  (define-key smerge-mode-map (kbd "p") #'smerge-prev)
  (define-key smerge-mode-map (kbd "u") #'smerge-keep-upper)
  (define-key smerge-mode-map (kbd "l") #'smerge-keep-lower)
  (define-key smerge-mode-map (kbd "a") #'smerge-keep-all)
  (define-key smerge-mode-map (kbd "q")
              (lambda ()
                (interactive)
                (smerge-mode -1)))

  ;; Display key binding help in the mode line
  (setf (alist-get 'smerge-mode minor-mode-alist)
        '(" SMerge [n]Next [p]Prev [u]Upper [l]Lower [a]All [q]Quit")))
```

## Key Bindings

| Key | Operation |
|---|---|
| `n` | Move to next conflict |
| `p` | Move to previous conflict |
| `u` | Keep upper change |
| `l` | Keep lower change |
| `a` | Keep both changes |
| `q` | Exit smerge-mode |

Since the key bindings are defined in `smerge-mode-map`, they are only applied while the mode is active.

Also, by changing `minor-mode-alist`, I made the key binding help display in the mode line.

```
SMerge [n]Next [p]Prev [u]Upper [l]Lower [a]All [q]Quit
```

For conflict markers as well, I only made them bold to fit in with the existing theme.

## Automatic Exit After Conflict Resolution

Since `smerge-auto-leave` is enabled, when the last conflict is resolved with an smerge command, the mode exits automatically.

The key bindings and help display are also automatically cleared, returning to the normal editing state. If you need to type text midway through, you can also exit with `q`.

By simply utilizing Emacs's minor mode mechanism, I was able to create an operation environment specialized for conflict resolution without depending on additional packages.
