---
title: "Launching an MSVC PowerShell"
description: "This example shows how to start a PowerShell session that is already configured for the Microsoft..."
pubDatetime: 2026-06-26T03:07:53.090Z
updatedDate: 2026-06-26T11:35:44.418Z
---

This example shows how to start a PowerShell session that is already configured for the Microsoft Visual C++ (MSVC) development environment.

Instead of opening the Visual Studio Developer PowerShell manually, the script automatically finds a suitable Visual Studio or Build Tools installation and starts an MSVC-enabled PowerShell session.

## Purpose

The script is useful when you want to:

* Start an MSVC development environment from a script.
* Use Visual Studio Build Tools without opening the Visual Studio IDE.
* Select a specific target or host architecture.
* Use the latest installed Visual Studio or Build Tools automatically.

## How It Works

The PowerShell script performs the following steps:

1. Parses command-line options.
2. Searches for a Visual Studio installation by using `vswhere.exe`.
3. Loads the Visual Studio Developer Shell module.
4. Starts the developer shell by calling `Enter-VsDevShell`.

If no suitable installation is found, the script prints an error message explaining what components are required.

## Command-Line Options

The script supports the following options:

| Option         | Description                                                                                                                                              |
| -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--arch`       | Target architecture (`x86`, `amd64`, `arm`, or `arm64`).                                                                                                 |
| `--host-arch`  | Host architecture.                                                                                                                                       |
| `--product`    | Visual Studio product to search for. The default is `Microsoft.VisualStudio.Product.BuildTools`. Use `*` to search all installed Visual Studio products. |
| `--prerelease` | Include prerelease Visual Studio installations.                                                                                                          |
| `--help`       | Display usage information.                                                                                                                               |

## Starting the Script

The following shell script starts the PowerShell script:

```sh
#!/bin/sh
exec /mnt/c/windows/system32/WindowsPowerShell/v1.0/powershell.exe \
    -NoLogo \
    -NoExit \
    -ExecutionPolicy Bypass \
    -File ./msvc-devshell.ps1
```

Running this script opens a PowerShell session with the MSVC development environment already configured. Because the session is started with `-NoExit`, the PowerShell window remains open after initialization, allowing you to run build tools such as `cl`, `link`, and `nmake`.

```cmd
powershell.exe -NoLogo -NoExit -ExecutionPolicy Bypass -File ./msvc-devshell.ps1
```

```powershell
$ErrorActionPreference = "Stop"

$Arch = "amd64"
$HostArch = "amd64"
$Product = "Microsoft.VisualStudio.Product.BuildTools"
$Prerelease = $false

function Show-Usage {
@"
Usage:
  msvc-pwsh.ps1 [options]

Options:
  --arch ARCH          Target architecture: x86, amd64, arm, arm64
  --host-arch ARCH     Host architecture: x86, amd64, arm, arm64
  --product PRODUCT    Visual Studio product id.
                       Default: Microsoft.VisualStudio.Product.BuildTools
                       Use "*" to search all Visual Studio products.
  --prerelease         Include prerelease Visual Studio instances.
  -h, --help           Show this help.
"@
}

$argv = @($args)

$i = 0
while ($i -lt $argv.Count) {
  switch ($argv[$i]) {
    { $_ -in @("-arch", "--arch") } {
      if ($i + 1 -ge $argv.Count) {
        [Console]::Error.WriteLine("missing value for $($argv[$i])")
        exit 2
      }
      $Arch = $argv[$i + 1]
      $i += 2
      continue
    }

    { $_ -in @("-host_arch", "--host-arch") } {
      if ($i + 1 -ge $argv.Count) {
        [Console]::Error.WriteLine("missing value for $($argv[$i])")
        exit 2
      }
      $HostArch = $argv[$i + 1]
      $i += 2
      continue
    }

    { $_ -in @("-product", "--product") } {
      if ($i + 1 -ge $argv.Count) {
        [Console]::Error.WriteLine("missing value for $($argv[$i])")
        exit 2
      }
      $Product = $argv[$i + 1]
      $i += 2
      continue
    }

    "--prerelease" {
      $Prerelease = $true
      $i += 1
      continue
    }

    { $_ -in @("-h", "--help") } {
      Show-Usage
      exit 0
    }

    default {
      [Console]::Error.WriteLine("unknown argument: $($argv[$i])")
      exit 2
    }
  }
}

$validArch = @("x86", "amd64", "arm", "arm64")

if ($Arch -notin $validArch) {
  [Console]::Error.WriteLine("invalid --arch: $Arch")
  exit 2
}

if ($HostArch -notin $validArch) {
  [Console]::Error.WriteLine("invalid --host-arch: $HostArch")
  exit 2
}

$programFilesX86 = ${Env:ProgramFiles(x86)}
if (-not $programFilesX86) {
  $programFilesX86 = "C:\Program Files (x86)"
}

$vswhere = Join-Path $programFilesX86 "Microsoft Visual Studio\Installer\vswhere.exe"

if (-not (Test-Path -LiteralPath $vswhere -PathType Leaf)) {
  [Console]::Error.WriteLine("vswhere.exe not found: $vswhere")
  exit 1
}

$vswhereArgs = @(
  "-latest",
  "-products", $Product,
  "-requires", "Microsoft.VisualStudio.Component.VC.Tools.x86.x64",
  "-property", "installationPath"
)

if ($Prerelease) {
  $vswhereArgs += "-prerelease"
}

$installationPath = & $vswhere @vswhereArgs |
  ForEach-Object { $_ -replace "`r$", "" } |
  Select-Object -First 1

if (-not $installationPath) {
  [Console]::Error.WriteLine(@"
Visual Studio / Build Tools with MSVC was not found.

Try:
  .\msvc-pwsh.ps1 --product "*"

Or install:
  - Visual Studio Build Tools
  - MSVC C++ build tools
  - Windows SDK
"@)
  exit 1
}

$devshellDll = Join-Path $installationPath "Common7\Tools\Microsoft.VisualStudio.DevShell.dll"

if (-not (Test-Path -LiteralPath $devshellDll -PathType Leaf)) {
  [Console]::Error.WriteLine("DevShell module not found: $devshellDll")
  exit 1
}

Set-Location $Env:USERPROFILE

Import-Module $devshellDll

Enter-VsDevShell `
  -VsInstallPath $installationPath `
  -SkipAutomaticLocation `
  -Arch $Arch `
  -HostArch $HostArch `
  -DevCmdArguments "-no_logo"
```
