# GNoME Materials Toolkit — Usage Guide

## What This Is

This toolkit provides three complementary access pathways to DeepMind's **GNoME** (Graph Networks for Materials Exploration) dataset and model ecosystem. GNoME is a 2023 Nature paper by Merchant et al. that used graph neural networks to predict the stability of ~380,000 previously unknown inorganic crystal structures — the largest single expansion of the known stable materials space in history.

This repo does not implement GNoME itself. It wraps three existing tools:
- The community OPTIMADE mirror of GNoME (`optimade-gnome.odbx.science`)
- DeepMind's publicly released stable materials CSV (520k+ entries)
- CHGNet — a graph neural network trained on the same DFT dataset as GNoME, used here as a local stability predictor

The value of this repo is having all three in one place with a consistent interface.

---

## Is This a Novel Pipeline?

Honestly: the components are not novel. GNoME, CHGNet, and OPTIMADE all existed before this repo. What this provides is:

1. A unified CLI interface across three different access patterns
2. Automatic fallback (GNoME endpoint -> MP endpoint if GNoME is down)
3. CHGNet cross-validation against JARVIS-DFT in a single call
4. A local cache of the full dataset for offline screening

The scientific novelty comes from *what you do with it*, not the toolkit itself. If you use it to find a new stable phase in a system no one has screened before, that's the contribution.

---

## The Three Pathways Explained

### Pathway 1: OPTIMADE API Query (`gnome_optimade_query.py`)

**What it does:** Sends a live query to the GNoME OPTIMADE server and returns matching structures.

**When to use it:**
- You want quick results for a specific element system
- You don't want to store 150 MB locally
- You want to integrate GNoME into a larger pipeline that already uses OPTIMADE

**What it can't do:**
- Filter by energy or formation enthalpy (the OPTIMADE GNoME mirror doesn't expose those fields cleanly)
- Return more than ~100 results per query without pagination

**Example use cases:**
```bash
# Find all GNoME structures containing Fe, Mn, and Al
python gnome_optimade_query.py Fe Mn Al

# Find Ti-Al-V structures (aerospace alloys)
python gnome_optimade_query.py Ti Al V

# Find Li-containing battery cathode candidates
python gnome_optimade_query.py Li Fe P O
```

**Output:** `gnome_optimade_results.json` — list of structure entries with IDs, formulas, and site counts.

**Limitation:** The GNoME OPTIMADE mirror (`optimade-gnome.odbx.science`) is community-maintained and unofficial. It may go down. The script falls back to the Materials Project OPTIMADE endpoint automatically.

---

### Pathway 2: Full Dataset Screener (`gnome_screen.py`)

**What it does:** Downloads DeepMind's official `stable_materials_summary.csv` (~150 MB, 520k+ entries) from Google Cloud Storage and screens it locally using pandas.

**When to use it:**
- You want to search across the *entire* GNoME dataset, not just what an API returns
- You want to filter by decomposition energy, formation energy, space group, or composition
- You want to do data science — statistics, distributions, comparisons

**What the CSV contains:**
| Column | Description |
|--------|-------------|
| Composition | Chemical formula |
| Elements | Space-separated element list |
| Decomposition Energy (eV/atom) | Distance from convex hull — lower = more stable |
| Formation Energy (eV/atom) | Energy relative to elemental references |
| Space Group Number | Crystal symmetry |
| Volume per Atom (A^3) | Unit cell volume normalized per atom |

All 520k+ entries are within 1 meV/atom of the convex hull — meaning GNoME only kept structures it predicted to be thermodynamically stable.

**Example use cases:**
```bash
# Screen for Fe-Mn-Al-Si-Ni-C system (default)
python gnome_screen.py

# Force re-download (if CSV was updated)
python gnome_screen.py --force
```

**To screen a different element system**, edit the `target_elements` set in `screen_fe_sma()` or add your own function:
```python
# Example: screen for high-entropy alloy candidates (Cantor alloy system)
results = screen_fe_sma(df, target_elements={"Fe", "Co", "Ni", "Cr", "Mn"})
```

**Output:** `data/gnome/fe_sma_gnome_hits.csv` — filtered entries sorted by decomposition energy.

**Practical value:** You might find that GNoME predicts a stable ternary or quaternary in your element system that no one has synthesized. That's a hypothesis you can then validate experimentally or with higher-level DFT.

---

### Pathway 3: Local GNN Prediction (`gnome_predict.py`)

**What it does:** Uses **CHGNet** (Crystal Hamiltonian Graph Neural Network) as a local surrogate model to predict the energy and forces of any crystal structure you provide.

**What CHGNet is:** A graph neural network trained on 1.5 million Materials Project DFT calculations, including magnetic ordering. It's one of the best-performing models on the Matbench Discovery benchmark for stability prediction — the same benchmark GNoME was evaluated on. It's not the GNoME model itself, but it was trained on overlapping data and performs comparably for stability screening.

**When to use it:**
- You have a hypothetical structure (from DiffractGPT, from literature, from your own design) and want a fast energy estimate
- You want to relax a structure computationally before running expensive DFT
- You want to compare predicted stability across a series of compositions

**What it can't do:**
- Replace DFT for high-accuracy results
- Handle extremely disordered structures well
- Predict phase transformation temperatures (it's 0K ground state energy only)

**Example use cases:**
```python
# Single-point energy prediction
python gnome_predict.py my_structure.cif
python gnome_predict.py my_structure.POSCAR

# Structure relaxation (Python API)
from gnome_predict import relax_structure
result = relax_structure("my_structure.POSCAR", steps=200)
print(f"Final energy: {result['final_energy_eV_atom']:.4f} eV/atom")
print(f"Converged: {result['converged']}")
```

**Cross-validation:** The script automatically queries JARVIS-DFT for any formula it recognizes, giving you a second independent DFT reference to compare against.

**Output:**
- Energy per atom (eV/atom)
- Max force on any atom (eV/Å) — tells you how far the structure is from a local minimum
- JARVIS-DFT comparison if available
- Relaxed structure file if you use `relax_structure()`

---

## Full Workflow Example: Screening a New Alloy System

Say you're designing a new Ti-Zr-Nb shape memory alloy and want to know what stable phases exist before going to the lab:

```bash
# Step 1: Quick API scan — what does GNoME know about Ti-Zr-Nb?
python gnome_optimade_query.py Ti Zr Nb
# -> Returns structures, saves to gnome_optimade_results.json

# Step 2: Download full dataset and screen for all Ti-Zr-Nb-containing stable phases
# (edit gnome_screen.py to use target_elements={"Ti", "Zr", "Nb"})
python gnome_screen.py
# -> Returns CSV of all stable phases sorted by decomposition energy

# Step 3: Take your most promising candidate CIF from the GNoME CSV,
# run CHGNet to get energy/forces, and relax if needed
python gnome_predict.py candidate.cif
```

This three-step workflow takes minutes and gives you a DFT-level survey of an entire composition space before touching a furnace.

---

## Limitations to Be Aware Of

| Limitation | Detail |
|------------|--------|
| GNoME is 0K DFT | No temperature, no entropy, no phase diagram — just ground state stability |
| CHGNet is a surrogate | ~50-100 meV/atom error vs actual DFT; use for screening, not final answers |
| OPTIMADE mirror is unofficial | `optimade-gnome.odbx.science` can go down; fallback to MP is automatic |
| GNoME CSV has no structure files | The 520k-entry CSV has formulas and energies but not atomic coordinates — for full CIF files you need the separate `by_reduced_formula.zip` (~several GB) |
| GNoME covered ~2023 MP data | Structures discovered or calculated after the training cutoff aren't included |

---

## Data Files Available from DeepMind

All files at `https://storage.googleapis.com/gdm_materials_discovery/gnome_data/`:

| File | Size | Contents |
|------|------|----------|
| `stable_materials_summary.csv` | ~150 MB | 520k+ stable materials with energies and properties |
| `stable_materials_r2scan.csv` | smaller | r²SCAN-validated subset (higher accuracy functional) |
| `by_reduced_formula.zip` | ~several GB | CIF files indexed by formula |
| `by_id.zip` | ~several GB | CIF files indexed by GNoME ID |
| `elemental_references.json` | small | DFT elemental reference energies used for formation energy calculation |

---

## References

- Merchant et al., "Scaling deep learning for materials discovery," *Nature* 624, 80–85 (2023). https://doi.org/10.1038/s41586-023-06735-9
- Deng et al., "CHGNet: Pretrained universal neural network potential for charge-informed atomistic simulations," *Nature Machine Intelligence* 5, 1031–1041 (2023). https://doi.org/10.1038/s42256-023-00716-3
- Evans et al., "Developments and applications of the OPTIMADE API for materials discovery," *Digital Discovery* 3, 1509 (2024). https://doi.org/10.1039/D4DD00039K
