---
name: cluster-screening
description: Use DiffPy apps MCP tools to preview, estimate local computational cost, and screen bounded in-memory libraries of finite metallic clusters against a measured PDF.
---

# Cluster-screening workflow (exploratory)

Use the `diffpy.apps` MCP server. This skill chooses the *workflow*; the server performs ASE structure generation, a common PDF refinement for each candidate, and Rw ranking.

## Start from the user's goal

The user can say: "Screen Pd clusters across several families from 50 to 700 atoms, testing at most 12 candidate structures." Do not require the user to specify ASE integer parameters. Confirm a readable experimental G(r) path. First inspect its header/parsed PDF metadata for experimental `qmin` and `qmax`; if either is missing or ambiguous, ask the user to supply it rather than inferring it from the r grid or silently applying a generic Q default. Ask for other essential unknown settings before calibration or screening. A local input file in the working directory may supply the path.

Respect `max_clusters` as the number of candidate fits authorized across the entire request, **including timing-calibration fits**. The server enforces the limit per call; this skill must enforce it across calls. Never silently expand the authorized budget or repeat a calibration fit during the final screen. Request a new budget for additional fits.

## Select a sparse initial library

Call `generate_cluster_library` with `element`, `min_atoms`, `max_atoms`, `max_clusters`, and either `candidate_specs` or `motif_ranges`.

The currently supported builders are: `ico` with `noshells`, `oct` with `length` and `cutoff`, and `deca` with `p`, `q`, and `r`. The ideal morphologies include Mackay icosahedra; regular/truncated FCC octahedra and cuboctahedra; and regular, Ino-type, Marks, and Ino-truncated Marks decahedra. Other motifs such as singly twinned FCC are **not implemented**.

Build a *small* set of explicit `candidate_specs` that samples several relevant families and a spread of atom counts. ASE builder integers represent distinct structural topologies, not just size; avoid taking the first N combinations from a dense Cartesian grid. For decahedra, regular variants have `q=1, r=0`; Ino-like examples use `q>1, r=0`; Marks-like examples use `r>0` (including combined variants with `q>1`). In the FCC octahedral builder, `cutoff=0` yields regular octahedra and other cutoff/length combinations yield different truncations. Favor diversity over dense enumeration.

**Preview before refining:** inspect the returned `models` (including atom counts, motifs, and morphology) and `skipped` combinations. If coverage is poor or many models fall outside the requested range, regenerate a better sparse preview before any fit. Multiple cheap previews are fine. Preserve the complete, approved candidate specifications and their model IDs across calibration and final screening. **Always perform the computational preflight below before requesting approval to refine the library.**

## Choose one refinement stage

Provide one explicit refinement configuration to `screen_cluster_library`:

- `r_range`: `[rmin, rmax, rstep]` in angstroms. Ask for a preferred fit range; if unspecified, **propose** `1.5–20 Å` with `0.01 Å` steps as an adjustable starting choice, subject to the actual data range and user approval.
- `q_range`: `[qmin, qmax]` in inverse angstroms. **First use the experimental PDF reduction metadata** (from the parsed profile or file header) when present and unambiguous. Display the values and their provenance. If one or both are absent, **ask the user**; there is no universal fallback Q range, and the r-grid spacing does not determine these values.
- `qdamp`, `qbroad`: prefer user-supplied, experimentally appropriate values, ideally determined by refining a suitable bulk calibrant under comparable measurement conditions. They are **fixed instrument parameters** in the default ClusterMining recipe, not automatically chosen fit variables. If unavailable, ask the user and **offer** `qdamp=0.04 Å⁻¹` and `qbroad=0.01 Å⁻¹` as *provisional Pd demonstration values only*, requiring explicit approval before calibration or screening. Do not silently adopt these numbers or present them as universal beamline defaults.
- `initial_values`: numerical **starting guesses** for `scale`, `zoomscale`, `uiso`, and `delta2`. If the user provides none, propose the provisional starting guesses `scale=1.0`, `zoomscale=1.01`, `uiso=0.01`, `delta2=1.0`. These are freely refined starting values, not physical constraints or guaranteed-convergent defaults. Report them to the user and allow overrides before calibration. The proposed guesses have passed two small Pd benchmark checks but are not guaranteed to converge for other data.
- Optional `free_parameters`: any nonempty selection of the four preset aliases, registered native `cluster.` parameter names, or declared extra helper variables. The four preset aliases are used when omitted.
- Optional `soft_bounds`: supported by the current refinement engine. There are no hard bounds in this tool. If used, explain the penalty-strength choices.

The default recipe targets **monatomic finite clusters**, with a shared atomic Uiso and an isotropic geometry scale. Extensions must be requested explicitly; do not introduce additional degrees of freedom silently. Distinguish **fixed, experiment-specific** `qdamp`/`qbroad` from the **refined, user-adjustable** starting guesses for the four default fit variables. The proposed starting guesses may need adjustment for different experiments or normalization conventions; do not treat them as physically validated for every sample. Apply the *same* fit settings to each candidate for comparable ranking.

Call `screen_cluster_library(preview=<the entire successful preview>, pdf_path=<measured PDF>, refinement=<configuration>)`. The server regenerates the previewed ASE structures **in memory**, fits each through an isolated `RefinementSession`, and returns numerical results. Do not write custom fitting scripts or generate XYZ files unless separately asked.

## Mandatory computational preflight: preview, pair cost, and local calibration

**This is the default first stage for every screen, not an optional feature.** Do not go directly from a general user request to a large refinement. Keep the following stages separate and obtain authorization for the fits that use compute.

**A. Preview and deterministic cost (no fits):** Use the actual `models` in `generate_cluster_library`'s successful response. Print a compact morphology-grouped summary with the number of candidates and smallest/largest `n_atoms` for **each subfamily**, total candidates, full observed atom-count range, chosen `ncores`, and number of skipped candidates. Use meaningful subfamilies when builder parameters distinguish them: `ico` -> icosahedral; `oct` with `cutoff=0` -> FCC regular, with `length=2*cutoff+1` -> FCC cuboctahedral, otherwise FCC truncated; `deca` with `r>0` -> Marks, otherwise `q>1` -> Ino, otherwise regular. Do not assume unsupported motifs were sampled. Make the full per-model table (model ID, morphology, `n_atoms`, builder parameters) available on request.

For each candidate with `N_i` atoms, calculate its unique-pair-count proxy

`P_i = N_i * (N_i - 1) // 2`.

**Always show** `P_library = sum(P_i)` (units: unordered atom pairs per *one* PDF evaluation per candidate), and optionally `P_library / 185136` as the number of equivalent 609-atom PDF evaluations. Show the pair-count subtotal or fraction for each morphology subfamily so the dominant costs are visible. Calculate from *actual generated atom counts*, not `sum(N_i)**2`, an approximate advertised size, or an arbitrary candidate-count/size threshold. This is a first-order Debye pair-work proxy, **not** elapsed time, total pair evaluations over an entire refinement, or a memory estimate. Explain briefly that the cost of a full fit also depends on PDF evaluations during optimization, r-grid, implementation overhead, and CPU workers.

**MCP provenance:** Directly below the grouped preview, show one compact line, for example `MCP: diffpy_cluster_dev | called: diffpy_cluster_dev.generate_cluster_library | screening: not run`. Use the **actual registered server and tool names** observed in this session; after screening, also name the `screen_cluster_library` call. The installed skill guides the workflow, while the MCP tools execute the scientific Python code. Never claim a specific local checkout, interpreter path, or package version without verifying it. Keep the morphology preview as the primary output.

Present the complete grouped preview and pair-cost index **before any calibration fits**. Include a concise **refinement-settings summary** that separates (a) Q range obtained from file/profile metadata, or **missing—user input required**, (b) proposed r fit range if not supplied, (c) calibrated `qdamp`/`qbroad` or explicitly labeled provisional demonstration values awaiting approval, and (d) the four adjustable starting guesses and free parameters. Clearly identify which settings are confirmed and which are only proposed; **show the proposed values even when asking for missing inputs**, without silently using them. State clearly: `PREVIEW ONLY — no refinements yet`. Obtain confirmation of missing settings and ask for permission to time **2–3 actual candidate fits** on this computer using the intended experimental data, refinement settings, and worker count. If a single candidate is requested, explain that a separate representative calibration cannot estimate its full-fit time without effectively running it; do not fabricate an estimate.

**B. Local calibration (requires separate approval):** Choose 2–3 *existing* candidates spanning small/medium/large pair counts and preferably different subfamilies. Refine each **once** using `screen_cluster_library`, with the same `pdf_path`, `r_range`, `q_range`, fitting variables, optimizer options, and `ncores` planned for the full screen. Use new validated preview objects for the selected subset(s); do not alter the builder parameters or quietly add candidates. Calibration fits consume the user's total candidate-fit budget. Retain their full result rows and model IDs for inclusion in the final ranking. If a calibration fails, report it and treat its elapsed time as a limited/inconclusive timing sample.

Record for each successful calibration: `model_id`, `n_atoms`, `P_i`, actual `elapsed_seconds`, `nfev`, parameter count, morphology, and workers. The `execution.elapsed_seconds` may include multiprocessing startup; distinguish it from per-candidate fit times. A reported `nfev` does **not** necessarily count every underlying PDF evaluation (e.g., Jacobian work).

**C. Estimate remaining wall time (mandatory before full-screen approval):** Use the *measured* calibration times on this machine to establish a simple **nonnegative overhead + pair-cost trend**, for example `t_fit(P) ≈ a + b*P`, where `a >= 0` and `b >= 0`. An agent may compute this with ordinary Python arithmetic; no new DiffPy solver or permanent profiling service is needed. Estimate the remaining, untested candidates with `sum(a + b*P_i)` plus any expected startup/serialization overhead, and report measured calibration time separately. If the calibration times do not follow pair count, report that explicitly and use a wider, conservative estimate rather than force a misleading fit. Consider the observed spread in `nfev`, optimization convergence, family, r-grid, and any extrapolation beyond sampled sizes when giving a **rough lower–upper range**. Do not describe this as a statistical confidence interval or promise an exact finish time. Do not use a universal seconds-per-pair constant from a different computer or worker setting.

**D. Approval summary:** Display the grouped library preview, `P_library`, cost distribution, calibration measurements, fitted-time estimate/range, selected worker count, number of fits already completed, and number remaining. Explain that the timing estimate is based partly on `sum(N_i(N_i-1)/2)` and partly on locally timed full refinements; the unknown number of PDF evaluations per fit creates uncertainty. **Flag a compute warning based on the estimated wall time** (especially if it exceeds the user's expressed time/resource budget), not because a library crosses a fixed number of clusters or atoms. Ask for explicit approval before refining the remaining library.

**E. Screen without repeating calibration fits:** After approval, generate a validated preview containing *only the remaining model IDs/specifications*, refine those candidates with the identical settings, and merge their results with the saved calibration results for one ranking and one Rw-versus-size plot. The full authorized budget counts each candidate fit once. If keeping unique subsets or merging results is not possible, disclose that constraint **before** running more fits; never silently repeat the calibration models. Do not automatically restart a timed-out MCP screen, as the server may still be computing.

Example user-facing preflight (actual numbers must be calculated, not copied):

```text
ClusterMining preview (no refinements yet)
  icosahedral       : ... structures, ...–... atoms | ... atom pairs
  fcc_regular       : ... structures, ...–... atoms | ... atom pairs
  fcc_truncated     : ... structures, ...–... atoms | ... atom pairs
  fcc_cuboctahedral : ... structures, ...–... atoms | ... atom pairs
  deca_regular      : ... structures, ...–... atoms | ... atom pairs
  deca_ino          : ... structures, ...–... atoms | ... atom pairs
  deca_marks        : ... structures, ...–... atoms | ... atom pairs
  Total: ... candidates | pair-work index: ΣN(N−1)/2 = ... pairs
  Workers: ... | local timing calibration: requires approval
```

## MCP call sequence

The two MCP tools follow the same compact call-list format as the DiffPy playbook (arguments abbreviated):

```jsonc
generate_cluster_library(element="Pd", min_atoms=50, max_atoms=700,
                         max_clusters=12, candidate_specs=[...])

screen_cluster_library(preview=<approved subset preview>,
                       pdf_path="measured.gr", refinement=<confirmed settings>)
```

`screen_cluster_library` uses Yuchen's existing `RefinementSession` to fit and rank the candidates. Use separate calls for approved timing-calibration and remaining subsets, keeping completed results without repeating fits. No individual `add_pdf_model`/`solve` calls are needed.

## CPU execution and interpretation

`refinement.ncores` controls worker processes **within each Debye PDF calculation** and defaults to 1; clusters are refined sequentially. One pool is reused within an MCP screening call and closed afterward. Ask the user for an appropriate worker budget; never automatically use every core. More workers can be slower for small clusters; do not claim an N-fold speedup. Keep the same worker setting for calibration and the full screen.

The result contains `execution.elapsed_seconds` and per-candidate `elapsed_seconds` and `nfev`. Report ranked Rw, fitted parameters, model IDs, morphologies, atom counts, configuration, timing, failed/skipped models, and physically questionable parameter values. A low Rw does not establish a unique structure. A later coarse-to-fine screen must respect the remaining or newly authorized budget.

## Optional refinement extensions

Keep the four-variable recipe unless the user requests a different physical model. The screening function also accepts:

- `parameter_values`: initial/fixed values keyed by registered native names, such as `{"cluster.delta1": 0.1}`. To refine a native parameter, also include its name in `free_parameters`.
- `extra_variables`: initial values for named scalar helper variables.
- `constraints`: native target parameter names mapped to mathematical constraint expressions. These override matching default ties; `null` removes a default tie. They are not executable Python snippets.
- `optimizer`: optional native solver settings (`method`, `x_scale`, `max_nfev`, `ftol`, `xtol`).

### Optional directional geometry scaling (Cartesian-frame example)

The ASE-to-DiffPy conversion creates a finite `Structure` in the default orthogonal, unit-length coordinate basis. Its native `cluster.phase.lattice.a`, `.b`, and `.c` parameters **scale every atom's Cartesian x, y, and z positions**, respectively. These DiffPy `lattice` parameter names describe the *coordinate mapping* in this workflow, **not a periodic crystal unit cell or established crystallographic axes**. The default `zoomscale` ties all three to one common isotropic scale.

Only if the user explicitly requests axis-dependent geometry, the current interface can fit a separate scale along **one selected Cartesian axis** while keeping the other two tied together. For example, a z-axis stretch (x and y share `zoomscale`):

```json
{
  "extra_variables": {"stretch_z": 1.01},
  "constraints": {"cluster.phase.lattice.c": "stretch_z"},
  "free_parameters": ["scale", "zoomscale", "stretch_z", "uiso", "delta2"]
}
```

The analogous x- or y-axis example targets `cluster.phase.lattice.a` or `.b`. The resulting global affine, strain-like deformation is defined **relative to the ASE cluster's Cartesian orientation**. Do not call it strain along FCC [001] or another Miller-index direction without an independently established orientation; do not assume a unique crystalline frame for multiply twinned clusters. A truly **arbitrary off-axis direction** needs an explicit oriented basis or affine-coordinate transformation, which this wrapper does not currently expose. Do not silently introduce an extra geometry degree of freedom, and distinguish geometric scaling from anisotropic atomic displacement parameters.

Use the same chosen parameterization for all candidates in a ranking. When comparing different parameterizations, report the number of free parameters and treat a lower Rw with more degrees of freedom cautiously. Flag negative ADPs, negative delta2 values, nonfinite values, or other scientifically questionable parameters for review; solver convergence does not establish physical validity.

## Default presentation of screening results

Once a screen is complete and the user has authorized local analysis output, produce a **morphology-resolved Rw versus atom-count map** by default. Use the numerical results already returned by the MCP screen, including any retained calibration fits. Plot the **actual sampled models** only. Show unconnected scatter points, with distinct marker shapes, colors, and morphology labels for each subfamily. Do not draw connecting lines or guides to the eye, regardless of how many clusters were sampled. Use the following proven visual mapping from the Pd validation screen when the families appear: icosahedral red `#b9402b` (octagon), FCC regular green `#50956b` (diamond), FCC truncated light green `#80ae83` (circle), FCC cuboctahedral teal `#338a87` (thin diamond), deca regular dark blue `#145d87` (pentagon), deca Ino blue `#227a9b` (hexagon), deca Marks slate blue `#4b7f9c` (hexagon). Keep readable inward ticks, clean axes `Number of atoms, N_a` and `R_w`, and a compact legend. **Do not** mark any fixed atom count, circle a presumed winner in red, or add commentary/annotations inside the plot. Record fit settings and sampling limitations in the report or caption instead. Use `matplotlib` or another plotting tool already available; avoid imposing plotting dependencies on `diffpy.apps`.

The user also expects an experimental G(r), calculated G(r), and vertically offset difference plot of the best-fit candidate when possible. **Do not duplicate this scientific-output functionality inside the cluster-screening module.** Yuchen's general-purpose MCP methods already include `get_model_evaluation`, `get_profile_data`, and `get_model_residual`, but they operate on the **main server session**, whereas cluster screening uses temporary per-candidate sessions. Therefore those general methods cannot retrieve a finished screening model by name without first creating it in the main session. If the best model must be recreated/refit in the main session, say so and request approval for that extra work; preserve the exact settings and verify its Rw before publishing a fitted-PDF figure. Do not silently repeat an approved library fit, guess Gcalc arrays from a ranked result, or add specialized plotting code to `cluster_screen.py`. A screen map does not require this additional model recreation.

All analysis files produced by the agent belong in the **user-approved validation/output directory**, never in either source repository. Saving and plotting numerical results are analysis activities, not code-development requests. Do not edit source code, commit, or push as part of a scientific screening request.

## Save numerical results and a screen map when authorized

The returned results already contain the atom counts, morphology, Rw, settings, failure information, and timings needed for a screen map. With the user's permission to write analysis outputs, save the unmodified returned JSON and a CSV, then plot Rw versus atom count, distinguishing motif families. Use ordinary plotting tools; do not recompute fits or replace the MCP fitting workflow with a custom script. Keep failures in the saved table, even if they cannot appear as finite-Rw points. No XYZ export is performed by these tools.

State the sampled range, the actual number of fitted candidates, and the limits of sparse coverage. A known reference included deliberately is a validation benchmark, not independent rediscovery. Reproducing an historical Rw also requires matching data reduction, fit settings, and permitted parameter ranges.
