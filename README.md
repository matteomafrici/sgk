# SGK — an idea for a Space Propulsion Designer

This is a pre-design document for SGK (Space Generative Kernel) a small, open-source software experiment: a tool that takes an engine concept and carries it through the whole design chain — geometry, physics, optimization — across engine families: chemical, electric, nuclear.

The scope is quite broad at this stage. The full system does not exist yet; what exists is a plan, and a small validation slice — sgk-mini, described at the end — whose only job is to check whether the plan makes sense before investing in it.

## Design goals

These are targets, not achieved results:

- Push the software side of the design process as far as it goes, so that fewer physical tests are needed to close a design iteration.
- Replace routine CFD with a trained surrogate wherever the surrogate's accuracy is sufficient for the decision being made. CFD/FEM remains for dataset generation and final verification.
- Produce a metal 3D-printable (L-PBF) engine as the native output, not as an afterthought on top of a traditionally designed geometry. This is reachable with today's technology and liquid chemical propulsion.
- Keep the architecture family-agnostic from the start: adding an engine family means adding a physics module, not rewriting the pipeline.

## The four layers

The pre-design is organized in four layers, each depending only on the one before it:

| Layer | Function | Technology | Role in the pipeline |
|---|---|---|---|
| 1 — Geometry | Generate engine geometry from 0D/1D analytical physics | C# / .NET, PicoGK, LEAP71 ShapeKernel and LatticeLibrary | Turns physical relations into a voxel engine, ready for L-PBF printing (.vdb/.nvdb) |
| 2 — Physics surrogates | Fast neural inference on pressure, temperature, velocity and stress fields | Python, PyTorch, NVIDIA PhysicsNeMo, GINO (+ optionally PINO) | Replaces CFD/FEA in early design phases, including for geometries not seen in training |
| 3 — Generative optimization | Search the design space for better geometries against a stated objective | Bayesian optimization / genetic algorithms | Uses Layer 2 as a fast oracle, so thousands of variants can be evaluated without running real CFD |
| 4 — Conversational interface | Natural-language control of the design | LLM with function calling over a JSON schema | Translates commands into Layer 1 parameter changes, triggering regeneration and re-evaluation |

### Layer 1 — Geometry

The first layer generates the engine geometry from 0D/1D analytical physics — the standard toolbox of preliminary design: chamber sizing, nozzle contour, wall thickness, heat transfer. All closed-form relations, nothing exotic; the layer is meant to be simple, deterministic, and fast.

The geometry is a voxel field, not a CAD model, for a practical reason. A regeneratively-cooled liquid engine has a dense network of cooling channels — easily a hundred of them — and boolean operations on B-Rep kernels get unstable on that kind of topology. Voxel fields sidestep the problem: booleans become set operations on a grid. The geometric core builds on PicoGK, LEAP71's open-source voxel kernel (the same kernel behind their Noyron models), so the hard part is already done by someone else.

Manufacturability is part of the geometric computation itself: minimum wall thickness, self-supporting overhang angles and dimensional tolerances are imposed while generating the geometry, not checked afterwards on a finished model. The output is a voxel field (.vdb/.nvdb) meant to be printable directly.

The layer accepts two parallel input modes:

- **Mission-driven (high level):** delta-v, vehicle architecture, power budget, target mass, operating environment. Engine parameters are derived through inverse sizing.
- **Engine-driven (low level):** thrust, specific impulse, chamber pressure, oxidizer-to-fuel ratio, nozzle expansion ratio.

Both converge on the same `EngineConfig` schema, and the system translates between the two levels in both directions.

### Layer 2 — Physics surrogates

The second layer replaces routine CFD/FEM evaluation with a trained neural operator. The candidate model is GINO (Geometry-Informed Neural Operator, NVIDIA/Caltech, NeurIPS 2023) — from what I've read, it is not a single network but a three-stage pipeline:

1. A geometry encoder (local GNO): the engine surface is discretized into a point cloud; each point of a regular latent grid is connected through a graph to nearby points of the actual geometry within a radius, and a learned kernel aggregates local geometric information.
2. A global model (FNO blocks): the latent representation, concatenated with the signed distance function of the geometry, is processed by Fourier layers on a regular grid via FFT — capturing interactions along the whole engine (for example, pressure waves across the full nozzle length).
3. A geometry decoder (local GNO): the result is projected back onto the original surface points, returning pressure, temperature and stress fields exactly where they are needed.

Training follows the standard pattern of NVIDIA PhysicsNeMo (formerly Modulus): a dataloader returns (geometry as point cloud + SDF, ground-truth physical field) pairs; GINO is available as a ready-made model in the framework; the loop is standard PyTorch — MSE loss against CFD/FEM ground truth, Adam optimizer, learning-rate scheduling — with multi-GPU training, logging and checkpointing provided by the library.

One point that came up while researching this: fVDB is not a native component of GINO. GINO's published pipeline works on point clouds and SDFs with hash-grid neighbor search (Open3D CUDA); fVDB is a separate framework for differentiable sparse voxel representations. In SGK, its plausible role is a pre-processing step — extracting surfaces and SDF values from the high-resolution .vdb/.nvdb files produced by PicoGK, to feed GINO the input format it expects. I haven't found this specific combination documented publicly, so it is an open technical risk to prototype on a minimal case before building the full data pipeline.

PINO (Physics-Informed Neural Operator) is considered complementary rather than an alternative: GINO handles the geometric representation, PINO reduces the amount of CFD/FEM ground truth needed by adding physical constraints to the training loss.

The quantitative anchor for sizing the dataset is the published GINO reference — 500 automotive geometries (Ahmed-body dataset), 7.2 million mesh points each, Reynolds numbers up to 6.8 million, 8.31% error on the full pressure field, roughly 26,000× speedup over OpenFOAM for drag evaluation. Those numbers come from the paper and an unrelated geometry class; they are used here only to size the approach, not as results obtained for propulsion.

Layer 2/3/4 claims have not been measured on SGK's own geometry yet.

### Layer 3 — Generative optimization

The third layer does not replace Layer 1; it drives it:

1. It proposes a variation of Layer 1 parameters (for example, local nozzle contour curvature, cooling channel width) via Bayesian optimization or a genetic algorithm.
2. Layer 1 regenerates the corresponding voxel geometry — deterministic, in milliseconds.
3. The trained Layer 2 surrogate evaluates the physical performance of the new geometry in milliseconds, acting as a fast oracle in place of real CFD.
4. The response drives the next search direction, iterating to an optimum or a multi-objective Pareto front (for example, maximizing specific impulse while keeping peak wall temperature below a structural limit).

No real CFD runs inside the optimization loop itself; each evaluation costs a surrogate inference instead of hours of simulation. That is what makes exploring thousands of variants tractable.

### Layer 4 — Conversational interface

The fourth layer is an LLM with function calling over a JSON schema matching `EngineConfig`. It translates natural-language design commands into Layer 1 parameter changes, triggering geometry regeneration and, where relevant, re-evaluation through Layers 2 and 3. It carries no physics of its own — it is a controlled interface onto the deterministic core.

## Dataset strategy

The dataset strategy does not assume a large dataset is required. The GINO reference — under 10% pressure-field error on a complex industrial geometry class with 500 high-fidelity samples — suggests that for a geometry-aware neural operator, the distribution and quality of the samples matter more than the raw count.

The procedure has four steps:

1. **Define the parameter space.** Engine design variables — chamber pressure, oxidizer-to-fuel ratio, nozzle expansion ratio, cooling channel geometry and count, structural wall thicknesses — with physically admissible ranges.
2. **Sample with Latin Hypercube Sampling.** LHS covers a multidimensional parameter space uniformly with a controlled, minimal number of samples, avoiding redundant coverage of similar design points.
3. **Generate geometry for every sample via Layer 1.** Deterministic and milliseconds per sample, so it can run for thousands of parameter combinations at no meaningful cost.
4. **Run high-fidelity CFD/FEM only on the selected subset.** Only the LHS-selected samples are sent to CFD (OpenFOAM) and FEM for ground-truth generation.

Based on the GINO reference point, a realistic first estimate for a working Layer 2 on liquid chemical engines is **300–800 high-fidelity CFD/FEM simulations** even if that may still be too many for a single developer. This estimate will, of course, be revised when the first dataset is actually generated.

The ground truth of each sample includes more than a converged steady-state snapshot:

- pressure trace along chamber and nozzle, including transients;
- thrust trace, including startup/shutdown transients;
- wall temperature distribution, to capture hot spots and melting risk;
- regenerative cooling effectiveness per channel (flow and temperature), not an aggregate average;
- mechanical stress field (FEM), for structural margin under combined thermal and pressure load.

Crowdsourced simulation contribution is a possible path to scale beyond the internal dataset, but only after the first surrogate has been validated on an internally controlled dataset — with standardized containers (fixed mesh generation, solver version, boundary conditions) and an automatic validation pipeline (convergence checks, mass/energy balance checks) before contributed simulations are trusted as training data.

## Engine families

The shared infrastructure — PicoGK geometry, the surrogate pipeline, the conversational layer — is never rewritten when a new family is added. Every family implements the same abstract contract (`GenerateGeometry()`, `ComputeThermodynamics()`, `ExportVDB()`) from a common `EngineBase`; Layers 2 through 4 remain unchanged regardless of the family. Each family brings its own physics module, rather than force-fitting one from another domain.

The intended class structure:

```
EngineBase
├── ChemicalEngine
│   ├── LiquidEngine
│   │   ├── LiquidBipropEngine (pump-fed / pressure-fed)
│   │   └── LiquidMonopropEngine
│   ├── SolidEngine
│   └── HybridEngine
├── ElectricEngine
│   ├── HallEffectEngine
│   ├── IonEngine (grid-based)
│   ├── MPDEngine
│   └── MicrowaveElectrothermalEngine
├── NuclearThermalEngine (NTP)
├── NuclearElectricEngine (NEP)
└── FusionEngine
    ├── MagneticNozzleEngine
    └── PlasmaConfinementModule (MHD)
```

| Family | Dedicated physics module | Notes |
|---|---|---|
| Solid | Internal ballistics, grain surface regression | Reuses voxel geometry and the chamber/nozzle toolbox adapted to a solid chamber |
| Hybrid | Solid grain regression + liquid/gaseous oxidizer injection | Composition of the solid and liquid modules |
| Hall / ion | Child-Langmuir law, low-density plasma dynamics | Physically separate from neutral-gas flow; needs its own dataset and surrogate model |
| MPD | Magnetohydrodynamic equations (Lorentz force, applied current) | Conceptual precursor of the fusion module; partially reusable |
| Microwave electrothermal | Microwave plasma heating, expansion through a conventional nozzle | Closest bridge to chemical engines (continuous neutral/partially ionized flow) |
| NTP | Reactor-to-propellant heat exchange, then convergent-divergent expansion | Natural extension of the chemical equations with a different heat source |
| NEP | Reactor + electrical power conversion + downstream electric thruster | Composition of reactor and electric modules; no new nozzle physics |
| Fusion | Magnetic confinement, coil configuration, MHD | Most speculative and complex module; treated as its own domain with a dedicated dataset and training run |

## Development order

Development starts with liquid chemical propulsion, for a practical reason: it is a family complex enough that every step of the chain — analytical geometry, physical plausibility, surrogate accuracy — can be checked against public reference data. That makes it the environment where the pipeline gets proven end to end, before pointing it at families where the reference data is sparser and the physics harder to validate independently.

The concrete sequence is:

1. **sgk-mini** (in progress): a deliberately small companion project that validates the stack on a controlled case in order to correct/tune the SGK pre-design toolchain (see Status).
2. **First complete problem in SGK proper:** a small but complete liquid engine — a blowdown configuration, for example — simple enough to close the whole loop (geometry, physics, validation) end to end, and real enough to matter.
3. **Expand** the liquid family, then add families one at a time, each through its own physics module and its own validation.


## Status

This repository currently contains only this pre-design document. None of the four layers is implemented here.

End-to-end technical validation is carried out in a separate companion repository, **sgk-mini**, whose purpose is to check that the stack stands up on a small controlled case before investing engineering time in the full architecture. So far, sgk-mini has validated:

- PicoGK voxel geometry generation and round-trip serialization to .vdb on Fedora 44 / .NET 9, with PicoGK's native runtime built from source (no official Linux binary is distributed upstream);
- a minimal hollow-cylinder geometry to compute an annular water flow, with feature extraction and JSON round-tripping through independent code paths;
- an analytical physics benchmark: steady, fully developed laminar axial flow in a concentric annulus, with both global targets (mean velocity, Reynolds number, pressure gradient and drop) and local targets (velocity profile, wall shear stress);
- OpenFOAM `simpleFoam` validated against that analytical benchmark on two geometric configurations (narrow gap and wide gap), reaching: peak velocity (Umax) error 0.023%, pressure-gradient error 0.74%, flow-rate error 0.86%, and a mean near-wall profile error of 3.6%, attributed to second-order finite-volume discretization at this aspect ratio rather than to an implementation defect.

Still missing in sgk-mini: building a small (geometry, CFD field) dataset, training a first tiny PhysicsNeMo surrogate, testing its inference accuracy and speed against the analytical/CFD ground truth, and closing a minimal Layer 3 optimization loop.

Open design points, not yet finalized:

- the fVDB → SDF/point cloud → GINO integration, to be prototyped on a minimal case;
- the exact LHS parameter space and sampling resolution;
- the crowdsourcing quality-control pipeline;
- Layer 2 acceptance error thresholds;
- the public reference engine set for Layer 1 physical validation (NASA open data).

## License

Apache 2.0, chosen for consistency with PicoGK's own licensing.
