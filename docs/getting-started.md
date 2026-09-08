# Running the original MATLAB experiments

[← Project overview](../README.md) · [Results](results.md) · [Method](method.md)

This repository preserves the research implementation from 2017. The instructions below are based on inspection of the source and bundled dependency documentation. The documentation refresh did not rerun MATLAB or change the algorithms, parameters, filenames, or dependencies.

## Requirements

| Experiment | Requirements |
| :--- | :--- |
| Aperture diffraction and lens propagation | MATLAB and the project files |
| Conventional grid reconstruction | MATLAB and the project files |
| Optical compressive sensing | MATLAB, Wavelet Toolbox with `wmpdictionary`, and a working SeDuMi installation |

The code creates dense arrays and repeatedly propagates padded fields, so the reconstruction experiments require substantially more memory and time than the single-aperture example. The thesis reports about **35.5 minutes** for its CS experiment on the original setup; that is historical context, not an estimate for your machine.

The CS dictionary is constructed using [`wmpdictionary`](https://www.mathworks.com/help/wavelet/ref/wmpdictionary.html). MathWorks marks it for future removal and recommends `sensingDictionary`; this archive retains the original API. No specific current MATLAB release or GNU Octave compatibility is claimed.

## First experiment

Clone the repository:

```sh
git clone https://github.com/josip-vukusic/CoherentOpticalSystemModel.git
cd CoherentOpticalSystemModel
```

Open MATLAB with the repository root as the Current Folder. Run:

```matlab
optical_system
```

This propagates a field from a square aperture and plots its magnitude at the sensor plane. Next, run the transparent-lens variant:

```matlab
optical_system_with_object
```

Both examples set their own parameters. They do not need the CS solver or an external input image.

## Compressive-sensing experiment

Add the supporting routines and the bundled SeDuMi directory to the MATLAB path:

```matlab
addpath('dependencies');
addpath(genpath(fullfile('dependencies', 'SeDuMi_1_3')));
which wmpdictionary -all
which sedumi -all
```

If `wmpdictionary` is missing, this run needs a MATLAB installation with the corresponding Wavelet Toolbox API. If SeDuMi is missing or its binaries cannot load, consult the bundled [installation instructions](../dependencies/SeDuMi_1_3/Install.txt). They describe rebuilding with `install_sedumi` when necessary, which may require a configured MATLAB C compiler. The bundled binaries are old, and their compatibility with your platform must be checked locally.

Once those dependencies are available, run:

```matlab
optical_system_compressive_sensing2
```

The experiment initializes the microscope, creates 32 masks, calculates their reference centroids, measures each object window, solves the x and y reconstruction problems, and displays the assembled maps and their combined magnitude.

It writes `Phi.mat`, `centers_CS_ref_x.mat`, `centers_CS_ref_y.mat`, and `object_mask_tensor.mat` in the Current Folder, replacing files with those names if they already exist. The conventional grid experiments also save `center_x.mat` and `center_y.mat`. Preserve outputs before another run if you want to compare the same measurement configuration later. These generated files are covered by `.gitignore`.

## Experiment map

| Script | Purpose and expected output |
| :--- | :--- |
| [optical_system.m](../optical_system.m) | Square-aperture diffraction; sensor-plane field magnitude |
| [optical_system_with_object.m](../optical_system_with_object.m) | Transparent lens; sensor-plane field magnitude |
| [optical_system_object_reconstruction.m](../optical_system_object_reconstruction.m) | Grid-based displacement reconstruction with an offset object; x and y maps |
| [optical_system_compressive_sensing1.m](../optical_system_compressive_sensing1.m) | Conventional grid reconstruction with a centered object; no CS solver call despite the filename |
| [optical_system_compressive_sensing2.m](../optical_system_compressive_sensing2.m) | Full optical CS experiment; reconstructed x/y maps and displacement magnitude |
| [optical_system_compressive_sensing0.m](../optical_system_compressive_sensing0.m) | Separate legacy image-reconstruction example; incomplete in this checkout, as detailed below |

The appendix of the Croatian thesis uses the original Croatian script names. This table points to the English filenames present in this repository.

## Saved defaults and thesis settings

The initializer files contain the source of truth for a new run:

| Setting | Current source value | Location |
| :--- | :--- | :--- |
| Wavelength | 632.8 nm | `initialize_microscope.m` |
| Spatial sampling interval | λ / 10 = 63.28 nm | `initialize_microscope.m` |
| Aperture width | 1,024 simulation samples ≈ 64.8 µm | `initialize_microscope.m` |
| Coarse grid | 4 × 4 openings | `initialize_microscope.m` |
| Reconstruction experiment propagation distance | 0.0001 m = 0.1 mm | `initialize_microscope.m` |
| Lens refractive index | 1.5 | `initialize_microscope.m` |
| Fine region width | 128 simulation samples ≈ 8.1 µm | `initialize_compressive_sensing.m` |
| Fine grid per opening | 8 × 8 = 64 values | `initialize_compressive_sensing.m` |
| Mask count | 32 | `initialize_compressive_sensing.m` |
| Mask generator | Balanced binary random patterns | `balanced_random.m` |
| Dictionary | DCT via `wmpdictionary` | `initialize_compressive_sensing.m` |

The standalone propagation scripts set **z = 0.001 m (1 mm)** themselves. The thesis also states **1 mm** for its reconstruction experiments, while the current shared initializer uses **0.1 mm**. The thesis's numerical sampling-interval text has a typographical inconsistency: the code's `lambda/10` evaluates to **63.28 nm**. Rounded physical dimensions in the thesis and conflicting grid-count prose should not replace the array dimensions in the source.

The figures are historical results, not a promise that the present defaults reproduce their exact values. Random masks vary between runs, and no fixed seed is set by the initializer. For a repeatable new run, you can set the random generator from the MATLAB command window before running the experiment:

```matlab
rng(0, 'twister');
optical_system_compressive_sensing2
```

This defines a new repeatable mask sequence; it does not recover the random sequence used for the published thesis figure. Record the MATLAB release, solver, parameters, seed, and saved `Phi` matrix alongside any new result.

## Known legacy details

- **The image demo is incomplete.** `optical_system_compressive_sensing0.m` references `lena.png`, `slice_lena`, and `lena_CS_reconstruction`, which are absent from this checkout. A differently named `image_CS_reconstruction.m` is present. Use the optical experiments above as the documented starting point.
- **Variable names are historical.** `phi_x` and `phi_y` contain displacements in meters in the active implementation. They are not phase values in radians.
- **Displayed magnitude is not physical intensity.** The code plots and computes centroids from `abs(sensor)`, while intensity is proportional to `abs(sensor).^2`.
- **Thickness integration is described in the thesis.** The preserved experiment scripts display displacement maps and their magnitude; they do not implement the final integration to object thickness.
- **Scripts use shared state.** Global variables and initializer functions control the reconstruction. Setting an unrelated workspace variable may be overwritten during initialization.

If you reproduce an experiment on a current MATLAB installation, a report with the environment and observed results would help future readers. See [CONTRIBUTING.md](../CONTRIBUTING.md).
