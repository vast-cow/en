---
title: "**Steps to install yaml-mode in Emacs**"
description: "This article explains the procedure for updating a GPG public key when a signature error occurs, the location of Emacs configuration files, and how to add the NonGNU ELPA repository."
pubDatetime: 2026-11-27T17:12:00+09:00
---
## **1. Updating the GPG public key (if signature errors occur)**

Perform this procedure when errors like the following occur.

```
Failed to verify signature archive-contents.sig:
No public key for 645357D2883A0966
```

In Emacs, press `M-:`, enter the following code, and press `RET`.

```
(progn
  (require 'package)
  (let ((package-check-signature nil)
        (package-archives
         '(("gnu" . "https://elpa.gnu.org/packages/"))))
    (package-refresh-contents)
    (package-install 'gnu-elpa-keyring-update)))
```

This updates the GNU ELPA public key. Signature verification is disabled only during this process and returns to its original setting after the process completes.

If the OS's package management feature can update the public key, that method is safer.

## **2. Check the location of the Emacs configuration file**

Run the following in Emacs.

```
C-h v user-init-file RET
```

The meaning of each operation is as follows.

| Operation         | Meaning                                          |
| ---------------- | ------------------------------------------------ |
| `C-h`            | Invokes the help feature                          |
| `v`              | Displays a variable's value and description (describe-variable) |
| `user-init-file` | Variable indicating the configuration file loaded at startup    |
| `RET`            | Confirms with the Enter key                       |

In the current environment, the following result was displayed.

```
Its value is "/home/naoto-aoki/.emacs"
```

Therefore, the configuration file is `~/.emacs`.

The Emacs configuration file varies depending on the environment.

| Candidate configuration files        |
| -------------------------           |
| `~/.emacs`                |
| `~/.emacs.el`             |
| `~/.emacs.d/init.el`      |
| `~/.config/emacs/init.el` |

By checking `user-init-file`, you can determine which configuration file is actually being used.

## **3. Add the NonGNU ELPA repository**

Open the configuration file.

```
C-x C-f ~/.emacs RET
```

Move to the end of the file.

```
M->
```

Append the following code.

```
;; NonGNU ELPA
(require 'package)

(add-to-list 'package-archives
             '("nongnu" . "https://elpa.nongnu.org/nongnu/") t)
```

This adds NonGNU ELPA as a package source.

## **4. Save the configuration file**

```
C-x C-s
```

## **5. Apply the configuration**

```
M-x eval-buffer RET
```

This executes the Emacs Lisp code in the currently open configuration file. There is no need to restart Emacs.

## **6. Update the package list**

```
M-x package-refresh-contents RET
```

This retrieves the list of installable packages from the registered repositories.

## **7. Install yaml-mode**

```
M-x package-install RET yaml-mode RET
```

## **8. Verify the operation**

Open a YAML file.

```
C-x C-f test.yaml RET
```

For example, enter the following content.

```
server:
  host: localhost
  port: 8080

database:
  name: example
  enabled: true
```

If `YAML` is displayed in the mode line at the bottom of the screen and syntax highlighting (color coding of syntax) is enabled, the installation is successful.

From then on, opening `.yaml` or `.yml` files will automatically enable `yaml-mode`.

## **Key operation reference**

| Key operation | Function                          |
| --------- | -------------------------------- |
| `C-h v`   | Displays a variable's value and description       |
| `C-h m`   | Displays a description of the current mode        |
| `C-x C-f` | Open or create a file               |
| `C-x C-s` | Save the file                    |
| `M-<`     | Move to the beginning of the file       |
| `M->`     | Move to the end of the file         |
| `M-:`     | Execute an Emacs Lisp expression |
| `M-x`     | Execute a command                |
| `C-g`     | Cancel an operation              |
| `RET`     | Enter key                       |
| `C`       | Ctrl key                        |
| `M`       | Meta key (usually the Alt key)   |

## **Reference links**

- [yaml-mode — GitHub](https://github.com/yoshiki/yaml-mode?utm_source=chatgpt.com)
- [NonGNU ELPA](https://elpa.nongnu.org/?utm_source=chatgpt.com)
- [GNU ELPA Keyring Update](https://elpa.gnu.org/packages/gnu-elpa-keyring-update.html?utm_source=chatgpt.com)
