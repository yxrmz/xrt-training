# xrt Training Environment

This repository provides a reproducible Python environment for running **xrt** examples using the **Pixi** package manager.

Pixi ensures installation of the same versions of Python and dependencies on **Linux**, **macOS**, and **Windows**.

## Disclaimer

This titorial uses **development versions** of **xrt**. While we do our best to ensure that everything works reliably during the training, absolute stability cannot be guaranteed.

The environment has been prepared to provide a consistent setup across different operating systems, but occasional issues may still occur due to platform differences or ongoing development.

---

# 1. Install Pixi

Skip this step if Pixi is already installed on your system.

Official installation instructions:
https://pixi.sh/latest/

---

## Linux / macOS

Open a terminal and run:

```bash
curl -fsSL https://pixi.sh/install.sh | bash
```

## Windows

Download official installer or\
Open powershell and run

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://pixi.sh/install.ps1 | iex"
```

To allow executing pixi scripts in vscode add an policy exception

```powershell
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
```

# 2. Download and unpack **xrt**

Download the archive with the source code from:

https://github.com/kklmn/xrt/archive/refs/heads/new_glow.zip

Unpack the archive into a location of your choice.

---

# 3. Install **xrt** in your IDE

## VS Code

1. Open **VS Code**.
2. Select **File → Open Folder**.
3. Open the folder where you unpacked **xrt** in the previous step.

Open a terminal inside VS Code:

```
Terminal → New Terminal
```

Install the environment:

```bash
pixi install -e gui
```

This will install all required dependencies and create the Pixi environment used by the project.

The installation may take a few minutes the first time.

---

## Spyder (WinPython)

If you are using **Spyder from WinPython**, you can run examples as is without installation.

---

The environment is now ready.
