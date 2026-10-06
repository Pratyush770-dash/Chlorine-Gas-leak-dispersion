# Chlorine Gas Leak Dispersion: 2D CFD with ANSYS Fluent Species Transport

This is a CFD study in ANSYS Fluent of a small chlorine leak from a tank sitting on the ground. The question I wanted answered was simple: how far downwind does the gas stay above the 10 ppm IDLH limit?

I did this as a self-directed portfolio project. The idea came from the kind of chemical hazard work done around plants that handle gases like HF, Cl₂ and NH₃, where somebody has to figure out how far a leak can actually hurt people. I wanted to go through the whole thing myself, from drawing the geometry to reading concentrations against a safety limit, instead of following a tutorial. Chlorine is the only gas I've finished so far. HF and NH₃ are next on my list.

![mole fraction of Cl2 contour]()

Table of Contents
- [What Was Done](#what-was-actually-done)
- [Setup / Software Used](#setup--software-used)
- [Reproducing the Results](#reproducing-the-results)
- [Results](#results)
- [Method, Briefly](#method-briefly)
- [Limitations](#limitations)
- [Repository Structure](#repository-structure)
- [Author](#author)

## What was actually done

- Built a 2D domain, 100 m long and 20 m high, with a 1 m × 1 m tank on the ground about 20 m in from the inlet
- Cut a 0.01 m leak into the tank's right wall near the ground and let pure chlorine come out of it into a 3 m/s wind
- Solved steady species transport (chlorine and air) with realizable k-ε turbulence and gravity turned on, because chlorine is heavier than air and sinks towards the ground
- Put three receptor points at 1.5 m height, roughly where someone would be breathing, at 10, 30 and 60 m downwind of the tank
- Ran two leak rates. My first guess of 0.02 m/s was way too strong and put every receptor above the limit, so I dropped it to 0.006 m/s to bring the hazard boundary inside the domain
- Turned the mole fractions into ppm and compared them with the 10 ppm IDLH for chlorine
- Kept the mesh small at 11,286 elements

## Setup / Software Used

* ANSYS DesignModeler for the geometry
* ANSYS Meshing, quad-dominant method with edge sizing and bias
* ANSYS Fluent 2026 R1, Student version, for the solver and post-processing

## Reproducing the Results

1. Open ANSYS Workbench and add a Fluid Flow (Fluent) system
2. In DesignModeler, draw one closed outline on the XY plane: the 100 m × 20 m rectangle with the tank as a notch in the bottom edge, between x = 19 m and x = 20 m. Split the tank's right wall so a 0.01 m piece near the ground is its own edge, since that piece is the leak. Turn it into a surface with Concept → Surfaces From Sketches
3. Make the named selections `inlet`, `outlet`, `top`, `ground`, `tank_wall` and `leak`. Watch out: the tank roof belongs in `tank_wall`, not `top`. I got that wrong the first time
4. Mesh with the quad-dominant method and hard edge sizing, with bias towards the leak, the tank walls and the ground. Set the defeature size smaller than 0.01 m or the mesher may quietly remove the leak edge
5. In Fluent pick 2D and double precision, then switch on gravity (−9.81 m/s² in y) and the energy equation. Use realizable k-ε with standard wall functions and turn on species transport
6. Edit the mixture material so it holds cl2 and air, with air listed last, and set density to incompressible ideal gas
7. Boundary conditions: `inlet` is a velocity inlet at 3 m/s with cl2 mass fraction 0 and 5% turbulence intensity. `leak` is a velocity inlet at 0.006 m/s with cl2 mass fraction 1 at 300 K. `outlet` is a pressure outlet at 0 Pa gauge. `top` is symmetry. `ground` is a rough wall with roughness height 0.05 m and roughness constant 0.5. `tank_wall` is a smooth no-slip wall
8. Use the coupled solver with pseudo-transient and second-order upwind for momentum, turbulence, species and energy
9. Create point surfaces at (30, 1.5), (50, 1.5) and (80, 1.5), then add a Vertex Average report definition of cl2 mole fraction for each. I also added a mass flow report over `inlet`, `outlet` and `leak` to check that the balance closes
10. Turn the residual convergence criterion off and keep running until the three receptor plots stop moving. Fluent's default rule stopped my very first run at 151 iterations while the plume was still settling, so the residuals alone aren't a good sign here
11. Multiply the mole fractions by 10⁶ to get ppm

## Results

**Receptor concentrations at 1.5 m height**

| Receptor | x (m) | Case 1: leak 0.02 m/s | Case 2: leak 0.006 m/s |
|---|---|---|---|
| `rec_10m` | 30 | 67.1 ppm | 25.9 ppm |
| `rec_30m` | 50 | 41.5 ppm | 13.1 ppm |
| `rec_60m` | 80 | 28.5 ppm | 8.8 ppm |

The names are the distance downwind of the tank's right wall, which sits at x = 20 m.

In the first case every receptor is above the limit, so the hazard zone runs off the end of my domain. I mostly kept that run to show how much the answer depends on the leak rate.

In the second case the concentration is above IDLH at 10 m and 30 m and falls below it by 60 m. If I interpolate between the 30 m and 60 m points, the 10 ppm boundary lands at roughly **50 m downwind of the tank** at breathing height. With only three points that's a rough number, and I wouldn't trust it to better than about 10 m either way.

The contour plot is scaled from 0 to 1×10⁻⁵, so anything above IDLH shows up red, and near the ground it looks quite different. The red layer hugs the surface and stretches much further than 50 m, almost to the outlet. So how far the hazard reaches depends a lot on the height you measure at.

The net mass flow over inlet, outlet and leak settled at about zero. The receptor curves were flat for the last stretch of the run, which was around 250 iterations in total, and the residuals ended up near 1×10⁻⁴ or lower.

Contours, receptor plots and residual history are in `results/`.

## Method, briefly

- **Geometry:** A single 2D surface with the tank cut out as a notch. I originally wanted a 1000 m × 200 m domain with a 10 m tank, but Surface From Sketches kept throwing "Invalid profile selection" on that sketch no matter what I tried. When I divided every dimension by 10 it went through without trouble. That's why the model is 100 m × 20 m with a 1 m tank, and I treat it as its own scale instead of pretending it matches a real plant.
- **Mesh:** Quad-dominant, with ten edge sizing controls. There are a few cells across the leak, and the cells grow slowly away from the leak, the tank and the ground. 11,286 elements in total.
- **Solver:** Pressure-based, steady, 2D, double precision, coupled scheme with pseudo-transient, second-order upwind.
- **Turbulence:** Realizable k-ε with standard wall functions.
- **Concentration check:** Vertex-averaged cl2 mole fraction at three points, converted to ppm.
- **Leak rate:** The leak velocity is something I chose. It isn't taken from a real incident. I tuned it until the IDLH boundary fell inside the domain.

## Limitations

The model is 2D, so the 0.01 m leak is really a slot and the release is per metre of depth. A real plume would also spread sideways, and this one can't.

The simulation is steady, isothermal and single-phase. A real chlorine release from a pressurised tank usually flashes and forms aerosol, and I haven't modelled any of that. The wind is a uniform 3 m/s too, not a proper atmospheric boundary layer profile.

I didn't do a mesh independence study, and there's no experimental or benchmark data to validate against yet. So these numbers show that the method works, but they shouldn't be used as safe-distance values. The 50 m boundary comes from interpolating just three points.

What I plan to add: a log-law inlet profile, HF and NH₃ cases with their own IDLH limits (30 ppm and 300 ppm), a mesh independence check, and a comparison with a Gaussian plume estimate.

## Repository structure

```
├── README.md
├── geometry/
│   └── geometry project/
├── mesh/
│   └── Meshing project/
├── results/
│   ├── contours/            → Cl2 mole fraction contours
│   ├── receptor_plots/      → cl2_10m, cl2_30m, cl2_60m and net mass flow plots
│   └── residuals/           → scaled residual history
└── case_files/
    └── (Fluent case and data files)
```

## Author

**Pratyush Dash**

B.Tech Chemical Engineering, KIIT University, Bhubaneswar

