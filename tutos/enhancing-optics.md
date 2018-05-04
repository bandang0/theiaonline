---
layout: page
title: "Enhancing theia -- Adding Optics"
---

In this tutorial, we will enhance theia by adding an optical component to the library.

> Note: in this tutorial we will only consider optics with spherical surfaces.
> Adding e.g. parabolic optics will be the object of a later tutorial.


This will be done in three steps:
- Letting theia read the configuration of the new optic from the configuration file,
- Prescribing the optical transformations that the new optical component will perform on incoming beams,
- Describing how to represent the new component on the text and CAD outputs.

Before proceeding, we must decide what effect the optic will have on beams:
- What is the effect on the power of the reflected and transmitted beams? Does this change according to whether the beams impacts on the HR or AR surface?
- Do you consider the reflected and/or transmitted beams as stray light?

In our case, we will add a filtering component with a precise passing wavelength. This component will transmit all the incoming power, reflect none, without changing the direction of the beam, without changing the strayness order of the beam, at the condition that the wavelength of the beam matches that of the filter. In the contrary case, the component produces neither reflected nor transmitted beam (like a beam dump).

### 1: Letting theia know how to read the configuration of the new component

Choose a tag for the component, in our case "fl". This is the short string at the beginning of the configuration line of a filter, and which will be recognized by the parser.

Then, choose the configuration parameters and the order with which they will appear on the configuration line.

For a filter, we need precising the position (X, Y, Z), orientation (Theta, Phi), diameter and thickness (Diameter, Thickness), a passing wavelength (Wavelength), and finally a reference string (Ref). In a configuration line with 'fl' as tag (first word of the line), the parser will read all this info in the order specified in the `inOrder` dictionary of the `helper.settings` module.

To signify all of this to theia, add a element in the `inOrder` dictionary of the `helpers.setting` module, with 'fl' as key and the configuration strings in the right order as entry, like so:

```python
'fl': ['X','Y','Z','Theta','Phi','Diameter','Thickness', 'Wavelength', 'Ref']
```

We can now specify a filter in a conf file as:

> fl 1.2 3.0 0.0 Thickness=2*cm, Wavelength=800*nm

Finally, we must add the constructor (which we haven't written yet) to the parser. The filter class (see next paragraph) will eventually be `optics.filter.Filter`. We link the 'fl' tag to the `Filter` constructor by adding the tag to the `tags` list in the `running.parser.readIn` function, like so:

```python
tags = ['bm', 'mr', 'bs', 'sp', 'bo', 'th', 'tk', 'bd', 'fl', 'gh']
```

We link the constructor by adding the `Filter` constructor in the `constructors` dictionary of the `running.simulation.Simulation.load` function. Don't forget to import the constructor into the module:

```python
# this up in the module
from ..optics.filter import Filter

# in the load function

constructors = { ...
    'fl': Filter,
    ...}

```

### 2: Specifying the optical properties of the component

We will describe the optics of filters through a new class.

All optics inherit from the `optics.optic.Optic` class, which provides methods to determine whether the optic was hit by a beam, on which face, etc.

For our new filter component, we will write a new `optics.filter.Filter` class with:
- A constructor that will read only the information relevant to the filter : X, Y, Z, Theta, Phi, Diameter, Thickness, Wavelength, Ref and call the `optics.optic.Optic` constructor,
- Reimplement the `hit` method to detail what the filter does to beams (as described in the intro above)
 
