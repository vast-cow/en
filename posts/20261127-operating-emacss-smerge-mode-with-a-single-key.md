---
title: "Operate Emacs smerge-mode with a single key"
description: "Emacs configuration that assigns conflict resolution operations to a single key and displays help in the mode line."
pubDatetime: 2026-11-27T17:58:00+09:00
---
Emacs comes standard with `smerge-mode` for resolving Git conflicts.

Since the operations during conflict resolution are somewhat limited, I've made it possible to operate with single keys without using modifier keys.

## Settings

```elisp
(with-eval-after-load 'smerge-mode
  ;; Automatically exit after resolving conflicts
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

## Key bindings

| Key | Operation |
|---|---|
| `n` | Move to next conflict |
| `p` | Move to previous conflict |
| `u` | Keep upper change |
| `l` | Keep lower change |
| `a` | Keep both changes |
| `q` | Exit smerge-mode |

Since the key bindings are defined in `smerge-mode-map`, they apply only while the mode is active.

Additionally, I modified `minor-mode-alist` to display key binding help in the mode line.

```
SMerge [n]Next [p]Prev [u]Upper [l]Lower [a]All [q]Quit
```

## Adjusting conflict marker display

In my environment, when using a light theme, the conflict markers displayed black text on a black background, which was hard to read.

Therefore, I removed the foreground and background color specifications for `smerge-markers` and made it inherit the `default` face. The colors are left to the theme, and the conflict markers are emphasized by making them bold.

This makes the display usable with both light and dark themes.

## Automatic exit after resolving conflicts

Since `smerge-auto-leave` is enabled, when the last conflict is resolved with an smerge command, the mode exits automatically.

The key bindings and help display are also automatically removed, returning to normal editing state. If you need to type text midway, you can also exit with `q`.

By simply utilizing Emacs's minor mode mechanism, I was able to create an operation environment specialized for conflict resolution without depending on additional packages.
