# Predict an absorption spectrum with pretrained X-MACE

You got this page plus two model files. That's the whole kit. You do **not** get access to anyone else's cluster folders, scripts, or env. Build yours, put the models in **your** scratch, and paste the script from section 5.

No training. No SMILES. The models read atom types and xyz coordinates.

| File they sent you | What it is | What it returns |
| --- | --- | --- |
| `chromophores_SVP_0.001_actual.model` | energies | 6 numbers |
| `chromophores_SVP_0.001_actual_osc_40.model` | oscillator strengths | 5 numbers |

Those two files. Energy model and oscillator model. That's it.

---

## 1. What an XYZ file is

Plain text. Line 1 = number of atoms. Line 2 = comment (anything). Then one line per atom: element, x, y, z in **Angstrom**.

To get the **same numbers** as section 5, save this as `molecule.xyz`. After that you can swap in your own geometries; same format.

```text
30
unsubstituted core
C        3.51512623       1.29020101       1.06830322
O        2.27745446       0.59723507       0.96940213
C        1.56698882       0.49135074      -0.23118639
O        2.04917717       0.96181966      -1.29912545
C        0.25137741      -0.26836544      -0.26214714
C        0.53156459      -1.72159205      -0.57667313
O        1.65201430      -2.17484282      -0.76715056
C       -0.68733971      -2.57840131      -0.63300920
C       -1.35663365      -2.50484764       0.72438194
C       -1.24762415      -1.41843457       1.51518305
C       -0.50948812      -0.20430528       1.05683738
C       -0.79318346       0.97690285       1.63869895
C       -0.52253840       2.29501950       1.01169733
C       -0.64727515       2.55714307      -0.30472521
C       -1.07388095       1.56712605      -1.32548856
C       -0.73245244       0.26449729      -1.29753042
C       -1.58327489      -0.71067864      -2.04167076
C       -1.58637897      -2.01915461      -1.71699873
H        3.87752106       1.24096861       2.11548553
H        3.37718106       2.35509196       0.78453407
H        4.26923517       0.81839529       0.40339584
H       -0.42360381      -3.63243537      -0.86755903
H       -1.94935370      -3.34632707       1.06046696
H       -1.76527885      -1.40981713       2.46701686
H       -1.31659948       0.98954468       2.58720909
H       -0.33625504       3.12798526       1.67762256
H       -0.55499307       3.58683708      -0.62598723
H       -1.75923809       1.92020643      -2.08669634
H       -2.26309228      -0.36019712      -2.80915927
H       -2.25915606      -2.69092551      -2.23512748
```

30 atoms, formula C15H12O3. The models know H, C, N, O, F, P, S, Cl, Br, I. Something else and the script should quit.

Only have SMILES? Build a 3D geometry first (UFF, xTB, Gaussian, whatever) and write XYZ. X-MACE will not parse a SMILES string.

---

## 2. What the two models output

Two calculators, same atoms.

**Energy model (6 slots)**

```text
energy[0]  =  S0   ground state
energy[1]  =  S1
energy[2]  =  S2
energy[3]  =  S3
energy[4]  =  S4
energy[5]  =  S5
```

Those are total energies. The gap you plot is

```text
ΔE(Sk)  =  energy[k] − energy[0]     for k = 1..5
```

**Oscillator model (5 slots)**

*f* was trained into the same `energy` field. The code still says `results["energy"]`, but those five numbers are oscillator strengths.

```text
osc[0]  pairs with  S1
osc[1]  pairs with  S2
osc[2]  pairs with  S3
osc[3]  pairs with  S4
osc[4]  pairs with  S5
```

Clip tiny negative *f* to 0. Wavelength: `nm = 1239.84193 / ΔE(eV)`. That's the stick spectrum.

---

## 3. From sticks to a curve

Lab UV-vis is broad, so we smear each stick with a Gaussian, FWHM = 0.30 eV:

```text
σ  =  0.30 / 2.35482
I(E)  =  Σ_k  f_k  exp( −½  ((E − ΔE_k) / σ)² )
```

Grid: 3.0 to 6.2 eV, 321 points. Then divide by the max so the peak is 1. Shape only, not a cross section.

---

## 4. Make your own env on Explorer

Explorer already has conda. Don't install conda. Check the docs just in case.

### 4.1 Create the env

```bash
conda create -n xmace python=3.11 -y
conda activate xmace
python --version
```

You want `3.11.something`. If you see 3.13, trash it and redo.

New terminal? Activate again:

```bash
conda activate xmace
```

### 4.2 Install the stack

Torch first, then the X-MACE deps, then plotting. Stay in `(xmace)`.

```bash
pip install torch
pip install e3nn==0.5.1 numpy opt_einsum ase torch-ema prettytable matscipy pandas
pip install matplotlib
```

No matplotlib = numbers in the terminal, no `spectrum.png`.

### 4.3 Clone X-MACE into *your* scratch

Not `pip install mace-torch`. That's a different package and Python will happily import the wrong `mace`.

```bash
mkdir -p /scratch/$USER/src
cd /scratch/$USER/src
git clone https://github.com/rhyan10/X-MACE.git
cd X-MACE
git checkout X-MACE_socs
```

Don't `pip install .`. Don't `pip install -e .`. Just export the folder:

```bash
export PYTHONPATH=/scratch/$USER/src/X-MACE
export TORCH_FORCE_NO_WEIGHTS_ONLY_LOAD=1
export MPLBACKEND=Agg
export PYTHONNOUSERSITE=1
export PYTHONUNBUFFERED=1
```

`MPLBACKEND=Agg` draws a PNG with no screen. The torch line stops new PyTorch from being picky about the checkpoint. Put the same four lines in your `sbatch`.

Check:

```bash
python -c "import numpy, ase, torch, matplotlib; from mace.calculators import MACECalculator; import mace; print('ok', mace.__file__)"
```

`mace.__file__` has to show `/scratch/$USER/src/X-MACE/...`. If it says `site-packages`, you grabbed PyPI MACE. `pip uninstall mace-torch` and try again.

One XYZ on the login node is usually fine. A pile of molecules: `sbatch` on `gpu`. Don't `pip install` inside a GPU job.

### 4.4 Park the two models you were sent

They are not on pip. Copy the files you received into your scratch, any folder you like. Example:

```bash
mkdir -p /scratch/$USER/models
# move/copy the two .model files you were sent into that folder
ls /scratch/$USER/models
```

You should see something like:

```text
chromophores_SVP_0.001_actual.model
chromophores_SVP_0.001_actual_osc_40.model
```

---

## 5. Save the script (it's in this page, nowhere else)


```bash
cat > /scratch/$USER/src/predict_one_spectrum.py << 'EOF'
#!/usr/bin/env python3
from pathlib import Path
import argparse
import sys

import numpy as np
import torch
from ase.io import read
from mace.calculators import MACECalculator
import matplotlib.pyplot as plt

HC_NM = 1239.84193
FWHM_EV = 0.30
SIGMA_EV = FWHM_EV / 2.35482
E_MIN = 3.0
E_MAX = 6.2
N_POINTS = 321
ALLOWED = {"H", "C", "N", "O", "F", "P", "S", "Cl", "Br", "I"}


def parse_args():
    p = argparse.ArgumentParser(description="XYZ in, X-MACE spectrum out.")
    p.add_argument("xyz", help="geometry, for example molecule.xyz")
    p.add_argument("--energy", default="chromophores_SVP_0.001_actual.model")
    p.add_argument("--osc", default="chromophores_SVP_0.001_actual_osc_40.model")
    p.add_argument("--out", default="", help="output folder; default = next to the XYZ")
    return p.parse_args()


def load_xyz(path):
    atoms = read(str(path))
    extra = sorted(set(atoms.get_chemical_symbols()) - ALLOWED)
    if extra:
        sys.exit("unknown elements: " + ", ".join(extra))
    return atoms


def load_calculators(energy_path, osc_path):
    energy_path = Path(energy_path)
    osc_path = Path(osc_path)
    if not energy_path.is_file():
        sys.exit("energy model not found: " + str(energy_path))
    if not osc_path.is_file():
        sys.exit("oscillator model not found: " + str(osc_path))
    if "auto" in energy_path.name.lower() or "auto" in osc_path.name.lower():
        sys.exit("that filename has 'auto' in it. use the actual files.")
    device = "cuda" if torch.cuda.is_available() else "cpu"
    print("device:", device, flush=True)
    print("energy model:", energy_path, flush=True)
    print("oscillator model:", osc_path, flush=True)
    energy_calc = MACECalculator(model_paths=str(energy_path), device=device, n_energies=6)
    osc_calc = MACECalculator(model_paths=str(osc_path), device=device, n_energies=5)
    return energy_calc, osc_calc


def raw_vector(calc, atoms):
    calc.calculate(atoms)
    return np.asarray(calc.results["energy"], dtype=float).ravel()


def sticks_from_models(energy_calc, osc_calc, atoms):
    energy = raw_vector(energy_calc, atoms)
    osc = raw_vector(osc_calc, atoms)
    if energy.size != 6:
        sys.exit("energy model returned %d numbers, expected 6" % energy.size)
    if osc.size != 5:
        sys.exit("oscillator model returned %d numbers, expected 5" % osc.size)
    delta_e = energy[1:6] - float(energy[0])
    f_use = np.maximum(osc, 0.0)
    rows = []
    for i in range(5):
        ev = float(delta_e[i])
        nm = (HC_NM / ev) if ev > 0 else float("nan")
        rows.append({"state": i + 1, "eV": ev, "f": float(f_use[i]), "f_raw": float(osc[i]), "nm": nm})
    return rows


def broaden(rows):
    e_grid = np.linspace(E_MIN, E_MAX, N_POINTS)
    y = np.zeros_like(e_grid)
    for r in rows:
        if r["eV"] > 0 and r["f"] > 0:
            y += r["f"] * np.exp(-0.5 * ((e_grid - r["eV"]) / SIGMA_EV) ** 2)
    ymax = float(np.max(y))
    y_norm = y / ymax if ymax > 0 else y.copy()
    nm = np.where(e_grid > 0, HC_NM / e_grid, np.nan)
    return e_grid, nm, y, y_norm


def write_tables(out_dir, rows, e_grid, nm, y, y_norm):
    sticks = out_dir / "sticks.csv"
    spec = out_dir / "spectrum.csv"
    with sticks.open("w", encoding="utf-8") as fh:
        fh.write("state,eV,nm,f,f_raw\n")
        for r in rows:
            fh.write("%d,%.6f,%.2f,%.6f,%.6f\n" % (r["state"], r["eV"], r["nm"], r["f"], r["f_raw"]))
    with spec.open("w", encoding="utf-8") as fh:
        fh.write("eV,nm,intensity,intensity_norm\n")
        for a, b, c, d in zip(e_grid, nm, y, y_norm):
            fh.write("%.6f,%.4f,%.8f,%.8f\n" % (a, b, c, d))
    print("wrote", sticks, flush=True)
    print("wrote", spec, flush=True)


def plot_spectrum(out_dir, rows, e_grid, nm, y_norm):
    fig, axes = plt.subplots(1, 2, figsize=(10.5, 4.0))
    ax = axes[0]
    ax.plot(e_grid, y_norm, color="#1f4e79", lw=2.0)
    for r in rows:
        if r["eV"] >= E_MIN and r["eV"] <= E_MAX and r["f"] > 0:
            h = r["f"] / max(max(x["f"] for x in rows), 1e-12)
            ax.vlines(r["eV"], 0.0, h, colors="#c44e52", lw=1.4)
            ax.text(r["eV"], h + 0.03, "S%d" % r["state"], ha="center", fontsize=8)
    ax.set_xlabel("Energy (eV)")
    ax.set_ylabel("Intensity (normalized)")
    ax.set_xlim(E_MIN, E_MAX)
    ax.set_ylim(0, 1.15)
    ax.set_title("X-MACE absorption")
    ax = axes[1]
    ax.plot(nm, y_norm, color="#1f4e79", lw=2.0)
    ax.set_xlabel("Wavelength (nm)")
    ax.set_ylabel("Intensity (normalized)")
    ax.set_xlim(HC_NM / E_MAX, HC_NM / E_MIN)
    ax.set_ylim(0, 1.15)
    ax.set_title("Same curve vs nm")
    fig.tight_layout()
    png = out_dir / "spectrum.png"
    fig.savefig(png, dpi=160)
    plt.close(fig)
    print("wrote", png, flush=True)


def main():
    args = parse_args()
    xyz = Path(args.xyz)
    if not xyz.is_file():
        sys.exit("XYZ not found: " + str(xyz))
    out_dir = Path(args.out) if args.out else xyz.resolve().parent
    out_dir.mkdir(parents=True, exist_ok=True)
    print("reading", xyz, flush=True)
    atoms = load_xyz(xyz)
    print("atoms:", len(atoms), "formula:", atoms.get_chemical_formula(), flush=True)
    energy_calc, osc_calc = load_calculators(args.energy, args.osc)
    rows = sticks_from_models(energy_calc, osc_calc, atoms)
    print("", flush=True)
    print("state    eV      nm        f", flush=True)
    for r in rows:
        print("  S%-2d  %6.3f  %7.1f  %7.4f" % (r["state"], r["eV"], r["nm"], r["f"]), flush=True)
    e_grid, nm, y, y_norm = broaden(rows)
    write_tables(out_dir, rows, e_grid, nm, y, y_norm)
    plot_spectrum(out_dir, rows, e_grid, nm, y_norm)
    print("done", flush=True)


if __name__ == "__main__":
    main()
EOF
```

Then `conda activate xmace`, the `export`s from 4.3, and run it. Use the `molecule.xyz` from section 1 if you want the table below. After that, swap in your own XYZ.

```bash
cd /scratch/$USER/wherever-your-xyz-is

python /scratch/$USER/src/predict_one_spectrum.py molecule.xyz \
  --energy /scratch/$USER/models/chromophores_SVP_0.001_actual.model \
  --osc    /scratch/$USER/models/chromophores_SVP_0.001_actual_osc_40.model
```

Those `--energy` / `--osc` paths are wherever **you** put the two models. For the table below, `molecule.xyz` has to be the 30-atom file from section 1.

You should get this (S2 is 0 because the raw *f* came out slightly negative and the script clips it):

```text
reading molecule.xyz
atoms: 30 formula: C15H12O3
device: cpu
energy model: .../chromophores_SVP_0.001_actual.model
oscillator model: .../chromophores_SVP_0.001_actual_osc_40.model

state    eV      nm        f
  S1    3.941    314.6   0.1876
  S2    4.430    279.9   0.0000
  S3    4.867    254.7   0.2322
  S4    5.184    239.2   0.1617
  S5    5.434    228.2   0.1931
wrote sticks.csv
wrote spectrum.csv
wrote spectrum.png
done
```

`sticks.csv` is those five lines. `spectrum.csv` is the smooth curve. `spectrum.png` is eV on the left, nm on the right.

---

## 6. What that script is doing (slow version)

Same ideas as section 5, in pieces, if you want to read it.

### 6.1 Open the two models

```python
device = "cuda" if torch.cuda.is_available() else "cpu"
energy_calc = MACECalculator(model_paths="chromophores_SVP_0.001_actual.model", device=device, n_energies=6)
osc_calc = MACECalculator(model_paths="chromophores_SVP_0.001_actual_osc_40.model", device=device, n_energies=5)
```

`n_energies=6` and `n_energies=5` have to stay like that.

### 6.2 Pair the sticks

```python
def raw_vector(calc, atoms):
    calc.calculate(atoms)
    return np.asarray(calc.results["energy"], dtype=float).ravel()

energy = raw_vector(energy_calc, atoms)   # length 6
osc = raw_vector(osc_calc, atoms)         # length 5
delta_e = energy[1:6] - float(energy[0])
f = np.maximum(osc, 0.0)

for i in range(5):
    ev = float(delta_e[i])
    print("S%d  %.3f eV  %.1f nm  f=%.4f" % (i + 1, ev, 1239.84193 / ev, float(f[i])))
```

### 6.3 Broaden and plot

```python
e_grid = np.linspace(3.0, 6.2, 321)
y = np.zeros_like(e_grid)
for ev, fi in zip(delta_e, f):
    if ev > 0 and fi > 0:
        y += fi * np.exp(-0.5 * ((e_grid - ev) / SIGMA_EV) ** 2)
y = y / y.max()
plt.plot(e_grid, y)
plt.xlabel("Energy (eV)")
plt.savefig("spectrum.png")
```

Versus nm: `plt.plot(1239.84193 / e_grid, y)`.

---

## 7. Many molecules

Your XYZ files, your folder. Example: first 20 `*.xyz` in one directory.

```bash
ROOT=/scratch/$USER/my_xyz_folder
EN=/scratch/$USER/models/chromophores_SVP_0.001_actual.model
OSC=/scratch/$USER/models/chromophores_SVP_0.001_actual_osc_40.model
PY=/scratch/$USER/src/predict_one_spectrum.py

i=0
for xyz in $ROOT/*.xyz; do
  i=$((i + 1))
  [ $i -gt 20 ] && break
  python "$PY" "$xyz" --energy "$EN" --osc "$OSC"
done
```

Reload 20 times is slow. For hundreds, load the two calculators once in Python and loop there.
