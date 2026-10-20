---
title: "Downloading and extracting a whl file without using pip"
description: "curlやwgetでダウンロードし、unzipやPythonのzipfileで展開する手順を紹介。"
pubDatetime: 2026-10-20T12:44:00+09:00
---

To download and extract a `.whl` file without using `pip`, you can download it with `curl` or `wget` and extract it with `unzip` or Python's built-in `zipfile` library.

Since `.whl` files are essentially ZIP archives, `pip` is not required.

## 1. Download the `.whl` file

If obtaining it from PyPI, first access:

[https://pypi.org/project/requests/#files](https://pypi.org/project/requests/?utm_source=chatgpt.com#files)

Obtain the URL of the desired `.whl` file and download it.

```
curl -L -o package.whl "https://files.pythonhosted.org/.../package.whl"
```

Or:

```
wget -O package.whl "https://files.pythonhosted.org/.../package.whl"
```

## 2. Extract the `.whl` file

### Method A: Using `unzip`

```
mkdir extracted
unzip package.whl -d extracted/
```

### Method B: Using only the Python standard library

```
import zipfile
with zipfile.ZipFile("package.whl") as z:
    z.extractall("extracted")
```

This will extract the contents of the `.whl` file.

## 3. Automate the download and extraction process using the PyPI API

This can be achieved using only the Python standard library, without using `pip` or external commands.

```
import json
import urllib.request
import zipfile
from pathlib import Path

package = "requests"
version = "2.32.3"

# Obtain package information from the PyPI API
url = f"https://pypi.org/pypi/{package}/{version}/json"
with urllib.request.urlopen(url) as response:
    data = json.load(response)

# Obtain the wheel file
wheel = next(
    f for f in data["urls"]
    if f["filename"].endswith(".whl")
)
wheel_path = Path(wheel["filename"])

# Download
urllib.request.urlretrieve(wheel["url"], wheel_path)

# Extract
output_dir = Path("extracted")
output_dir.mkdir(exist_ok=True)
with zipfile.ZipFile(wheel_path) as z:
    z.extractall(output_dir)

print(f"Downloaded: {wheel_path}")
print(f"Extracted: {output_dir}")
```

## 4. Directory structure after extraction

For example, for `requests`:

```
extracted/
├── requests/
│   ├── __init__.py
│   ├── api.py
│   ├── sessions.py
│   └── ...
└── requests-2.32.3.dist-info/
    ├── METADATA
    ├── WHEEL
    └── RECORD
```

If you want to import the extracted files, you can do so as follows:

```
import sys
sys.path.insert(0, "extracted")
import requests
```

However, you will also need to install the libraries that `requests` depends on separately. Simply extracting the files will not automatically resolve dependencies.

Also, for wheel files that include C extensions, you need to choose a wheel that is compatible with your Python version, OS, and CPU architecture.
