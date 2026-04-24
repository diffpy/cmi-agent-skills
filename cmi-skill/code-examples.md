# initialize profile
<!--
description: Load experiment PDF data from profile_path as `Profile` instance.
-->
```python
    profile = Profile()
    parser = PDFParser()
    parser.parseString(Path(profile_path).read_text())
    profile.loadParsedData(parser)
    if qmin:
        profile.meta["qmin"] = qmin
    if qmax:
        profile.meta["qmax"] = qmax
    profile.setCalculationRange(xmin=xmin, xmax=xmax, dx=dx)
```

# initalize generator
<!--
description: Load structure model as `PDFGenerator` instance.
-->
```python
    stru_parser = getParser("cif")  # for cif file.
    structure = stru_parser.parse(structure_path.read_text())
    sg = getattr(stru_parser, "spacegroup", None)
    spacegroup = sg.short_name if sg is not None else "P1"
    pdfgenerator = PDFGenerator(f"G1")  # G1 as the name this PDFGenerator instance
    pdfgenerator.setStructure(structure)
```

# activate multiprocessing
<!--
description: Activate multi-processing for each `PDFGenerator` instance.
-->
```python
    if run_parallel:
        try:
            import multiprocessing
            from multiprocessing import Pool

            import psutil

            syst_cores = multiprocessing.cpu_count()
            cpu_percent = psutil.cpu_percent()
            avail_cores = numpy.floor((100 - cpu_percent) / (100.0 / syst_cores))
            ncpu = int(numpy.max([1, avail_cores]))
            pool = Pool(processes=ncpu)
            pdfgenerator.parallel(ncpu=ncpu, mapfunc=pool.map)
        except ImportError:
            warnings.warn(
                "\nYou don't appear to have the necessary packages for "
                "parallelization. Proceeding without parallelization."
            )
```
# initalize contribution
<!--
description: Create `FitContribution` instance to link `PDFGenerator` instance(s) and `Profile` instnace
-->
```python
    contribution = FitContribution("pdfcontribution")  # pdfgenerator as the name of this FitContribution instance.
    contribution.setProfile(profile)
    contribution.addProfileGenerator(pdfgenerator)
    contribution.setEquation("s1*G1")
```

# intialize recipe
<!--
Load all `FitContribution` instnaces into `FitRecipe` instnace
-->
```python
    recipe = FitRecipe()
    recipe.addContribution(contribution)
```

# add usual variables in the recipe
<!--
description: Create `delta1`, `delta2`, `s1`(the one appears in the contribution equation), `qdamp` and `qbroad` inside `FitRecipe` instance:
-->
```python
    delta1 = getattr(pdfgenerator, "delta1")
    delta2 = getattr(pdfgenerator, "delta2")
    s0 = getattr(contribution, "s0")
    qdamp = getattr(pdfgenerator, "qdamp")
    qbroad = getattr(pdfgenerator, "qbroad")
    recipe.addVar(delta1, name="delta1", fixed=False)
    recipe.addVar(delta2, name="delta2", fixed=False)
    recipe.addVar(s0, name="s0", fixed=False)
    recipe.addVar(qdamp, name="qdamp", fixed=False)
    recipe.addVar(qbroad, name="qbroad", fixed=False)
```
# add spacegroup related variables in the recipe
<!--
description: Impose spacegroup symmertry, and create the constrained lattice parameters, atomic displacement parameters, and atom position parameters inside `FitRecipe` instance
-->
```python
    tructure_parset = pdfgenerator.phase
    spacegroupparams = constrainAsSpaceGroup(structure_parset, spacegroup)
    for par in spacegroupparams.latpars:
        recipe.addVar(par, name=par.name, fixed=False)
    for par in spacegroupparams.xyzpars:
        recipe.addVar(par, name=par.name, fixed=False)
    for par in spacegroupparams.adppars:
        recipe.addVar(par, name=par.name, fixed=False)
```
# set initial guess for varialbes in the recipe
<!--
description: Set initial guess of the parameters if neccessary
-->
```python
parameter_obj = recipe._parameters.get(parameter_name)
parameter_obj.setValue(new_value)
```

# register nanocrystal characteristic funciton
<!--
description: create the spherical characteristic function named 'f1' with vriable `r` and `psize_1`, and apply the
effect to the `G1` term, the phase that contains the
nanoparticles.
-->

```python
    from diffpy.srfit.pdf.characteristicfunctions import sphericalCF
    contribution.registerFunction(sphericalCF, name="f1", argnames=['r', 'psize_1'])
    contribution.setEquation("s1*G1*f1")  # 'f1' times the term with `G1`, the name of the phase containning spherical nanocrystal
```

# add spherical characteristic function variables in the recipe

```python
    recipe.addVar(contribution.psize, PSIZE_I, tag="psize")
```

# restrain spherical characteristic function variables in the recipe
<!--
description: restrain the `psize_1` variable of the spherical characteristic function to larger than 0 and lower than 200.
-->

```python
    recipe.restrain("psize_1", lb=0.0, ub=200.0,scaled=True, sig=0.00001)
```

# add spacegroup variables but ignore ADP parameters
<!--
description: Ignore ADP parameters when impose group symmertry constraints
-->
```python
    structure.anistropy = False
    spacegroupparams = constrainAsSpaceGroup(
        pdfgenerator.phase, sg, constrainadps=False
    )
    for par in spacegroupparams.latpars:
        recipe.addVar(par, fixed=False, tag="lat")
    for par in spacegroupparams.xyzpars:
        recipe.addVar(par, fixed=False, tag="xyz")
```

# constrain the adp from the same elements to be the same
<!--
description: constrain the ADP parameters for the atoms belong to the same species to be the same along the refinement process.
-->

```python
    atoms = pdfgenerator.phase.getScatterers()
    species_to_uiso_var = {}
    for atom in atoms:
        uiso_var = species_to_uiso_var.get(atom.element, None)
        if uiso_var is None:
            uiso_var = recipe.newVar(f"{atom.element}_Uiso", tag='adp')
            species_to_uiso_var[atom.element] = uiso_var
        recipe.constrain(atom.Uiso, uiso_var)
```

# refine the parameters
<!--
description: refine the parameters contained in the `FitRecipe`
-->
refine one parameter
```python
    from scipy.optmize import least_squares
    recipe.free('<name-of-the-parameter-to-be-refined>')
    least_squares(recipe.residual, recipe.values)
```
or refine multiple parameter sequentially
```python
parameter_names = []  # e.g. ["delta2", "s0", ...]
for pname in parameter_names:
    recipe.free(pname)
    least_squares(recipe.residual, recipe.values)
```
