---
layout: page
title: "Enhancing theia -- Customizing the Output"
---

In this tutorial, we will learn how to customize the text output of theia (not the CAD output). There are two methods which we choose according to whether we want the information output to be the same in each simulation (method 1), or we want each simulation to be customizable by a separate configuration file (method 2).

Virtually, any information that is accessible to the objects (beams, optics, simulations, etc.) can be output, andit suffices to cha


### Method 1: permanent changes

For all beams and optics, the text output is systematically described by the `lines` method of objects. This method returns a list of string which are meant to be output in a curly-braces-structured paragraph, as explained in the very last paragraph of the User Guide. Each string in the list returned by the `lines` method will appear on a line in the text output, and they will be organized according to the braces ('{', '}') contained in the lines.

For example, here is a extract from the `lines` method of the `optics.beam.GaussianBeam` class:

```python
def lines(self):
        '''Returns the list of lines necessary to print the object.

        '''
        sph = geometry.rectToSph(self.Dir)
        sphx = geometry.rectToSph(self.U[0])
        sphy = geometry.rectToSph(self.U[1])

        return ["Beam: %s {" %self.Ref,
        "Power: %sW/Index: %s/Wavelength: %snm/Length: %sm" \
                % (str(self.P), str(self.N), str(self.Wl/nm), str(self.Length)),
        "Order: %s" %str(self.StrayOrder),
        "Origin: %s" %str(self.Pos),
        #...
        "Rayleigh: %sm" % str(self.rayleigh()),
        "ROC: " + str(self.ROC()),
        "}"]
```

In the text output, this produces output like so:

> Beam: LAS-tt {
> 	Power: 2.0W/Index: 1.0/Wavelength: 1064nm/Length: 1.0m
>	Order: 2
>	Origin: (0.2, 0.1, 0.0)
>	...
>	Rayleigh: 2.0m
>	ROC: 3.0m, 2.0m 
> }

