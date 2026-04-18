# gnome-materials

General-purpose toolkit for working with DeepMind's GNoME dataset (520,000+ AI-discovered stable materials). Designed for any composition system — not project-specific (not tied to my Fe-SMA research; this is broader computation).

## Three Pathways

| Script | Approach | Use when |
|--------|----------|----------|
| `gnome_optimade_query.py` | OPTIMADE API query | Quick lookup, no local storage needed |
| `gnome_screen.py` | Full CSV download + local screen | Bulk screening across many compositions |
| `gnome_predict.py` | CHGNet local GNN inference | Predicting stability of your own structures |

## Quick Start

```bash
pip install -r requirements.txt

# Query GNoME database for any element system
python gnome_optimade_query.py Fe Mn Al
python gnome_optimade_query.py Ti Al V

# Download full GNoME dataset and screen (~150 MB, cached after first run)
python gnome_screen.py                        # defaults to Fe-Mn-Al-Si-Ni-C
# Edit screen_fe_sma() or add your own screening function for other systems

# Predict energy/stability of a structure file
python gnome_predict.py path/to/structure.cif
python gnome_predict.py path/to/structure.POSCAR
```

## Data Sources

- **GNoME CSV**: `https://storage.googleapis.com/gdm_materials_discovery/gnome_data/stable_materials_summary.csv`
- **OPTIMADE mirror**: `https://optimade-gnome.odbx.science` (community-maintained, unofficial)
- **CHGNet model**: downloads automatically on first run (~400k parameter GNN)

## References

- [GNoME paper — Merchant et al., Nature 2023](https://www.nature.com/articles/s41586-023-06735-9)
- [CHGNet](https://chgnet.lbl.gov/)
- [MatGL](https://matgl.ai/)
- [OPTIMADE spec](https://www.optimade.org/)
