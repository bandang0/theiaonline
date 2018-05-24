---
layout: page
title: "The Implementation of Gaussian Optics in theia."
---

In this tutorial we will detail the implementation of Gaussian optics in theia. More precisely, we will tackle:
1. The description of the **general astigmatic Gaussian beam** and its free propagation, and of spherical surface optics,
2. The **search for interaction points** of beams on optics,
3. The calculation of the **characteristics of daughter beams** after interaction

For the physical formulae, we will follow Kochkina, Wanner, Schmelzer, Tröbs, Heinzel, 2013, see at the end.

### 1. Beams, free propagation and optics

#### a. The general astigmatic Gaussian beam

At an abscissa *z* along the Gaussian beam, the electric field is described in a set *(x, y, z)* of tranverse coordinates (with basis vectors *(Ux, Uy, Uz)*, *Uz* being the direction of propagation) by a **complex 2x2 tensor** (matrix), denoted by *Q*, the **complex curvature tensor**.

Neglecting the Gouy phase (as in theia) and setting the accumulative phase to 0., this electric field is given as in (6) (eq. 6 of Kochkina et al. 2013).

In theia, a `optics.beam.GaussianBeam` object only saves this matrix **at the origin of the beam** (the `np.array` attribute `QTens`) as well as the **basis of transverse vectors** (the `np.array` tuple attribute `U`) in which this `QTens` is expressed. All the Gaussianity is held by these two objects, and the game of this tutorial is to show how these are transformed by interactions. The rest of the interesting beam attributes are geometrical (direction, etc.)

From the `QTens` attribute, **all the useful Gaussian parameters can be calculated**, and that is the function of the following methods:
- `ROC`: radius of curvature along the two basis vectors at anydistance along the beam from the origin,
- `waistPos`: coordinates of waists along the beam,
- `rayleigh`: Rayleigh ranges in the two principal directions (given by the basis vectors),
- `waistSize`: transverse sizes of the waists.

#### b. Free propagation

With this description, the equation for free propagation is simple. You obtain the curvature tensor at a distance D along the beam from the origin by applying (9) to the origin curvature tensor.

This is exactly what is done in the `Q` method of `GaussianBeam`s.

#### c. Spherical surface optics

In theia, only **spherical** surface optics are simulated. They are regarded as cylinders, in which spherical surfaces have been carved (in or out) of the ends, and one of them possibly wedged (see section 2.3 of the User Guide).

The `Optics` object carries HR and AR face center points (`np.array` 3D vectors) and HR and AR normal vectors (same) **which always point to the outside** (regardless of whether the surfaces are convex or concave).

### 2. Finding interaction points

Given a beam and an optic, these are the steps to determine on which face and at which point a beams impacts the optic if it does (the methods described here are implemented in the `optics.optic.Optic` class):

1. Call the `isHitDics` method of the optic on the beam. This uses **geometrical functions** (`helpers.geometry.line{Plane,Surf,Cyl}Inter`) to determine for each of the (spherical) HR and AR face and the (cylindrical) side of the optic if a beam (with an origin and a direction) impacts it or not.

2. This is done within the `isHit` method, which after calling `isHitDics` chooses among the eventual impact points on the AR, HR and side of the optic **which point is hit first**. Then it returns the info on the closest impact point, the impacted face (HR, AR or side) and the distance from the origin of the beam to this impact point.

3. This is done within the `hit` method, which after calling `isHit` returns the daughter beams produced by the interaction at the location given by `isHit`. The description of how this is done is the object of the next paragraph. 

### 3. Calculation of daughter beams

For general optics (that produce reflected and transmitted beams), this `hit` method calls either `hitAR`, `hitHR` or `hitSide` according to whether the interaction happens on the AR, HR or side.

> Note: in the beam dump version of `hit`, no beam is calculated because all beams are stopped.

We will describe `hitHR` as an example.

It receives the beam, the interaction point, the simulation order and threshold, and returns a dictionary containing the reflected and transmitted beams (having calculated their curvature tensors, their directions, their transverse basis vectors, strayness order and power).

The physics behind the method implemented here (developed in Kochkina et al. 2013) is the **continuity of the phase of the beams at the impact point**, and assumes that the beam radii upon interaction are small compared to the surface curvature radii.

1. Determine the local norm to the surface at the impact point. This is the unitary vector pointing from the point to the center of the sphere which makes the surface. Notice that this is distinct from the HR norm, and that if these two point in the same direction, then the surface is concave (curvature > 0).

2. Determine the curvature to adopt. In the formulae whichwe will apply, the curvature's sign depends on whether the beam points in the same direction as the local norm or not:
```python
if np.dot(beam.Dir, localNorm) > 0.:
            localNorm = - localNorm
            K = np.abs(self.HRK)
        else:
            K = -np.abs(self.HRK)
```

3. Determine the optical indices in the media the beam is exiting and the one it is entering (`self` here designates the optic):
```python
# determine whether we're entering or exiting the substrate
        if np.dot(beam.Dir, self.HRNorm) < 0.:
            #entering
            n1 = beam.N
            n2 = self.N
        else:
            #exiting
            n1 = self.N
            n2 = 1.
```

4. Calculate the unitary directions of the reflected and transmitted beams (these are uniquely defined by the incoming direction, the local norm and the indices, see (47, 48)) using the `newDir` geometry function:
```python
# daughter directions
        dir2 = geometry.newDir(beam.Dir, localNorm, n1, n2)
```

5. `dir2` now contains the directions for the daughter beams. We can now (arbitrarely) choose for each daughter beam two unitary vectors, which will be their transverse basis vectors. The only requirement is that **these vectors along with the direction of the beam form a right-handed orthonormal basis** (RONB). This is done by the `basis` geometry function:
```python
# Calculate new basis
        if not 'r' in ans:   # for reflected
            Uxr, Uyr = geometry.basis(dir2['r'])
            Uzr = dir2['r']

        if not 't' in ans:   # for refracted
            Uxt, Uyt = geometry.basis(dir2['t'])
            Uzt = dir2['t']
```

6. As we will see, we will need an orthonormal basis at the impact point containing the local normal. Thus, we calculate these two vectors:
```python
Lx, Ly = geometry.basis(localNorm)
```

	Now, we have four RONB:
	- (Lx, Ly, localNorm),
	- (beam.U[0], beam.U[1], beam.Dir),
	- (Uxr, Uyr, Uzr),
	- (Uxt, Uyt, Uzt)

	And their equivalents in Kochkina et al. 2013:
	- (d1, d2, n)
	- (xi, yi, zi)
	- (xr, yr, zr)
	- (xt, yt, zt)

7. Determine the incoming curvature tensor at the point of impact:
```python
#distance from origin to impact
d = np.linalg.norm(point - beam.Pos)
Qi = beam.Q(d)
```

8. We need the local geometrical curvature matrix (curvature of the sphere). It is defined in (43), where *z'* is the **altitude function of the optical surface in local coordinates** *(x', y')* . In the case of a spherical surface, this matrix is the same everywhere:
```python
C = np.array([[K, 0.], [0, K]])
```

	In the case of more complex surface (paraboloids, etc.), this matrix will depend on where exactly you are on the surface. **This is the only sensitive place to change to extend theia to surfaces other than spheres.**

9. Then just apply (55, 56), where the *K*s are defined in (51), to calculate the curvature tensors of the daughter beams:
```python
# Calculate daughter curv tensors
        Ki = np.array([[np.dot(beam.U[0], Lx), np.dot(beam.U[0], Ly)],
                        [np.dot(beam.U[1], Lx), np.dot(beam.U[1], Ly)]])
        Kit = np.transpose(Ki)
        Xi = np.dot(np.dot(Kit, Qi), Ki)

        if not 't' in ans:
            Kt = np.array([[np.dot(Uxt, Lx), np.dot(Uxt, Ly)],
                        [np.dot(Uyt, Lx), np.dot(Uyt, Ly)]])
            Ktt = np.transpose(Kt)
            Ktinv = np.linalg.inv(Kt)
            Kttinv = np.linalg.inv(Ktt)
            Xt = (np.dot(localNorm, beam.Dir) -n2*np.dot(localNorm, Uzt)/n1) * C
            #here
            Qt = n1*np.dot(np.dot(Kttinv, Xi - Xt), Ktinv)/n2

        if not 'r' in ans:
            Kr = np.array([[np.dot(Uxr, Lx), np.dot(Uxr, Ly)],
                        [np.dot(Uyr, Lx), np.dot(Uyr, Ly)]])
            Krt = np.transpose(Kr)
            Krinv = np.linalg.inv(Kr)
            Krtinv = np.linalg.inv(Krt)
            Xr = (np.dot(localNorm, beam.Dir) - np.dot(localNorm, Uzr)) * C
            #here
            Qr = np.dot(np.dot(Krtinv, Xi - Xr), Krinv)
```

10. Finally, return the beams with these curvatures, directions, and tranverse basis (and update the power and order if necessary):
```python
# Create new beams
        if not 'r' in ans:
            ans['r'] = GaussianBeam(Q = Qr,
                    Pos = point, Dir = Uzr, Ux = Uxr, Uy = Uyr,
                    N = n1, Wl = beam.Wl, P = beam.P * self.HRr,
                    StrayOrder = beam.StrayOrder + self.RonHR,
                    Ref = beam.Ref + 'r',
                    Optic = self.Ref, Face = 'HR',
                    Length = 0., OptDist = 0.)

        if not 't' in ans:
            ans['t'] = GaussianBeam(Q = Qt, Pos = point,
                Dir = Uzt, Ux = Uxt, Uy = Uyt, N = n2, Wl = beam.Wl,
                P = beam.P * self.HRt,
                StrayOrder = beam.StrayOrder + self.TonHR,
                Ref = beam.Ref + 't', Optic = self.Ref, Face = 'HR',
                Length = 0., OptDist = 0.)

        return ans
```


Eventually, these beams are added to the recursive tree of beams and their daughter beams are computed.

Each optic can have its own implementation of `hit` and `hitHR`, etc. according to the effect it has on the beams. As long as these implementations respect the structure of the returned dictionary, there is no restriction on the optics one can simulate.

##### Bibliography

E. Kochkina, G. Wanner, D. Schmelzer, M. Tröbs, G. Heinzel, *Modeling of the General Astigmatic Gaussian Beam and Propagation through 3D Optical Systems*, Journal of Applied Optics 52, 2013