# Conduit to Column

A single self-contained HTML page that solves a steady 1D volcanic conduit and a bent-over plume live in the browser, and checks that live solution against real runs of the research codes it is built to approximate. Built for EASC10132, Magmatic & Volcanic Processes, School of GeoSciences, University of Edinburgh.

## Run it

Open **https://ebreard.github.io/conduit-to-column/** in any recent browser, or download `conduit_to_column.html` and open it locally. That one file (about 12.5 MB) is the whole tool: no build step, no server, no installation. It also runs offline; without a connection the IBM Plex fonts fall back to your system fonts.

## What it does

Five tabs: **Conduit** (depth profile of pressure, velocity, gas fraction, viscosity), **Column** (plume height, neutral buoyancy, collapse), **Regime map** (mass eruption rate across water content and conduit radius, and where the explosive window closes with magma type, temperature and crystal content), **Scaling** (column height against mass eruption rate for every chained run, beside the quarter-power law), **Chemistry** (equilibrium decompression of named real magmas). Every state can be exported as CSV: the conduit and plume profiles, the precomputed MAMMA grids, and a sweep of any one control.

Every slider move re-solves the live physics instantly, in JavaScript, in the browser. Nothing about the interactive page is a neural network: the tiles you read are always the direct output of the 1D conduit ODE and the plume integration described below. The networks described in this README do one job only, they draw the **second line**, the answer the full research code (MAMMA, PLUME-MoM-TSM, rhyolite-MELTS) would have given at that exact setting, everywhere the page does not already have a real run of that code to show you directly.

## Three layers, not one model

1. **Live model.** A steady, 1D, two-phase (melt + exsolving gas + crystals) conduit solver and a top-hat plume model, both solved directly on every interaction. Fast because the physics is simplified (steady state, no bubble growth kinetics, equilibrium degassing), not because anything is cached or learned.
2. **Real precomputed runs.** Actual runs of the full codes, used verbatim wherever they cover the chosen settings: 4,608 MAMMA runs on five grids (3,924 converged; the dashed conduit curve), 15,168 PLUME-MoM-TSM columns chained onto them at four wind speeds (the dashed plume curve; the named magmas are chained from their own volcano's summit), and a rhyolite-MELTS decompression table for the Chemistry tab's six named magmas that covers every setting of the tab's controls: five oxygen buffers, water 2-8 wt% and the whole temperature slider (1,667 of 1,704 cells converged, each pooled from 6 to 36 MELTS runs).
3. **The emulator.** Small neural networks (called `surrogate` in the source comments; this document calls them the emulator throughout) trained on much larger HPC campaigns of the same three codes, which answer wherever the precomputed runs above do not reach, following every slider rather than only the grid corners the real runs sit on. The page always labels which of the three answered.

## The codes, briefly

| Code | What it does here | Reference |
|---|---|---|
| **MAMMA** | Steady 1D two-phase conduit solver (melt, exsolving water, crystals as compressible stiffened gases); supplies the conduit depth profile and mass eruption rate the live model is tuned against. Patched locally to add three more fragmentation criteria beyond its default porosity threshold. | M. de' Michieli Vitturi (INGV Pisa) and G. La Spina. Code: [github.com/demichie/MAMMA](https://github.com/demichie/MAMMA). Representative application: Arzilli et al. (2019), *Nature Geoscience*, 12, 1023-1028 |
| **PLUME-MoM-TSM** | 1D integral volcanic column model (method of moments over a particle size distribution) with umbrella cloud spreading; supplies column height, neutral buoyancy level and collapse behaviour. | de' Michieli Vitturi, M. & Pardini, F. (2021), *Geoscientific Model Development*, 14, 1345-1377. Code: [github.com/demichie/PLUME-MoM-TSM](https://github.com/demichie/PLUME-MoM-TSM) |
| **rhyolite-MELTS** (1.0.2 and 1.2.0) | Thermodynamic phase-equilibrium code; supplies the Chemistry tab's equilibrium decompression paths (crystal fraction, dissolved/exsolved volatiles, melt composition and viscosity vs pressure). 1.2.0 adds a mixed H2O-CO2 fluid. | Gualda, Ghiorso, Lemons & Carley (2012), *Journal of Petrology*, 53(5), 875-890; CO2 capability: Ghiorso & Gualda (2015), *Contributions to Mineralogy and Petrology*, 169:53 |
| **Giordano, Russell & Dingwell (2008)** | Melt viscosity model from composition, water and temperature; used live for every named and custom magma, and as an input feature to the MAMMA emulator. | *Earth and Planetary Science Letters*, 271, 123-134 |
| **Hess & Dingwell (1996)** | Non-Arrhenian viscosity model for the page's one generic (not compositionally specified) rhyolite option. | *American Mineralogist*, 81, 1297-1300 |
| **Costa (2005)** | Relative viscosity of a crystal-bearing melt as a function of crystal fraction; combines with melt viscosity for the bulk magma viscosity the conduit solver uses. | *Geophysical Research Letters*, 32, L22308 |

## The HPC campaigns

All training data, and the precomputed MELTS table, came from **ARCHER2** (the UK national HPC service, EPCC), run as a task-farm: a campaign is a Sobol or Saltelli design over named parameter ranges (`design.py`), handed out in claimable blocks on the Lustre filesystem (`mkdir rounds/rNNN/claims/bNNNNNN` is atomic, so a worker just takes whatever is free), executed by `node_worker.py` calling the underlying Fortran/C code once per sampled point, and merged back into arrays by `consolidate.py`. ARCHER2's `short` QoS caps a job at 20 minutes, so campaigns ran as chains of jobs with `--dependency=singleton` (one job of a given name running at a time, in submission order, no job-id bookkeeping), each chain computing its own remaining time from Slurm rather than assuming a fixed wall clock. No state passes between chained jobs: work not finished when the clock runs out is simply picked up by the next one.

| Campaign | Code | Design | Runs | Converged / usable |
|---|---|---|---|---|
| MAMMA, fixed fragmentation (porosity 0.80) | MAMMA | 9-axis Sobol, 1,048,576 points | 1,374,424 (with retries) | 384,087 choked and converged |
| MAMMA, A1 (porosity threshold as an input) | MAMMA, patched | 10-axis Sobol | 65,536 | 47,116 (71.9 %) |
| MAMMA, A2 (wall-shear brittle failure) | MAMMA, patched | 10-axis Sobol | 131,072 | 106,124 (81.0 %) |
| PLUME-MoM-TSM, main campaign | PLUME-MoM-TSM | 15-axis Sobol | 8,388,608 columns | 8,387,871 |
| rhyolite-MELTS 1.0.2, water only | rhyolite-MELTS | 5-axis Sobol (water, a two-parameter composition manifold, temperature, oxygen buffer) plus a 65,536-path silicic, water-rich top-up | 1,114,112 paths, 64 pressures each | 1,033,877 (92.8 %) |
| rhyolite-MELTS 1.2.0, water + CO2 | rhyolite-MELTS | as above plus a CO2 axis, 0-5000 ppm | 1,048,576 paths | 978,916 (93.4 %) |
| rhyolite-MELTS 1.0.2, named-magma table | rhyolite-MELTS | 6 magmas x 5 oxygen buffers x 7 water contents x up to 13 temperatures, six pressure ladders per cell and up to 30 more where MELTS stuck | 26,028 ladders | 1,667 of 1,704 cells |

Measured throughput: 5,656 MAMMA runs per node-hour; PLUME-MoM-TSM at 0.13 s per column on one core at the median (0.40 s at the 90th percentile); the 26,028 ladders of the MELTS table took about 16 minutes of wall clock on four to eight nodes.

## How each emulator was trained

All are small feedforward networks, `tanh` hidden layers and a linear output, with inputs standardised to zero mean and unit variance before the first layer. Weights ship as plain JSON inside the page, so inference is a few thousand multiply-adds in JavaScript, no runtime or library needed.

**MAMMA emulator.** Thirteen input features (log10 radius, conduit length, chamber pressure over lithostatic, water content, crystal fraction, temperature, log10 melt viscosity, the Giordano B and C coefficients wet and dry, a groundwater flag, log10 country-rock permeability), predicting log10(mass eruption rate), paired with a classifier for whether a setting chokes at all. It replaced an earlier 252-run lookup table that only answered at its own grid corners. The two patched fragmentation criteria each add a fourteenth feature (the porosity threshold, or log10 of the wall-shear relaxation fraction); the wall-shear version needs a third network, a classifier between the fragmenting and non-fragmenting branches, because those are different regimes that a single smooth network would blur. Held-out error: 0.6 % (fixed criterion), 2.3 % (A1), 5.9 % / 1.7 % (A2, fragmenting / unfragmented branch).

**Plume emulator.** Fifteen input features (log MFR, log exit velocity, vent gas fraction, mixture temperature, vent elevation, relative humidity, grain-size shift, two entrainment coefficients, a five-parameter wind profile, and whether the column entrains above its neutral buoyancy level), with a classifier for collapse and three regressors (column top, neutral buoyancy level, umbrella radius). Held-out error 0.27 %, 0.30 % and 0.43 %, collapse called correctly 99.9 % of the time against a 79 % base rate. Two of the wind-related inputs sit partly outside the page's own slider range, so the page maps its single wind control onto the nearest in-training-box profile; that mapping is itself calibrated against the 799 chained runs of the page's main grid, not assumed, and comes back unbiased in column top, 1 % low in neutral buoyancy, and agreeing with the real runs on collapse 96.6 % of the time.

**Chemistry emulator.** Staged rather than flat: a classifier first identifies which silicate assemblage is stable, then one small regressor per assemblage returns the melt composition, crystal fraction and volatile content, which is what lets it reproduce the sharp jumps at a phase boundary that one smooth network averages across. Six inputs for the water-only (1.0.2) calibration (water content, a two-parameter position in composition space, temperature, oxygen fugacity, log pressure); a seventh, dissolved CO2, for the 1.2.0 mixed-volatile calibration. Held-out error: assemblage correct 90 % (1.0.2) / 89.9 % (1.2.0) of the time, crystal fraction within 0.019-0.04, melt SiO2 within 0.38-0.47 wt%, viscosity within 0.09-0.14 dex. With the named magmas covered by real MELTS runs, it answers for the generic magma, an analysis typed in by hand, the CO2 calibration, and the few table cells where MELTS never converged.

## How it was checked

Three separate checks, kept distinct rather than blended into one error bar:

1. **The emulator against held-out points from its own training campaign.** The error figures quoted above are all against points the network never trained on.
2. **The emulator against real runs it never trained on.** For the plume emulator, the 2,628 chained PLUME-MoM-TSM columns of the main grid; the wind mapping, needed where the page's slider ranges exceed the training box, was set on the 799 columns of the earlier, smaller grid. For the chemistry emulator, 799 pressure levels of real rhyolite-MELTS run on the campaign's own composition manifold: crystal fraction within 0.009 at the median and 0.043 at the 90th percentile, viscosity within 0.13 dex at the 90th percentile.
3. **The precomputed MELTS paths against physics.** Rhyolite-MELTS sometimes stops moving: in about a fifth of the table's first pressure ladders a node returns the same state, to the fourth decimal, at every remaining pressure. Each ladder therefore ends where its state stops changing once crystals or a fluid are present, and any level whose melt holds well over the dissolved water of that magma's fluid-saturated melts at the same pressure is dropped. Cells left thin got up to 30 more runs: finer, denser at low pressure, more strongly jittered, or started at each pressure by cooling to the target temperature, which reaches the same equilibrium from a melt-rich first guess. Where the rebuilt table and the first 180-cell table both pass these checks they agree to better than 0.001 in crystal fraction at the median.

Where a network's own classifier is unsure, or where its answer would require physically impossible behaviour (dissolved water rising as pressure falls, for instance), the page shows a gap rather than a confident wrong number, and the MELTS levels dropped in check 3 show as gaps in the same way.

## Repository contents

- `conduit_to_column.html` - the tool, single file, no build step.
- `index.html` - sends the GitHub Pages address to the tool.
- `README.md` - this file.
- `LICENSE` - MIT.

## License

MIT, see `LICENSE`. The model results embedded in the page come from MAMMA, PLUME-MoM-TSM and rhyolite-MELTS, cited above; each magma composition carries its own published source in the page.

## Attribution

EASC10132, Dr Eric C. P. Breard, School of GeoSciences, University of Edinburgh. MAMMA and PLUME-MoM-TSM courtesy of Mattia de' Michieli Vitturi, INGV Pisa. All emulator training, HPC campaign design and execution, and page implementation: E. Breard, 2026.
