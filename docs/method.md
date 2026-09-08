# How it works

[← Project overview](../README.md) · [Results](results.md) · [MATLAB guide](getting-started.md)

The research asks how a simulated laser microscope can recover finer information about a transparent object. Its starting point is a wave model; its distinguishing extension is an extra optical mask that makes compressive measurements possible.

## 1. Propagate a coherent field

The source is monochromatic, with a wavelength of **632.8 nm**. An aperture limits the incident field. Under the Fresnel approximation, propagation can be computed by applying a transfer function to the field's Fourier transform:

$$
U_z = \mathcal{F}^{-1}\!\left\{\mathcal{F}\{U_0\}\,H\right\},
\qquad
H(f_x,f_y)=e^{ikz}e^{-i\pi\lambda z(f_x^2+f_y^2)},
\qquad k=\frac{2\pi}{\lambda}.
$$

This is the operation implemented in [fresnel_advance.m](../fresnel_advance.m). [propagate_light_through_space.m](../propagate_light_through_space.m) surrounds each input window with zeros before propagation. The mathematical development is in thesis §§1.1–1.3.

## 2. Encode the object in phase

A transparent object changes the optical path length. The thin-object model uses a thickness function and a refractive index to turn that change into a complex phase mask. Multiplying the incident field by this mask produces the field immediately after the object.

[lens_thickness.m](../lens_thickness.m), [create_lens.m](../create_lens.m), and [apply_lens.m](../apply_lens.m) implement the modeled lens and its placement. The thesis develops this model in chapter 2.

## 3. Turn pattern shifts into measurements

An attenuation grid divides the field into local openings. First, the model calculates a reference diffraction pattern without the object. It then inserts the object and measures how each pattern's centroid shifts in x and y.

[center_of_mass.m](../center_of_mass.m) finds the centroid, and [calculate_phi.m](../calculate_phi.m) converts its displacement from sample indices to meters. Although the source calls the outputs `phi_x` and `phi_y`, the active calculation returns **centroid displacements**, not angles or phase in radians. These displacements carry the information used to estimate phase gradients.

The thesis describes how gradient information can be integrated to recover a thickness profile (§3.1). The preserved experiment scripts expose the displacement maps and, in the CS experiment, plot their combined magnitude. They do not perform that final thickness integration.

## 4. Add the compressive-sensing mask

![Original thesis Figure 5.9, showing the additional fine measurement mask between the attenuation grid and the object.](assets/compressive-optical-setup.jpg)

*Light source → attenuation grid → fine measurement mask → transparent object → sensor.*

The fine mask splits each coarse opening into **8 × 8 regions**. Each binary pattern transmits light through a different selection of those regions, yielding a different centroid observation. The model also measures a reference for each pattern, so shifts are calculated against the matching mask without the object.

[initialize_compressive_sensing.m](../initialize_compressive_sensing.m) creates the patterns and reconstruction dictionary. The active mask generator, [balanced_random.m](../balanced_random.m), opens exactly half of the 64 elements in each row and randomly permutes their positions. [create_CS_reference_windows.m](../create_CS_reference_windows.m) calculates the corresponding references.

## 5. Use sparsity to recover the finer signal

For a local signal x, the CS model is:

$$
x = \Psi s,\qquad y = \Phi\Psi s = \Theta s.
$$

The dictionary Ψ represents the signal through transform coefficients s. When those coefficients are sparse or compressible, a suitable set of measurements can constrain a useful reconstruction even when M is smaller than N. In this experiment, **M = 32** and **N = 64**, with a DCT dictionary.

[CS_reconstruction.m](../CS_reconstruction.m) calls [cs_sr07.m](../cs_sr07.m) separately for the two displacement components. The wrapper uses SeDuMi to solve the sparse-recovery problem:

$$
\hat{s}=\underset{s}{\arg\min}\;\|s\|_1
\quad\text{subject to}\quad
\|y-\Theta s\|_2\leq\varepsilon.
$$

The optical reconstruction passes **ε = 0**, giving the equality-constrained case. The recovered coefficients are mapped through Ψ and reshaped to an 8 × 8 patch. [reconstruct_from_windows.m](../reconstruct_from_windows.m) assembles all 16 patches into a 32 × 32 output map.

**Why it matters:** information within each opening is encoded across multiple patterns, then recovered computationally. This is the step that raises the output sampling density above one value per opening.

The linear model is the reconstruction assumption used by the implementation. The thesis's synthetic results demonstrate its behavior for the selected object and masks; they do not establish exact recovery for arbitrary centroid measurements or objects. The [results notes](results.md) explain the evidence and its limits.

## References

- **Josip Vukušić (2017).** [*Model koherentnog optičkog sustava korištenog u laserskoj mikroskopiji*](thesis/josip-vukusic-masters-thesis-2017.pdf). Master's thesis no. 1469, University of Zagreb, FER. Primary source for this project's theory, experiments, and figures.
- **Joseph W. Goodman (1996).** *Introduction to Fourier Optics*, second edition. The wave-optics foundation cited in the thesis and propagation routine.
- **Kaye Morgan, David Paganin, and Karen Siu (2011).** [*Quantitative single-exposure x-ray phase contrast imaging using a single attenuation grid*](https://research.monash.edu/en/publications/quantitative-single-exposure-x-ray-phase-contrast-imaging-using-a). *Optics Express* 19(20), 19781–19789. The grid-based phase-imaging method cited by this work. The university publication record supplies the 2011 date; dates in the original thesis and code comments differ.
- **Richard G. Baraniuk (2007).** [Compressive-sensing publications and resources at Rice University](https://dsp.rice.edu/cs/). The thesis cites his *Compressive Sensing* article in *IEEE Signal Processing Magazine*, July 2007.
- **Jos F. Sturm (1999).** *Using SeDuMi 1.02, a MATLAB toolbox for optimization over symmetric cones*. *Optimization Methods and Software* 11–12, 625–653. Citation supplied in the bundled [SeDuMi README](../dependencies/SeDuMi_1_3/Readme.txt).
