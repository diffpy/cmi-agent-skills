---
name: cmi-skill
description: write python code utilizing `diffpy.cmi` module to refine material structure models against experiment data. Used when researching the structure of materials with experimental pair-distribution-function(PDF).
---

# Introduction

For claude code agent: check [code-examples](code-examples.md) for example code snippets when needed.

# How to write a `diffpy.cmi` script for PDF (pair-distribution-function) refinement

1. write code to create a structural model.
2. refine the structural model against the experiment data.

# Advance: temperature/composition series

1. Parse out the temperature/Composition from the experiment data file names, or get it from a input variable. `e.g. exp_300K.dat`, `exp_0.2.dat`, ...
2. From low to high or high to low temperature/composition, for each PDF data, repeat all steps in [How to write a `diffpy.cmi` script for PDF refinement](#how-to-write-a-diffpycmi-script-for-pdf-pair-distribution-function-refinement) for each data obtained at the corresponding temperature/composition.
3. After each iteration, use the parameter value result as the initial guess for the enxt iteration.

# How to create the structural model

1. initialize profile.
2. initialize generator.
3. activate multiprocessing.
4. initialize contribution
5. initialize recipe
6. add usual variables
7. add constrained variables(spacegroup variables)
8. constrain the variables if necessary
9. unconstrain the variables if necessary
8. Set initial guess of the variables if necessary

# How to create the structural model when multiple phases are presented

Activate only when user input indicates multiple phases are presented.

Repeat the steps in [How to write a `diffpy.cmi` script for PDF refinement](#how-to-write-a-diffpycmi-script-for-pdf-pair-distribution-function-refinement), but

1. Modify the step2. Create an other PDFGenerator instances using the same pattern
2. Modify the step3. Parallel the other PDFGenerator instances under try block
3. Modify the step4. Add the other PDFGenerator instances into the FitContribution, and count them in the equation.
e.g. two phase: "s0*(s1*G1 + (1-s1)*G2)", three phases: "s0*(s1*G1 + s2*G2 + (1-s1-s2)*G3)", ...
4. Modify the step6. Create the parameters from other PDFGenerator instances.
5. Modify the step7. Impose the symmertry constraints and create parameters for other PDFGenerator instances.
6. After the step7

# How to create the structural model when nanoparticle phase is presented

Activate only when user input indicates nanoparticle phases are presented.

Repeat the steps in [How to write a `diffpy.cmi` script for PDF refinement](#how-to-write-a-diffpycmi-script-for-pdf-pair-distribution-function-refinement), but

When initialize contribution
1. Registre the spherical nanoparticle characteristic function.

When add usual variables in the `FitRecip` instance
1. Add spherical characteristic function variables in the `FitRecipe` instance.

After adding all variables in the `FitRecipe` instnace,
1. Restrain the `psize` parameter inside the characteristic function.


# How to create the structural model with isotropic ADP (atom displacement parameter) constrained separetely from spacegroup symmertry constraints

Activate only when user input indicates ADP parameters are isotropic and the same for the same elements.

Repeat the steps in [How to write a `diffpy.cmi` script for PDF refinement](#how-to-write-a-diffpycmi-script-for-pdf-pair-distribution-function-refinement), but

When add spacegroup variables in the recipe:
1. Ignore ADP parameters when impose group symmertry constraints
2. Constrain the adp from the same elements to be the same during the refinment.

# How to create the structural model with changed composition by inserting/replacing atoms in the specific sites

Activate when user need to change the input structure model by inserting/replacing atoms in the specific sites.

The ratio of the inserted/replaced atom, if is given, is set via the occupancy of the atom in the specific site.

Repeat the steps in [How to write a `diffpy.cmi` script for PDF refinement](#how-to-write-a-diffpycmi-script-for-pdf-pair-distribution-function-refinement), but

1. After loading the structure file, insert/replace atoms in the specific sites according to the composition. 
e.g.
```python
for atom in structure:
    if "<target-site-element>" in atom.element:
        stru1.addNewAtom(Atom("<new-element>", xyz=atom.xyz))
```
2. If the insert/replace ratio is given, after insert the atom, set the occupancy of the atoms in the specific sites. Atoms in the same sites should have the occupancy sum up to 1.
e.g. For two element site:
```python
for atom in structure:
    if "<target-site-element>" in atom.element:
        atom.occupancy = 1-new_occupancy
    if "<new-element>" in atom.element:
        atom.occupancy = new_occupancy
```
3. If the composition is a variable to be determined, create a corresponding variable in the recipe, and set the constraint mentioned above.
e.g. For two element site:
```python
new_occupancy = recipe.addVar("new_element_occupancy", value=0.5, bounds=(0,1))
for atom in recipe.<contribution_name>.<pdfgegnerator_name>.phase.atoms:
    if "<target-site-element>" in atom.element:
        recipe.constrain(atom.occupancy, "1.0 - new_element_occupancy")
    if "<new-element>" in atom.element:
        recipe.constrain(atom.occupancy, "new_element_occupancy")
```
and then refine the `new_element_occupancy` variable together with other variables in the recipe.

# How to unconstrain spacegroup parameters

Activate when user specifically asks to unconstrain spacegroup parameters to break the spacegroup symmertry constraints.

Currently supported parameters: a, b, c, alpha, beta, gamma, atom.x, atom.y, atom.z. 

If a parameter is not supported here:
1. Verify if it is a spacegroup parameter(lattice parameters, atoms positions, and atomicc displacement parameters).
2. Give instructions about how to unconstrain them.

If it is a spacegroup parameter:
1. Complete the script as usual
2. Add the following changes after ``constrainAsSpaceGroup`` is 
called:
   1. Let the 'ParameterSet' to which the parameter belongs constrain all the parameters.
   2. Then let the ``ParameterSet`` unconstrain the specific parameter.
   
You can 
1. constrain the lattice parameter set by `sgpars._constrainLattice()`
2. constrain the xyz parameter set by `sgpars._constrainXYZs()`
suppose `sgpars` is the object returned by `constrainAsSpacegroup`
   
a, b, c parameter is created as `pdfgenerator.phase.lattice.<parameter-name>` and belongs to the parameterset `pdfgenerator.phase.lattice`
xyz parametetr is created as `pdfgenerator.phase.getScatters()[<atom-index>].<parameter-name>` and belongs to the parameterset `pdfgenerator.phase.getScatters()`

