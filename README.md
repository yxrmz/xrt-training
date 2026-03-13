# xrt Training Environment

This repository provides a reproducible Python environment for running **xrt** examples using the **Pixi** package manager.

Pixi ensures installation of the same versions of Python and dependencies on **Linux**, **macOS**, and **Windows**.

## Disclaimer

This titorial uses **development versions** of **xrt**. While we do our best to ensure that everything works reliably during the training, absolute stability cannot be guaranteed.

The environment has been prepared to provide a consistent setup across different operating systems, but occasional issues may still occur due to platform differences or ongoing development.

---

# 1. Download and unpack **xrt**

Download the archive with the source code from:

https://github.com/kklmn/xrt/archive/refs/heads/new_glow.zip

Unpack the archive into a location of your choice.

---

# 2. Set up the environment

The project requires Python and several scientific libraries.  
Two installation options are provided:

- **Pixi (recommended)** – reproducible environment
- **Conda (alternative)** – manual environment setup

---

## 2.1 Pixi (recommended)

Pixi provides a reproducible environment across Linux, macOS, and Windows.

### Install Pixi

Official installation instructions:

https://pixi.sh/latest/

### Linux / macOS

```bash
curl -fsSL https://pixi.sh/install.sh | bash
```

Restart the terminal afterwards.

### Windows (PowerShell)

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://pixi.sh/install.ps1 | iex"
```

Restart the terminal or VS Code afterwards.

### Verify installation

```bash
pixi --version
```

### Install the environment

Open a terminal in the **xrt** folder and run:

```bash
pixi install -e gui
```

This installs all required dependencies and prepares the environment.

---

## 2.2 Conda (alternative)

If Pixi is not available, the environment can be created manually using Conda.

Create a new environment:

```bash
conda create -n xrt311 python=3.11
```

Activate the environment:

```bash
conda activate xrt311
```

Install required packages:

```bash
conda install numpy scipy matplotlib pyopencl pyopengl freetype-py qtpy pyqt pyqtwebengine
```

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

To allow executing pixi scripts in vscode on Windows add an policy exception

```powershell
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
```

---

## Spyder (WinPython)

If you are using **Spyder from WinPython**, you can run examples as is without installation.

---

The environment is now ready.

# 4. Calculating synchrotron sources

Navigate to the example folder:

```
xrt/examples/withRaycing/00_xRayCalculator
```

---

## Bending magnet example

Open the script:

```
calc_bm.py
```

Find the line:

```python
compareWithLegacyCode = True
```

Change it to:

```python
compareWithLegacyCode = False
```

(This avoids running the legacy implementation and speeds up the example.)

---

## Run the script

Run the script from your IDE or from the terminal.

Example using Pixi:

```bash
pixi run python calc_bm.py
```

After running the script, a plot window should appear showing the **bending magnet spectrum**.

