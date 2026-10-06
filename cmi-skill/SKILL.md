---
description: Playbook for using the diffpy_apps MCP server
---

# diffpy_apps MCP server: how to use it

## General Guidelines

1. Before starting, call `list_models()` / `list_profiles()` first (session is a shared global singleton, not per-conversation). Call `clear()` to reset.
2. Always ask if you are unsure about the refinement settings. Do not make any
   assumptions.

## MCP call-format rules

- **Always pass explicit `model_name`/`profile_name`.** `add_equation_model`/`add_pdf_model` default `model_name` to a UUID computed once at import time — every omitted call gets the _same_ value. `profile_name` defaults to `None` and gets echoed straight into the confirmation message.
- **`combine_models(parent_model_name, child_model_names)`: call once per child.** The child is registered into the parent's equation under its own model name. Only an equation model can be a parent.
- **`add_pdf_model(model_name=..., from_model_name=<existing>)` shares structure, not values.** Lattice/xyz/ADP(Uiso or Biso)/occupancy become the _same_ live object as the source — set them once, on the source, never per clone. Generator params (`scale`, `qdamp`, `qbroad`, `delta1`, `delta2`) are NOT shared — set each clone's separately or tie them with a `solve` constraint. Constrain symmetry on the source **before** cloning.
- **`solve`**: `profile_names`/`model_names`/`residual_equations`/`weights` must be equal length, one entry per contribution. `constraints` is exactly two dicts: `[0]` helper-variable -> initial value, `[1]` variable -> constraint-equation string. Pass `name=` to inspect later via `list_recipe_parameters`. `include_sgpars=true` auto-adds symmetry-freed structural params instead of listing each one.
- **Parsing results**: `check_profile_meta`/`list_profiles`/`list_models` -> plain JSON. `get_variable`/`list_model_parameters`/`list_recipe_parameters` -> formatted text (`"Variable 'x': 0.42"`), parse it yourself. `get_model_evaluation`/`get_model_residual` write a JSON array to `data_path`; `get_profile_data` writes `{"xobs":[...], "yobs":[...]}` to `data_path` (no `dyobs`/calculated curve). `solve` returns a fit-report string. Errors come back as `"{ExceptionType}: message"`.

## Case A — nested equation models (no structure, e.g. fitting `A*sin(a*x)`)

```jsonc
add_profile_from_arrays(xarray=[...], yarray=[...], profile_name="sine_profile")
add_equation_model(model_name="sub", equation_str="a*x")
add_equation_model(model_name="main", equation_str="A*sin(sub)")
combine_models(parent_model_name="main", child_model_names=["sub"])   // "sub" is registered under its own model name
set_variables_value(name_value_dict={"main.A": 0.8, "main.sub.a": 0.5})
solve(profile_names=["sine_profile"], model_names=["main"],
      variable_names=["main.A", "main.sub.a"], name="sine_fit")
get_variable(variable_name="main.A")
get_variable(variable_name="main.sub.a")
```

## Case B — single PDF model (e.g. Ni), optionally wrapped for a free scale

```jsonc
add_profile_from_file(profile_path="Ni.gr", profile_name="ni_profile")
update_profile_meta(profile_name="ni_profile", meta={"qmin": 0.1})
set_profile_calculation_range(profile_name="ni_profile", xmin=1.5, xmax=20, dx=0.01)

add_pdf_model(model_name="pdf", structure_file_path="Ni.cif")
constrain_pdf_model_space_group_symmetry(model_name="pdf")   // omit space_group to auto-detect
set_variables_value(name_value_dict={"pdf.scale": 0.4, "pdf.delta2": 2, "pdf.qdamp": 0.04, "pdf.qbroad": 0.02})
solve(profile_names=["ni_profile"], model_names=["pdf"],
      variable_names=["pdf.scale", "pdf.delta2", "pdf.qdamp", "pdf.qbroad"],
      include_sgpars=true, name="ni_fit")

// -- optional: promote to a free scale factor via a wrapping equation model --
set_variables_value(name_value_dict={"pdf.scale": 1})   // freeze pdf's own scale at 1
add_equation_model(model_name="ni_model", equation_str="s*pdf")
combine_models(parent_model_name="ni_model", child_model_names=["pdf"])
set_variables_value(name_value_dict={"ni_model.s": 0.4})
solve(profile_names=["ni_profile"], model_names=["ni_model"],
      variable_names=["ni_model.s", "pdf.delta2", "pdf.qdamp", "pdf.qbroad"],
      include_sgpars=true, name="ni_fit_scaled")
get_variable(variable_name="ni_model.pdf.phase.lattice.a")   // combined child addressed as parent.child.param
```

## Case C — multi-contribution joint refinement (Ni x-ray + Ni neutron + Si x-ray + mixed Si–Ni x-ray)

```jsonc
add_profile_from_file(profile_path="ni-q27r60-xray.gr", profile_name="ni_xray")
add_profile_from_file(profile_path="ni-q27r100-neutron.gr", profile_name="ni_neutron")
add_profile_from_file(profile_path="si-q27r60-xray.gr", profile_name="si_xray")
add_profile_from_file(profile_path="si90ni10-q27r60-xray.gr", profile_name="total_xray")
set_profile_calculation_range(profile_name="ni_xray", xmax=20)
set_profile_calculation_range(profile_name="ni_neutron", xmax=20)
set_profile_calculation_range(profile_name="si_xray", xmax=20)
set_profile_calculation_range(profile_name="total_xray", xmax=20)

add_pdf_model(model_name="pdf_ni", structure_file_path="Ni.cif")
constrain_pdf_model_space_group_symmetry(model_name="pdf_ni")   // constrain BEFORE cloning
add_pdf_model(model_name="pdf_ni_neutron", from_model_name="pdf_ni")
add_pdf_model(model_name="pdf_ni_partial", from_model_name="pdf_ni")

add_pdf_model(model_name="pdf_si", structure_file_path="Si.cif")
constrain_pdf_model_space_group_symmetry(model_name="pdf_si")
add_pdf_model(model_name="pdf_si_partial", from_model_name="pdf_si")

add_equation_model(model_name="main", equation_str="scale * (pdf_ni_partial + pdf_si_partial)")
combine_models(parent_model_name="main", child_model_names=["pdf_ni_partial"])
combine_models(parent_model_name="main", child_model_names=["pdf_si_partial"])

set_variables_value(name_value_dict={
  "pdf_ni.qdamp": 0.055, "pdf_ni_neutron.qdamp": 0.030, "pdf_ni_partial.qdamp": 0.052,
  "pdf_si.qdamp": 0.051, "pdf_si_partial.qdamp": 0.052,
  "main.scale": 1.0, "pdf_si.scale": 1.0, "pdf_ni.scale": 1.0
})

solve(
  profile_names=["ni_xray", "ni_neutron", "si_xray", "total_xray"],
  model_names=["pdf_ni", "pdf_ni_neutron", "pdf_si", "main"],
  residual_equations=["resv", "resv", "resv", "resv"],
  variable_names=[
    "pdf_ni.scale", "pdf_si.scale", "pdf_ni_neutron.scale", "main.scale", "pscale",
    "pdf_ni.phase.lattice.a", "pdf_ni.phase.Ni0.Uiso", "pdf_si.phase.a", "pdf_si.phase.Si.Biso",
    "ni_delta2", "si_delta2"
  ],
  constraints=[
    {"pscale": 0.8, "ni_delta2": 2.5, "si_delta2": 2.5},
    {
      "pdf_ni.delta2": "ni_delta2", "pdf_ni_neutron.delta2": "ni_delta2", "main.pdf_ni_partial.delta2": "ni_delta2",
      "pdf_si.delta2": "si_delta2", "main.pdf_si_partial.delta2": "si_delta2",
      "main.pdf_si_partial.scale": "1 - pscale", "main.pdf_ni_partial.scale": "pscale"
    }
  ],
  name="multi_fit"
)

get_variable(variable_name="pdf_ni.phase.lattice.a")
get_variable(variable_name="pdf_ni.phase.Ni0.Uiso")   // Biso equivalent = value * 8 * pi^2
get_variable(variable_name="pdf_si.phase.Si.Biso")
```

Notes specific to this case: `pdf_ni`/`pdf_ni_neutron`/`pdf_ni_partial` share one structure (clones), so only `pdf_ni.phase.*` needs to be in `variable_names`. `delta2` is generator-level and NOT shared, hence the explicit constraints tying every clone's `delta2` back to one free `ni_delta2`/`si_delta2`. `pscale`/`1 - pscale` splits the mixed `total_xray` contribution's intensity between the Ni and Si partials.
