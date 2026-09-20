---
title: "Installing bcache-tools from Oracle Linux 9 Repository on Rocky Linux 9"
description: "When running Rocky Linux 9, you may want to install a package that is only available in Oracle Linux..."
pubDatetime: 2026-01-19T12:51:51.204Z
---

When running Rocky Linux 9, you may want to install a package that is only available in Oracle Linux 9’s repositories — such as **bcache-tools**. However, enabling a whole external repository can cause compatibility issues if many packages are pulled in unintentionally. To avoid this, you can configure **`includepkgs`** so that only the desired package is allowed from the Oracle repository.

This article shows a minimal setup to safely install `bcache-tools` on Rocky Linux 9 using Oracle’s repositories.



## Step 1: Import Oracle’s GPG Key

First, download and import the Oracle GPG key:

```bash
sudo rpm --import https://yum.oracle.com/RPM-GPG-KEY-oracle-ol9
```



## Step 2: Create a Repository File

Next, create a repo file that points to Oracle Linux 9’s BaseOS repository. Use `includepkgs` to allow only `bcache-tools`:

```bash
sudo tee /etc/yum.repos.d/ol9-oracle.repo > /dev/null <<'EOF'
[ol9-baseos]
name=Oracle Linux 9 BaseOS Latest ($basearch)
baseurl=https://yum.oracle.com/repo/OracleLinux/OL9/baseos/latest/$basearch/
gpgcheck=1
enabled=1
includepkgs=bcache-tools
EOF
```

**Key points:**

* `includepkgs=bcache-tools` ensures only that package can be installed from this repo.
* You can list multiple packages separated by commas, e.g. `includepkgs=pkg1,pkg2`.
* Using `enabled=1` is fine since the filter prevents other packages from being pulled in. If you prefer, set `enabled=0` and enable it only when needed with `--enablerepo`.



<!-- ## Step 3: Check Dependencies

Before installing, check dependencies to make sure Rocky Linux’s repos can satisfy them:

```bash
sudo dnf install -y dnf-plugins-core
sudo dnf repoquery --requires --resolve bcache-tools
```

If some dependencies are also only in Oracle’s repo, add them to `includepkgs`. -->



## Step 4: Install bcache-tools

Finally, install the package:

```bash
sudo dnf install bcache-tools
```

Thanks to `includepkgs`, only `bcache-tools` (and any explicitly listed dependencies) will be pulled from Oracle’s repo.



## Notes and Caveats

* `includepkgs` is a **per-repo filter** and does not affect other repositories.
* If `bcache-tools` depends on packages that Rocky does not provide, you must add those to the `includepkgs` list as well.
* Use `dnf --downloadonly` or `repoquery` to inspect what will be installed before proceeding.
* Remember that `bcache` also requires kernel support. Installing `bcache-tools` alone is not sufficient if your kernel does not have `bcache` enabled.



✅ With this method, you can safely use Oracle Linux’s repository on Rocky Linux to install just `bcache-tools` — without risking unintentional system-wide package replacements.
