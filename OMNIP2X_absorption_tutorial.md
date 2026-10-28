# Predict an absorption spectrum with MLatom / OMNI-P2x

You got this page. That's the kit. You do **not** get access to anyone else's cluster folders, scripts, or env. Build yours. OMNI-P2x lives inside MLatom — there is no extra `.model` file to copy. First `predict` downloads the weights to your account.

No training. No SMILES. The model reads atom types and xyz coordinates.

| What you use | What it is | What it returns |
| --- | --- | --- |
| `ml.omnip2x()` | one calculator, inside MLatom | ΔE and *f*, already paired |

That's it. One object, two arrays.

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

30 atoms, formula C15H12O3.

Only have SMILES? Build a 3D geometry first (UFF, xTB, Gaussian, whatever) and write XYZ. OMNI-P2x will not parse a SMILES string.

---

## 2. What the model outputs

One calculator, same atoms.

**Energies** — already gaps, not total energies. They come out in **hartree**. Multiply by `ml.constants.hartree2eV`.

```text
excitation_energies[0]  =  ΔE(S1)
excitation_energies[1]  =  ΔE(S2)
excitation_energies[2]  =  ΔE(S3)
...
```

`nstates=11` means ground + 10 excited states. The array you plot is the 10 ΔE values (S1–S10). You do **not** subtract a ground-state total energy.

**Oscillator strengths** — same length, same order.

```text
oscillator_strengths[0]  pairs with  S1
oscillator_strengths[1]  pairs with  S2
oscillator_strengths[2]  pairs with  S3
...
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

Same cluster, **your** account. Don't point at someone else's conda env. You won't own it and it dies when they move stuff.

Explorer already has conda. Don't install conda.

### 4.1 Create the env

```bash
conda create -n mlatom python=3.11 -y
conda activate mlatom
python --version
```

You want `3.11.something`. MLatom wants 3.9–3.12. If you see 3.13, trash it and redo.

New terminal? Activate again:

```bash
conda activate mlatom
```

### 4.2 Install the stack

```bash
pip install --upgrade mlatom
pip install matplotlib
```

That pulls torch and the rest. No matplotlib = numbers in the terminal, no `spectrum.png`.

### 4.3 Exports and a sanity check

```bash
export MPLBACKEND=Agg
export PYTHONUNBUFFERED=1
export PYTHONNOUSERSITE=1
```

`MPLBACKEND=Agg` draws a PNG with no screen. Put the same three lines in your `sbatch`.

Check:

```bash
python -c "import mlatom as ml, matplotlib; print('ok', ml.__version__, ml.omnip2x)"
```

You want a version ≥ 3.22 so `ml.omnip2x` exists.

One XYZ on the login node often gets `Killed`. A pile of molecules: `sbatch` on `gpu`. Don't `pip install` inside a GPU job.

### 4.4 No extra model files

pip does not leave a `.model` next to your XYZ. First `omni.predict(...)` downloads OMNI-P2x into **your** home/cache. Let it. Don't copy a neighbor's cache.

---

## 5. Save the script (it's in this page, nowhere else)

Nobody is sending you a `.py` from another account. Paste this into a file on **your** scratch.

```bash
mkdir -p /scratch/$USER/src
cat > /scratch/$USER/src/predict_omni_spectrum.py << 'EOF'
#!/usr/bin/env python3
from pathlib import Path
from collections import Counter
import argparse
import sys

import numpy as np
import matplotlib.pyplot as plt
import mlatom as ml

HC_NM = 1239.84193
FWHM_EV = 0.30
SIGMA_EV = FWHM_EV / 2.35482
E_MIN = 3.0
E_MAX = 6.2
N_POINTS = 321
NSTATES = 11


def hill_formula(xyz):
    lines = Path(xyz).read_text(encoding="utf-8").splitlines()
    n = int(lines[0])
    counts = Counter(lines[i].split()[0] for i in range(2, 2 + n))
    order = ["C", "H"] + sorted(el for el in counts if el not in {"C", "H"})
    bits = []
    for el in order:
        k = counts.get(el, 0)
        if k:
            bits.append(el if k == 1 else f"{el}{k}")
    return n, "".join(bits)


def parse_args():
    p = argparse.ArgumentParser(description="XYZ in, OMNI-P2x spectrum out.")
    p.add_argument("xyz", help="geometry, for example molecule.xyz")
    p.add_argument("--nstates", type=int, default=NSTATES)
    p.add_argument("--out", default="", help="output folder; default = next to the XYZ")
    return p.parse_args()


def sticks_from_omni(xyz, nstates):
    mol = ml.data.molecule.from_xyz_file(str(xyz))
    omni = ml.omnip2x(warnings=False)
    omni.predict(molecule=mol, nstates=nstates)
    ev = np.asarray(mol.excitation_energies, dtype=float) * ml.constants.hartree2eV
    osc = np.asarray(mol.oscillator_strengths, dtype=float)
    f = np.maximum(osc, 0.0)
    rows = []
    for i in range(len(ev)):
        e = float(ev[i])
        rows.append(
            {
                "state": i + 1,
                "eV": e,
                "f": float(f[i]),
                "f_raw": float(osc[i]),
                "nm": (HC_NM / e) if e > 0 else float("nan"),
            }
        )
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
    ax.set_title("OMNI-P2x absorption")
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
    natom, formula = hill_formula(xyz)
    print("atoms:", natom, "formula:", formula, flush=True)
    rows = sticks_from_omni(xyz, args.nstates)
    print("nstates requested", args.nstates, "sticks", len(rows), flush=True)
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

Then `conda activate mlatom`, the `export`s from 4.3, and run it. Use the `molecule.xyz` from section 1 if you want the table below. After that, swap in your own XYZ. Do this on a GPU job, not the login node.

```bash
cd /scratch/$USER/wherever-your-xyz-is

python /scratch/$USER/src/predict_omni_spectrum.py molecule.xyz --nstates 11
```

`molecule.xyz` has to be the 30-atom file from section 1 for the table below.

You should get this:

```text
reading molecule.xyz
atoms: 30 formula: C15H12O3
nstates requested 11 sticks 10
state    eV      nm        f
  S1    (see sticks.csv after your GPU run)
  ...
wrote sticks.csv
wrote spectrum.csv
wrote spectrum.png
done
```

`nstates=11` → ten sticks (S1–S10). `sticks.csv` is those lines. `spectrum.csv` is the smooth curve. `spectrum.png` is eV on the left, nm on the right.

---

## 6. What that script is doing (slow version)

Same ideas as section 5, in pieces, if you want to read it.

### 6.1 Open the model

```python
import mlatom as ml
omni = ml.omnip2x(warnings=False)
mol = ml.data.molecule.from_xyz_file("molecule.xyz")
omni.predict(molecule=mol, nstates=11)
```

`nstates=11` has to stay like that if you want ten excitations.

### 6.2 Pair the sticks

```python
import numpy as np
ev = np.asarray(mol.excitation_energies, dtype=float) * ml.constants.hartree2eV
f = np.maximum(np.asarray(mol.oscillator_strengths, dtype=float), 0.0)

for i in range(len(ev)):
    print("S%d  %.3f eV  %.1f nm  f=%.4f" % (i + 1, ev[i], 1239.84193 / ev[i], float(f[i])))
```

Already ΔE. No `energy[k] - energy[0]`.

### 6.3 Broaden and plot

```python
e_grid = np.linspace(3.0, 6.2, 321)
y = np.zeros_like(e_grid)
for evi, fi in zip(ev, f):
    if evi > 0 and fi > 0:
        y += fi * np.exp(-0.5 * ((e_grid - evi) / SIGMA_EV) ** 2)
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
PY=/scratch/$USER/src/predict_omni_spectrum.py

i=0
for xyz in $ROOT/*.xyz; do
  i=$((i + 1))
  [ $i -gt 20 ] && break
  python "$PY" "$xyz" --nstates 11
done
```

Reload 20 times is slow. For hundreds, load `ml.omnip2x(warnings=False)` once in Python and loop there.
