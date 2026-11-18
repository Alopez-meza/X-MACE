# ADC(2)-SOS with TURBOMOLE

Automated workflow for running ADC(2)-SOS excited-state calculations on batches of molecules using [TURBOMOLE](https://www.turbomole.org/) on a Slurm cluster.

**Method:** ADC(2)-SOS / def2-SVP — 5 singlet roots, frozen core, spectrum + transition moments + excited-state gradients.

---

## Workflow

```mermaid
flowchart LR
    A["📂 xyz/*.xyz"]:::input --> B["stage_xyz.sh"]:::script
    B --> C["prepare.sh\n(x2t + define)"]:::script
    C --> D["submit_jobs.sh"]:::script
    D --> E["dscf → ricc2\n(per molecule)"]:::compute
    E --> F["ricc2.out\n(spectrum)"]:::output

    classDef input fill:#4a90d9,stroke:#2c5f8a,color:#fff
    classDef script fill:#6ab04c,stroke:#3d7a28,color:#fff
    classDef compute fill:#e67e22,stroke:#a35b16,color:#fff
    classDef output fill:#9b59b6,stroke:#6c3483,color:#fff
```

| Step | What runs | What it produces |
| --- | --- | --- |
| **Stage** | `bash stage_xyz.sh` | `mol01/molecule.xyz`, `mol02/molecule.xyz`, … |
| **Prepare** | `sbatch prepare.sh` | `coord`, `control`, `basis`, `auxbasis`, `mos` per molecule |
| **Submit** | `bash submit_jobs.sh [N]` | One Slurm job per molecule |
| **Compute** | `dscf` then `ricc2` | `dscf.out`, `ricc2.out`, `energy`, `exstates` |
| **Monitor** | `bash check_status.sh` | Progress summary |

---

## Prerequisites

- **TURBOMOLE** (tested with version <!-- TODO: add your version, e.g. 7.7 -->). The workflow uses `x2t`, `define`, `dscf`, and `ricc2`.
- **Slurm** workload manager on your cluster.
- **Python 3** (only for the optional `define_answers.txt` sanity check).
- Standard XYZ geometry files (ångström) for every molecule you want to calculate.

> [!IMPORTANT]
> **Not on the Lopez cluster?** Before running anything, edit the `TURBODIR` path in `prepare.sh` and `adc2.slurm`, and set your Slurm partition in `config.sh`. See [Configuration](#configuration).

---

## Quick start

```bash
# 1. Put your XYZ files into xyz/
mkdir -p xyz
cp /path/to/your/structures/*.xyz xyz/

# 2. Create one directory per molecule
bash stage_xyz.sh

# 3. Run define on a compute node (writes TURBOMOLE input files)
sbatch prepare.sh
# Wait until prepare_*.out shows: DONE ok=N fail=0

# 4. Submit a small test batch first
bash submit_jobs.sh 20
squeue -u "$USER"
bash check_status.sh

# 5. If the test batch looks good, submit everything
bash submit_jobs.sh
```

Each molecule gets its own Slurm job. `submit_jobs.sh` is idempotent — rerun it after a queue limit or failure and it skips finished molecules.

---

## Configuration

The scripts work out of the box for the Lopez group (partition `lopez`, TURBOMOLE at the group path). To run on a different cluster, there are two things to change:

**1. `TURBODIR` in `prepare.sh` and `adc2.slurm`.** Both scripts set the TURBOMOLE path in their body. Edit the `export TURBODIR=…` line in each file to point to your installation.

**2. Slurm partition and resources via `config.sh`.** `submit_jobs.sh` sources `config.sh` and passes its values as `sbatch` command-line flags, which override the `#SBATCH` defaults inside `adc2.slurm`:

For `prepare.sh`, which you submit directly with `sbatch`, edit the `#SBATCH -p` line in the file itself, or pass the partition on the command line: `sbatch -p yourpartition prepare.sh`.

> [!NOTE]
> The defaults (8 CPUs, 32 GB, 5-day walltime) work well for molecules up to ~50 heavy atoms with def2-SVP. Larger molecules may need more memory — see [Troubleshooting](#troubleshooting).

---

## Directory layout

Before running anything you need this:

```text
your_run/
├── config.sh               ← Slurm overrides (partition, CPUs, memory, walltime)
├── define_answers.txt       ← scripted input for TURBOMOLE define
├── stage_xyz.sh             ← xyz/*.xyz → <name>/molecule.xyz
├── prepare.sh               ← x2t + define (Slurm job, all molecules)
├── adc2.slurm               ← dscf + ricc2 (copied per molecule)
├── submit_jobs.sh           ← submit adc2.slurm per molecule
├── check_status.sh          ← count staged / prepared / finished
└── xyz/                     ← your input geometries
    ├── mol01.xyz
    ├── mol02.xyz
    └── ...
```

Each `.xyz` file stem becomes a molecule directory: `mol01.xyz` → `mol01/`.

**XYZ format** — standard ångström coordinates. Line 1 is the atom count, line 2 is a comment, then one atom per line:

```text
40
mol01
C        1.43366600       3.66672000       0.12074800
O        1.95266200       2.38359500       0.42909400
...
```

---

## How TURBOMOLE is used

Each molecule lives in its own directory with `control` (the calculation recipe) and `coord` (geometry in bohr). TURBOMOLE's `define` program is an interactive setup tool — this workflow feeds it a scripted answer file (`define_answers.txt`) so it runs non-interactively.

| Program | Stage | What it does |
| --- | --- | --- |
| `x2t` | prepare | Converts XYZ (ångström) → `coord` (bohr) |
| `define` | prepare | Interactive setup driven by `define_answers.txt`. Writes `control`, `basis`, `auxbasis`, `mos` |
| `dscf` | compute | Ground-state SCF (Hartree–Fock). Converges orbitals in `mos` |
| `ricc2` | compute | ADC(2)-SOS: 5 singlets, spectrum, transition moments, excited-state gradients |

**Parallelism:** each molecule runs as an 8-core SMP job (`PARA_ARCH=SMP`). Many molecules run as independent Slurm jobs — `submit_jobs.sh` submits one job per folder.

Slurm does not load your shell rc files. `prepare.sh` and `adc2.slurm` set `TURBODIR` themselves.

---

## Files

### `config.sh`

Slurm overrides for clusters other than Lopez. `submit_jobs.sh` sources this and passes the values as `sbatch` flags (command-line flags override `#SBATCH` directives inside the job script).

```bash
#!/bin/bash
# config.sh — Slurm overrides for the ADC(2)-SOS workflow.
#
# submit_jobs.sh sources this file and passes these values as
# sbatch command-line flags, which override the #SBATCH defaults
# inside adc2.slurm.
#
# For prepare.sh (submitted directly), edit its #SBATCH -p line
# or pass: sbatch -p yourpartition prepare.sh
#
# TURBODIR is set inside prepare.sh and adc2.slurm directly —
# edit those two files if your TURBOMOLE is at a different path.

# Partition for ADC(2) compute jobs.
SLURM_PARTITION="lopez"

# Resources per molecule.
SLURM_CPUS=8
SLURM_MEM="32G"
SLURM_TIME="5-00:00:00"
```

---

### `define_answers.txt`

Scripted keyboard input for TURBOMOLE's `define`. Configures: `c1` symmetry, def2-SVP basis, extended Hückel guess, charge 0, closed shell, 1500 MiB per core, ADC(2)-SOS with frozen core, 5 singlet roots, spectrum + excited-state gradients, and SCF damping.

> [!WARNING]
> The file **must start with two blank lines**. If those are missing, `define` receives answers in the wrong order and produces an incorrect `control` file. This is the most common setup error.

```text


a coord
*
no
b all def2-SVP
*
eht
y
0
y
cc
memory 1500
ricc2
adc(2)
sos
maxiter 5000
*
cbas
*
freeze
core
*
exci
irrep=a multiplicity=1 nexc=5
spectrum states=all operators=diplen,dipvel,angmom,qudlen
tmexc istates=all fstates=all operators=diplen,soc
xgrad states=all
*
resp
gradient
fop relaxed operators=diplen,qudlen
*
*
scf
damp
0.600
0.050
0.100

*
```

The sequence sets: `no` to symmetry → def2-SVP → EHT guess → charge `0` → closed shell (`y`) → 1500 MiB → ADC(2)-SOS → frozen core → 5 singlets (`multiplicity=1 nexc=5`) → spectrum + `xgrad` → relaxed dipole/quadrupole → SCF damping 0.600 / 0.050 / 0.100.

It matches a **neutral closed-shell singlet** with no extra symmetry prompts. If your molecules are charged or open-shell, run `define` interactively once, note which extra questions it asks, and adjust this file.

**Sanity check:**

```python
python3 -c "
from pathlib import Path
b = Path('define_answers.txt').read_bytes()
assert b.startswith(b'\n\na coord\n'), 'ERROR: need two blank lines before \"a coord\"'
assert b'def2-SVP' in b
print('define_answers.txt OK')
"
```

---

### `stage_xyz.sh`

Copies each `xyz/*.xyz` into its own directory as `<name>/molecule.xyz`.

```bash
#!/bin/bash
# Copy xyz/*.xyz into one folder per molecule.
set -euo pipefail
ROOT="$(cd "$(dirname "$0")" && pwd)"
cd "$ROOT"

if [[ ! -d xyz ]]; then
  echo "Need a folder named xyz/ next to this script."
  exit 1
fi

n=0
for f in xyz/*.xyz; do
  [[ -f "$f" ]] || { echo "No .xyz files in xyz/"; exit 1; }
  name=$(basename "$f" .xyz)
  mkdir -p "$name"
  cp "$f" "$name/molecule.xyz"
  n=$((n + 1))
done

echo "STAGED $n molecules"
```

---

### `prepare.sh`

Runs `x2t` and `define` in every staged molecule directory. Submit with `sbatch` so it runs on a compute node.

```bash
sbatch prepare.sh
# Check: tail -f prepare_*.out
# Done when you see: DONE ok=N fail=0
```

A folder is marked **OK** when `define.log` contains "define ended normally" and `control` has `adc(2)` with singlet multiplicity. On **FAIL**, partial TURBOMOLE files are removed and `define.log` is kept for debugging. Folders that already have `control` are **SKIP**ped — delete `control` to force re-preparation.

```bash
#!/bin/bash
#SBATCH -J adc2-define
#SBATCH -p lopez
#SBATCH -N 1
#SBATCH -n 1
#SBATCH -c 1
#SBATCH --mem=4G
#SBATCH -t 08:00:00
#SBATCH -o prepare_%j.out
#SBATCH -e prepare_%j.err

# Builds coord, control, basis, auxbasis and mos in every molecule folder.
# Reads define_answers.txt from this directory.

ROOT="$(cd "$(dirname "$0")" && pwd)"
cd "$ROOT"

export TURBODIR=/projects/lopez/share_from_Andres/SOFTWARE/Turbomole/TURBOMOLE
source "$TURBODIR/Config_turbo_env"
export PARA_ARCH=SMP
ulimit -s unlimited

ANSWERS="$ROOT/define_answers.txt"
ok=0
fail=0
skip=0

for dir in "$ROOT"/*/; do
  name=$(basename "$dir")
  [[ "$name" == "xyz" ]] && continue
  xyz="$dir/molecule.xyz"
  if [[ ! -f "$xyz" ]]; then
    continue
  fi
  if [[ -f "$dir/control" ]]; then
    echo "SKIP $name"
    skip=$((skip + 1))
    continue
  fi
  (
    cd "$dir"
    rm -f coord basis auxbasis mos define.log
    x2t molecule.xyz > coord
    define < "$ANSWERS" > define.log 2>&1
  )
  if grep -q "define ended normally" "$dir/define.log" \
     && grep -q "adc(2)" "$dir/control" \
     && grep -q "multiplicity=  1" "$dir/control" \
     && ! grep -q "multiplicity=  3" "$dir/control"; then
    echo "OK $name"
    ok=$((ok + 1))
  else
    echo "FAIL $name"
    rm -f "$dir/control" "$dir/basis" "$dir/auxbasis" "$dir/mos" "$dir/coord"
    fail=$((fail + 1))
  fi
done

echo "DONE ok=$ok fail=$fail skip=$skip"
```

---

### `adc2.slurm`

The per-molecule Slurm job. Runs `dscf` (ground-state SCF) followed by `ricc2` (ADC(2)-SOS excited states). `submit_jobs.sh` copies this into each molecule folder and submits it.

```bash
#!/bin/bash
#SBATCH -J adc2
#SBATCH -p lopez
#SBATCH -N 1
#SBATCH -n 1
#SBATCH -c 8
#SBATCH --mem=32G
#SBATCH -t 5-00:00:00
#SBATCH -o dscf-ricc2-%j.out
#SBATCH -e dscf-ricc2-%j.err

# ADC(2)-SOS single point. Submit from the molecule folder.

export TURBODIR=/projects/lopez/share_from_Andres/SOFTWARE/Turbomole/TURBOMOLE
source "$TURBODIR/Config_turbo_env"
export PARA_ARCH=SMP
export PARNODES=$SLURM_CPUS_PER_TASK
export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK
ulimit -s unlimited

dscf > dscf.out
ricc2 > ricc2.out
```

---

### `submit_jobs.sh`

Copies `adc2.slurm` into each prepared molecule folder and submits one Slurm job per molecule. Sources `config.sh` and passes its values as `sbatch` flags (overriding the `#SBATCH` defaults in `adc2.slurm`).

```bash
bash submit_jobs.sh 20    # test batch: first 20 prepared molecules
bash submit_jobs.sh       # all remaining molecules
```

Finished molecules (those with "ricc2 : all done" in `ricc2.out`) are skipped automatically, so you can safely rerun after a queue limit or partial failure. If Slurm returns `QOSMaxJobsPerUserLimit`, wait and run `submit_jobs.sh` again.

```bash
#!/bin/bash
# Copy adc2.slurm into each molecule folder and submit that copy.
# Optional first argument: maximum number of new jobs (e.g. 20).
# Run after prepare.sh has written control.
#
# Sources config.sh and passes its values as sbatch flags, which
# override the #SBATCH defaults inside adc2.slurm.

ROOT="$(cd "$(dirname "$0")" && pwd)"
cd "$ROOT"

# ── Load overrides from config.sh (if present) ──
if [[ -f "$ROOT/config.sh" ]]; then
  source "$ROOT/config.sh"
fi
SLURM_PARTITION="${SLURM_PARTITION:-lopez}"
SLURM_CPUS="${SLURM_CPUS:-8}"
SLURM_MEM="${SLURM_MEM:-32G}"
SLURM_TIME="${SLURM_TIME:-5-00:00:00}"

LIMIT="${1:-}"
if [[ -n "$LIMIT" && ! "$LIMIT" =~ ^[0-9]+$ ]]; then
  echo "Usage: $0 [max_jobs]"
  exit 1
fi

submitted=0
skipped=0

for dir in "$ROOT"/*/; do
  name=$(basename "$dir")
  [[ "$name" == "xyz" ]] && continue
  if [[ ! -f "$dir/molecule.xyz" ]]; then
    continue
  fi
  if [[ ! -f "$dir/control" ]]; then
    echo "NO CONTROL $name"
    continue
  fi
  if [[ -f "$dir/ricc2.out" ]] && grep -q "ricc2 : all done" "$dir/ricc2.out"; then
    echo "DONE $name"
    skipped=$((skipped + 1))
    continue
  fi
  if [[ -n "$LIMIT" && "$submitted" -ge "$LIMIT" ]]; then
    break
  fi
  cp "$ROOT/adc2.slurm" "$dir/adc2.slurm"
  echo "SUBMIT $name"
  sbatch --chdir="$dir" \
         --job-name="$name" \
         --partition="$SLURM_PARTITION" \
         --cpus-per-task="$SLURM_CPUS" \
         --mem="$SLURM_MEM" \
         --time="$SLURM_TIME" \
         "$dir/adc2.slurm"
  submitted=$((submitted + 1))
done

echo "submitted=$submitted already_done=$skipped"
```

---

### `check_status.sh`

Reports how many molecules are staged, prepared, finished with `dscf`, finished with `ricc2`, or flagged as errors.

```bash
#!/bin/bash
# Summarize ADC(2) progress in this directory.
ROOT="$(cd "$(dirname "$0")" && pwd)"
cd "$ROOT"

staged=0
prepared=0
dscf_ok=0
ricc2_ok=0
failed=0

for dir in "$ROOT"/*/; do
  name=$(basename "$dir")
  [[ "$name" == "xyz" ]] && continue
  [[ -f "$dir/molecule.xyz" ]] || continue
  staged=$((staged + 1))
  [[ -f "$dir/control" ]] || continue
  prepared=$((prepared + 1))
  if [[ -f "$dir/dscf.out" ]] && grep -q "dscf : all done" "$dir/dscf.out"; then
    dscf_ok=$((dscf_ok + 1))
  fi
  if [[ -f "$dir/ricc2.out" ]] && grep -q "ricc2 : all done" "$dir/ricc2.out"; then
    ricc2_ok=$((ricc2_ok + 1))
  elif [[ -f "$dir/dscf.out" ]] || [[ -f "$dir/ricc2.out" ]]; then
    if grep -Eqi "error|abnormally|killed" "$dir"/dscf.out "$dir"/ricc2.out 2>/dev/null; then
      echo "CHECK $name"
      failed=$((failed + 1))
    fi
  fi
done

echo "staged=$staged prepared=$prepared dscf_done=$dscf_ok ricc2_done=$ricc2_ok flagged=$failed"
```

---

## Output reference

### After `prepare.sh` (TURBOMOLE input)

```text
mol01/
├── molecule.xyz     ← your input (ångström)
├── coord            ← geometry in bohr (x2t)
├── control          ← calculation recipe (define)
├── basis            ← orbital basis (def2-SVP)
├── auxbasis         ← RI auxiliary basis (cbas)
├── mos              ← orbital guess (extended Hückel)
└── define.log       ← define transcript
```

After a correct `define`, `control` contains these blocks:

```text
$coord    file=coord
$basis    file=basis
$scfmo   file=mos
$scfdamp   start=  0.600  step=  0.050  min=  0.100
$ricc2
    adc(2)
    sos   cos=  1.30000000
    maxiter = 5000
$cbas    file=auxbasis
$freeze
 implicit core=   ...
$excitations
    irrep=a  multiplicity=  1  nexc=  5
    spectrum  states=all  operators=diplen,dipvel,angmom,qudlen
    xgrad  states=all
    tmexc  istates=all fstates=all  operators=diplen,soc
$response
   gradient = on
   fop  relaxed  operators=diplen,qudlen
```

Every element should show `basis =… def2-SVP` and `cbas =… def2-SVP`. Check `define.log` for "define ended normally".

---

### After a successful job (full results)

```text
mol01/
├── molecule.xyz
├── coord
├── control
├── basis
├── auxbasis
├── mos                           ← converged SCF orbitals
├── define.log
├── adc2.slurm
├── dscf.out                      ← ground-state SCF log
├── ricc2.out                     ← ADC(2)-SOS log (spectrum is here)
├── energy                        ← SCF energy (Hartree)
├── exstates                      ← 5 ADC(2) excitation energies (Hartree)
├── statistics, statistics.ricc2  ← timing data
├── restart.cc
├── CC*                           ← ricc2 intermediates
├── adcp2-*.cao, mp2-*.cao        ← ADC(2) densities
└── dscf-ricc2-<jobid>.{out,err}  ← Slurm stdout/stderr
```

**How to tell a job finished:**

```bash
grep -q "dscf : all done"  mol01/dscf.out  && echo "SCF OK"
grep -q "ricc2 : all done" mol01/ricc2.out  && echo "ADC(2) OK"
```

### Key numbers

**`energy`** — SCF total energy in Hartree:

```text
$energy      SCF               SCFKIN            SCFPOT
     1 -1361.578381391      1356.502479974     -2718.080861365
$end
```

**`exstates`** — five singlet excitation energies in Hartree:

```text
$excitation_energies_ADC(2)__1^a___  threxc=.10000E-05  nstates=    5
    1    0.15398615989047E+00
    2    0.15795619884228E+00
    3    0.16871750867755E+00
    4    0.19913135642667E+00
    5    0.20983840286099E+00
```

**`ricc2.out`** — the main harvest file. MP2 ground-state energy, excitation energies in eV and cm⁻¹, and length-gauge oscillator strengths:

```text
     *   Final MP2 energy                        :  -1365.2458896562      *
  |    frequency :   0.1539861599 a.u.     4.19018 e.V.     33796.1 rcm     |
       oscillator strength (length gauge)   :      0.04125459
  |    frequency :   0.1579561988 a.u.     4.29821 e.V.     34667.4 rcm     |
       oscillator strength (length gauge)   :      0.12918555
   ****  ricc2 : all done  ****
```

---

## Troubleshooting

| Problem | Likely cause | Fix |
| --- | --- | --- |
| `define.log` does not end with "define ended normally" | Missing blank lines in `define_answers.txt`, or the molecule prompted extra questions (charge, symmetry) | Verify the two leading blank lines. Open `define.log` and look for where `define` asked an unexpected question |
| `prepare.sh` marks a molecule FAIL | `define` wrote `control` without `adc(2)` or with triplet multiplicity | Inspect `define.log`; the molecule may need a custom answer file |
| `dscf` does not converge | Poor SCF starting guess or difficult electronic structure | Try tighter damping: edit `control` and change `$scfdamp start=0.700 step=0.025 min=0.050`, or switch to `ridft` |
| `ricc2` runs out of memory | Molecule too large for the allocated memory at the given core count | Increase `--mem` in `config.sh` or directly in `adc2.slurm` |
| `QOSMaxJobsPerUserLimit` on submit | Slurm queue limit reached | Wait for running jobs to finish, then run `submit_jobs.sh` again — it skips finished molecules |
| Job killed by walltime | Molecule needs more than 5 days | Increase `SLURM_TIME` in `config.sh` and ensure your partition allows it |

**Monitoring jobs:**

```bash
squeue -u "$USER" -o '%.18i %.16j %.10P %.2t %.10M %R'
bash check_status.sh
```

---


