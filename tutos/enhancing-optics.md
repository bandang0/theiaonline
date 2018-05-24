---
layout: page
title: "Enhancing theia -- Adding Optics"
---

In this tutorial, we will enhance theia by **adding an optical component to the library**.

> Note: in this tutorial we will only consider optics with spherical surfaces.

> Adding e.g. parabolic optics will be the object of a later tutorial.

> The theia sources which incorporate the changes described in this tutorial can be found in the `enhancing-optics` branch of the GitHub repository. 


This will be done in three steps:
- Letting theia **read the configuration** of the new optic from the configuration file,
- Prescribing **the optical transformations** that the new optical component will perform on incoming beams,
- Describing **how to render the new component** on the text and CAD outputs.

Before proceeding, we must decide what effect the optic will have on beams:
- What is the effect on the power of the reflected and transmitted beams? Does this change according to whether the beams impacts on the HR or AR surface?
- Do you consider the reflected and/or transmitted beams as stray light?

In our case, we will add a **filtering component** with a precise passing wavelength. This component will transmit all the incoming power, reflect none, without changing the direction of the beam, without changing the strayness order of the beam, at the condition that the **wavelength of the beam matches that of the filter**. In the contrary case, the component produces neither reflected nor transmitted beam (like a beam dump).

### 1: Letting theia know how to read the configuration of the new component

Choose a tag for the component, in our case `fl`. This is the short string at the beginning of the configuration line of a filter, and which will be recognized by the parser.

Then, choose the configuration **parameters** and the **order** in which they will appear on the configuration line.

For a filter, we need to provide the position (X, Y, Z), orientation (Theta, Phi), diameter and thickness (Diameter, Thickness), a passing wavelength (Wl), and finally a reference string (Ref). In a configuration line with 'fl' as tag (first word of the line), the parser will read all this info in the order specified in the `inOrder` dictionary of the `helper.settings` module.

To signify all of this to theia, add a element in the `inOrder` dictionary of the `helpers.setting` module, with 'fl' as key and the configuration strings in the **order you want them to be parsed**, like so:

```python
'fl': ['X', 'Y', 'Z', 'Theta', 'Phi', 'Diameter', 'Thickness', 'Wl', 'Ref']
```

We can now specify a filter in a configuration file as:
```C
fl 1.2, 3.0, 0.0, Thickness=2*cm, Wl=800*nm
```

Finally, we must add the constructor (which we haven't written yet) to the parser. The `Filter` class (see next paragraph) will eventually be `optics.filter.Filter`. We link the 'fl' tag to the `Filter` constructor by adding the tag to the `tags` list in the `running.parser.readIn` function, like so:

```python
tags = ['bm', 'mr', 'bs', 'sp', 'bo', 'th', 'tk', 'bd', 'gh', 'fl']
```

We **link the constructor** by adding the `Filter` constructor in the `constructors` dictionary of the `running.simulation.Simulation.load` function. Don't forget to import the constructor into the module:

```python
# this in beginnig of the module
from ..optics.filter import Filter

# in the load function

constructors = { ...
    'fl': Filter,
    ...}

```

### 2: Specifying the optical properties of the component

We will describe the optics of filters through a **new class**.

All optics inherit from the `optics.optic.Optic` class, which provides methods to determine whether the optic was hit by a beam, on which face, what the daughter beams characteristics are, etc.

For our new filter component, we will write a new `optics.filter.Filter` class with:
- A **constructor that will read only the information relevant to the filter**: X, Y, Z, Theta, Phi, Diameter, Thickness, Wavelength, Ref and call the `optics.optic.Optic` mother constructor,
- **Reimplement** the `hit` method to detail what the filter does to beams (as described in the intro above).

All of the material we will describe here will go into a new file `optics/filter.py`

> Note: it is recommended when changing the sources of theia to respect the coding guidelines. The are briefly described in the last paragraph "Miscellaneous remarks" of the User Guide.

> For a typical optical component file, you can take example on the 'optics/mirror.py' file.

#### a. Class inheritance and attributes

All active optical component classes inherit from the `optics.optic.Optic` class. Furthermore, for printing purposes, optical component classes all have a `Name` class attribute. You can go ahead and start the file with:

```python
'''Defines the Filter class for theia.'''

import numpy as np
from ..helpers import geometry, settings
from ..helpers.units import cm, nm, deg, pi
from .optic import Optic
from .beam import GaussianBeam # need this for the hit method

class Filter(Optic):
	'''Write some documentation!
	   Take inspiration on the Mirror class file.
	'''

	Name = "Filter"
```

#### b. The constructor

The contructor will be exactly like that of the `Mirror` class, except that:

1. The input parameters are not the same as for mirrors, here we require X, Y, Z,Theta, Phi, Diameter, Thickness, Wavelength and Reference. Thus the signature of the constructor is:
```python
	def __init__(self, X = 0., Y = 0., Z = 0., Theta = pi / 2.,
			Phi = 0., Diameter = 1.e-1, Thickness = 2.e-2,
			Wl = 800 * nm, Ref = None):
```
And you have the liberty to **specify any default values** for the input parameters.
2. Regardless of whether the beam incides on HR or AR face, the beam strayness is not increased, thus we have:
```python
		# actions
		TonHR = 0
		RonHR = 0
		TonAR = 0
		RonAR = 0
```
You then have to initialize the types (for safety) of the user-provided data:
```python
		Theta = float(Theta)
		Phi = float(Phi)
		Diameter = float(Diameter)
		Thickness = float(Thickness)
		Wl = float(Wl)
```
And finally calculate the AR and HR face centers and normal vectors for the upcoming `Optic` constructor. This is the same as in the `Mirror` class, except that the Wedge is 0. here:
```python
		HRNorm = np.array([np.sin(Theta) * np.cos(Phi),
				 np.sin(Theta) * np.sin(Phi),
				 np.cos(Theta)], dtype = np.float64)

		HRCenter = np.array([X, Y, Z], dtype = np.float64)

		#Calculate ARCenter and ARNorm with thickness:
		ARCenter = HRCenter - Thickness * HRNorm
		ARNorm = - HRNorm
```
3. The `Optic` constructor is called with different values for the optical parameters (curvature is 0., index is 1. etc.)

	```python
	    super(Filter, self).__init__(ARCenter = ARCenter, ARNorm = ARNorm,
					N = 1., HRK = 0., ARK = 0., ARr = 0., ARt = 1., HRr = 0., HRt = 1.,
					KeepI = True, HRCenter = HRCenter, HRNorm = HRNorm,
					Thickness = Thickness, Diameter = Diameter,
					Wedge = 0., Alpha = 0.,
					TonHR = TonHR, RonHR = RonHR, TonAR = TonAR, RonAR = RonAR,
					Ref = Ref)
		#Warnings for console output
		if settings.warning:
			self.geoCheck("filter")
	```
	> Note: in this example, we have set the reflection and transmission coefficients to 0. or 1., which is arbitrary because in the end (next paragraph), we will compare the beam wavelength to that of the filter to determine whether the power is transmitted.
4. Filters also have the Wl attribute, which we add:

	```python
			self.Wl = Wl
	```

#### b. Printing function

Though is not yet used in the actual version of theia, it can be useful to reimplement the `lines` method. As described in the [*Enhancing theia's Output*](enhancing-output.html) tutorial, this function returns the lines to text-print information on the optic. Here is a decent example:

```python
	def lines(self):
		'''Returns the list of lines necessary to print the object'''
		sph = geometry.rectToSph(self.HRNorm)

		return ["Filter: %s {" %str(self.Ref),
			"Wavelength: %snm" %str(self.Wl/nm),
			"Thick: %scm" %str(self.Thick/cm),
			"Diameter: %scm" %str(self.Dia/cm),
			"HRCenter: %s" %str(self.HRCenter),
			"HRNorm: (%s, %s)deg" % (str(sph[0]/deg), str(sph[1]/deg)),
			"}"]
```

#### c. Specifying the effect of filters on beams

Here is the interesting part. We reimplement the `optics.optic.Optic.hit` method. Which is called on a beam if it hits the optic and does two things:

1. **Updates this (incident) beam's attributes** (length, target optic, width at the end of the beam) now that we know where the beam terminates,

2. **Calculates the daughter beams** emerging from the interaction of the beam with the optic (reflected and refracted), and returns them as a dictionary `{'r': ..., 't': ...}`.

In the usual case, the daughter beams are calculated using the `hitAR`, `hitHR`, `hitSide` methods, and the daughter beam curvatures are calculated there (see the [*Implementation of Gaussian Optics in theia*](optics-implementation.html) tutorial for details on these). But in our case, the beams are just the continuation of the input beams, if the wavelengths match. Thus, is goes like this:

> Note: the order and threshold parameters are order and threshold limits for daughter beams, and have to be given to the method in order for our reimplementation to integrate well with the rest of theia, even though they have no influence in the case of filters.


```python
	def hit(self, beam, order, threshold):
		'''Write some info!
		'''
		# get impact parameters and update beam
		# this is present in all the hit implementation and
		# never changes
		# it updates the data of the beam now we know where is ends
		dic = self.isHit(beam)
		beam.Length = dic['distance']
		beam.OptDist = beam.N * beam.Length
		beam.TargetOptic = self.Ref
		beam.TargetFace = dic['face']
		endSize = beam.width(beam.Length)
		beam.TWx = endSize[0]
		beam.TWy = endSize[1]
```

Now we return the dictionary of daughter beams. The reflected is None. The transmitted is None if the wavelengths do not match, and if they do, then it is a beam with:
- same direction as incoming beam,
- initial position on the impact point,
- curvature, the value of that of the incoming beam at its end (the impact point). 

The info on the face that is hit and the impact point is held in `dic`, returned by the `isHit` method. The new beams are created by their constructor:

```python
		if self.Wl != beam.Wl or dic['face'] == 'Side':
			return {'r': None, 't': None}

		return {'r': None,
			't': GaussianBeam(Q = beam.Q(beam.Length), #curvature
				Pos = dic['intersection point'], # position
				Dir = beam.Dir, Ux = beam.U[0], Uy = beam.U[1],
				N = beam.N, Wl = beam.Wl, P = beam.P, StrayOrder = beam.StrayOrder,
				Ref = beam.Ref + 't', Optic = self.Ref, Face = dic['face'],
				Length = 0., OptDist = 0.)}

```

We may as well have a band-pass filter, as so:

```python
		# band-pass transfer function
		powerTransfer = 1 / (1 + ((beam.Wl - self.Wl) / self.Wl) ** 2)

		if powerTransfer * beam.P < threshold:
			# under threshold, no daughter transmitted beam
			return {'r': None, 't': None}

		return {'r': None,
			't': GaussianBeam(...,
					P = beam.P * powerTransfer ,
					...)}
```

> Note: it is through this `hit` method that all the optics is done, and every optical component can have its own reimplementation, according to the physics you want to simulate.

### 3. Rendering the optics

The optics are rendered in 3D through a **FreeCAD shape** and with **features**. These features are the information which will appear in the left panel of the FreeCAD viewer when a `.fcstd` file produced by theia is read with FreeCAD. Both the shape and the features are contained in a **FreeCAD object**, which is created during the writing of the FreeCAD file.

#### a. Triggering the rendering

To an optic is associated a `Name` and a FreeCAD object constructor. Add these in the `FCDic` dictionary of the writeToCAD function of the `rendering.writer` module. The constructor is not written yet, but it will be named `FCFilter`:

```python
# first import the constructor
from .features import FCFilter

def writeToCAD(component, doc):
	...
	FCDic = {...,
		'Filter': FCFilter}
```

Also, add the name `'Filter'` to the list of optics which will be rendered right after this:

```python
		if component.Name in [..., 'Filter']:
			...
```

#### b. The 3D shape of the component

The function to generate FreeCAD shapes are all in the `rendering.shapes` module, and **each optic has its own**. These function use the FreeCAD `Part` API. To learn how to use it please refer to the FreeCAD dcoumetation on this [here](https://www.freecadweb.org/wiki/index.php?title=Part_API) and [there](https://www.freecadweb.org/wiki/Part_Module).

All these functions do is **generate shapes** using the `Part` API and **return them**.

In our case, we render a filter with a cylinder the same size and orientation as the filter, as so:

```python
def filterShape(fil):
    '''
    Returns a filter shape for a CAD file object.

    '''

    # this factor is just to convert milimeters (FreeCAD) to meters
    fact = settings.FCFactor    #factor for units in CAD
    return Part.makeCylinder((fil.Dia/2.)/fact, fil.Thick/fact,
                                Base.Vector(0,0,0),
                                Base.Vector(tuple(-fil.HRNorm)))
```

If you get good at FreeCAD shapes API manipulation, you can render the optics with more artistry. The shape of beams (`rendering.shapes.beamShape`) is more elaborate and thus longer to render for FreeCAD in the end.

#### c. The FreeCAD features

To **bundle up the shape and the features in a FreeCAD object**, add a class (the constructor of which you triggered in paragraph *a.*) to the `rendering.features` module. It must inherit from the `FCObject` class and the constructor has to:

1. **Call the mother class** constructor on the filter object,
	```python
	from .shapes import filterShape
	from ..helpers.units import nm
	#...
	class FCFilter(FCObject):
		def __init__(self, obj, fil):
			super(FCFilter, self).__init__(obj)
	```
2. **Associate the shape** from paragraph *b.* to the object's `Shape` attribute,
	```python
			obj.Shape = filterShape(fil)
	```
3. **List all the features** you want to have in the FreeCAD panel when the optic is clicked on in the viewer. The syntax for these calls is to call the `addProperty` method on the object, with arguments:
- the property type (e.g. "App::PropertyString"). The list of available types is [here](https://www.freecadweb.org/wiki/Scripted_objects). They basically differ in the number of digits and the units shown. "App::PropertyDistance" is good for lengths
- the string used to refer to the property (e.g. "Dia")
- the name of the optic (e.g. "Filter")
- the sentence shown when hovering over the property on the 3D viewer (e.g. "Diameter of the filter")
	
	And finally to call the string used to refer to the property (e.g. "Dia") and assign the value. Lets put in the filter diameter, thickness and wavelength:
	```python
			obj.addProperty("App::PropertyDistance", "Thick", "Filter",
				"Thickness of filter").Thick = fil.Thick/self.fact
			obj.addProperty("App::PropertyDistance", "Dia", "Filter",
				"Diameter").Dia = fil.Dia/self.fact
			obj.addProperty("App::PropertyString", "Wl", "Filter",
				"Passing wavelength").Wl = str(fil.Wl/nm) + 'nm'
	```
4. Place the shape at the right position in the 3D scene,
	```python
			obj.Placement.Base = Base.Vector(tuple(fil.HRCenter/self.fact))
	```

The filter optical component is now all set, it will be parsed, simulated and 3D rendered. The theia version incorporating the filter component can be found in the `enhancing-optics` branch of the GitHub repo, and tested with the `tutos/filter.tia` tutorial file therein.