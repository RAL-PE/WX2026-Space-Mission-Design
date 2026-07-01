# GMAT Tutorial - Sentinel-4A GEO Transfer

## Introduction

The Sentinel 4A spacecraft is part of the EU Copernicus Earth Observation program. Sentinel-4A operates in Geostationary Earth Orbit (GEO) above the Prime Meridian. Sentinel-4A monitors of trace gas concentrations and aerosols in the atmosphere to track air quality across Europe in real time.

---

## 1. Spacecraft

`Sentinel4A` starts in a Low Earth Orbit, as GMAT cannot effectively model a launch from Earth's surface:

```
Create Spacecraft Sentinel4A;
Sentinel4A.DateFormat = UTCGregorian;
Sentinel4A.Epoch = '1 Jul 2025 21:04:28.000';
Sentinel4A.CoordinateSystem = EarthMJ2000Eq;
Sentinel4A.DisplayStateType = Keplerian;
Sentinel4A.SMA = 6600;
Sentinel4A.ECC = 0;
Sentinel4A.INC = 5.1597;
Sentinel4A.RAAN = -55;
Sentinel4A.AOP = 0;
Sentinel4A.TA = 0;
```

The above parameters match a post-launch parking orbit reachable from Kourou, French Guiana, at a 5 degree latitude.

> NOTE: In reality, Sentinel 4-A was launched from a Falcon 9 from Florida, directly toward it's GEO slot without using a parking orbit. But there's less to learn from that!

---

## 2. GroundStations

The ESA Space Operations Centre in Darmstadt, Germany is represented by `GSDarmstadt`:

```
Create GroundStation GSDarmstadt;
...
GSDarmstadt.StateType = Spherical;
GSDarmstadt.HorizonReference = Sphere;
GSDarmstadt.Location1 = 49.8692;
GSDarmstadt.Location2 = 8.6198;
...
GSDarmstadt.MinimumElevationAngle = 7;
```

`Location1` and `Location2` correspond to Darmstadt's latitude and longitude. The `MinimumElevationAngle` defines a 7 degree cone from the local horizon that excludes surface features like trees and mountains which could break a communication link.

---

## 3. ForceModels

Additional perturbations aside from Earth's gravity are included with solar radiation pressure:

```
Create ForceModel AllForces;
AllForces.CentralBody = Earth;
AllForces.PrimaryBodies = {Earth};
AllForces.PointMasses = {Jupiter, Luna, Mars, Sun, Venus};
AllForces.SRP = On;
...
```

These perturbations cause positional drift once `Sentinel4A` is in GEO, which must be detected and removed in real life "Station Keeping" maneuvers.

---

## 4. Propagators

A Prince–Dormand 7/8 integrator balances simulation accuracy with the long time steps needed to transfer to GEO:

```
Create Propagator EarthOrbitProp;
EarthOrbitProp.FM = AllForces;
EarthOrbitProp.Type = PrinceDormand78;
...
EarthOrbitProp.MaxStep = 24000;
...
```

The `AllForces` force model configured above is linked to the propagator and the max timestep set to 1/3 of a day, which is the same as 1/3 of an orbit in GEO.

---

## 5. Burns

Three impulsive burns are defined: `Transfer`, `Correction` and `Insertion`:

- The **Transfer** burn raises apogee to above GEO altitude
- The **Correction** burn shapes the transfer orbit, aligning it to the Equator
- The **Insertion** burn circularises the orbit at GEO

All burns use the default Earth `VNB` frame which links `Element1` to the negative radial and 'Element2' to the positive normal directions, using the right hand rule:

```
Create ImpulsiveBurn Correction;
...
Correction.Axes = VNB;
Correction.Element1 = 0;
Correction.Element2 = 0;
Correction.Element3 = 0;
...
```

> NOTE: The burns are initially assumed as zero or 'not needed' to allow GMAT to calculate the burn vectors magnitudes based on the simulation state later

---

## 6. EventLocators

The `ContactLocator` checks visibility between `Sentinel4A` and ESA's Space Operations Centre for uplinking the burn commands during the transfer and downlinking the science data from GEO:

```
Create ContactLocator ContactLocator;
ContactLocator.Target = Sentinel4A;
...
ContactLocator.Observers = {GSDarmstadt};
ContactLocator.LightTimeDirection = Transmit;
...
```

Simulations are core to ensuring ESA can see `Sentinel4A` for when its time to do the transfer burns in real life.

---

## 7. Solvers

GMAT uses the Newton–Raphson algorithm to iteratively change the burn vector magnitudes to achieve the transfer trajectory in a way that `Sentinel4A` arrives at GEO in the right longitude slot. The solver uses all default settings.

---

## 8. Subscribers

The `EarthOrbitView` provides a three views of `Sentinel4A` in the Earth's, Spacecraft's and Darmstadt's frame:

```
Create OpenFramesInterface EarthOrbitView;
...
EarthOrbitView.Add = {Sentinel4A, GSDarmstadt, Earth, Luna, Sun};
EarthOrbitView.View = {OrbitView, EarthView, StationView};
EarthOrbitView.CoordinateSystem = EarthMJ2000Eq;
...
```

`GroundTrackPlot` shows where `Sentinel4A` is in relation to Darmstadt, showing where it is likely to loose contact with ESA during the transfer to GEO:

```
Create GroundTrack GroundTrackPlot;
...
GroundTrackPlot.CentralBody = Earth;
GroundTrackPlot.Add = {GSDarmstadt, Sentinel4A};
...
```

---

## 9. Objects Views

Three camera views are configured:

- `OrbitView` provides a Equator-referenced isometric view of `Sentinel4A` from the sunlit side of Earth:

```
Create OpenFramesView OrbitView;
...
OrbitView.DefaultEye = [ -60000 -60000 60000 ];
...
```

- `EarthView` observes Earth from `Sentinel4A`:

```
Create OpenFramesView EarthView;
EarthView.ViewFrame = Sentinel4A;
...
EarthView.LookAtFrame = Earth;
...
EarthView.DefaultEye = [ 0 -7000 2000 ];
...
```

> NOTE: The `DefaultEye` pulls the camera behind and slightly above `Sentinel4A`, so the viewer can see the Earth too!

- `StationView` represents the perspective of communications antennae at Darmstadt:

```
Create OpenFramesView StationView;
StationView.ViewFrame = GSDarmstadt;
...
StationView.LookAtFrame = Sentinel4A;
...
StationView.DefaultEye = [ 0 -10 0 ];
...
```

> NOTE: The negative `DefaultEye` avoids the view clipping the surface of the Earth... sometimes...

---

## 10. Mission Sequence

GMAT is given a sequence of targeted maneuvers to reach GEO at the right location along the Equator:

```
Propagate 'Prop to Equator' EarthOrbitProp(Sentinel4A) {Sentinel4A.Z = 0...
...
   Vary 'Vary Prograde Burn' DC(Transfer.Element1 = 1...
...
   Propagate 'Prop To Apogee' EarthOrbitProp(Sentinel4A) {Sentinel4A.Apoapsis};
   Achieve 'Achieve RMAG' DC(Sentinel4A.RMAG = 45000, {Tolerance = .1});
...
```

The apoapsis is raised by the `Transfer` burn to produce an elliptical orbit that intersects with the satellite ring at GEO altitude (35,768 km). Burning at the the Equator crossing from a circular orbit guarantees that the highest point in the orbit will also be an Equator crossing.

Next, GMAT skips an orbit and waits until `Sentinel4A` reaches its apoapsis along with the Equator crossing:

```
Propagate 'Prop to Perigee' EarthOrbitProp(Sentinel4A) {Sentinel4A.Earth.Periapsis};
Propagate 'Prop to Equator Crossing' EarthOrbitProp(Sentinel4A) {Sentinel4A.Z = 0...
...
   Vary 'Vary Prograde Burn' DC(Correction.Element1 = 0.1...
   Vary 'Vary Normal Burn' DC(Correction.Element2 = 0.1...
...
   Achieve 'Achieve INC' DC(Sentinel4A.EarthMJ2000Eq.INC = 0, {Tolerance = .01});
   Achieve 'Achieve RMAG' DC(Sentinel4A.RMAG = 42195, {Tolerance = .001});
...
```

The `Change Plane/Perigee` sequence uses both prograde and normal burn components of the `Correction` burn to push the periapsis to match the GEO altitude (35,768 km) and the trajectory to track the Equator at its periapsis.

> NOTE: `RMAG` is measured from the centre of the Earth. For altitude, 6,378 km needs to be subtracted from `RMAG`.

The last transfer component is the `Correct Position` sequence to adjust the orbital longitude to match the Prime Meridian. Resetting the normal component in the `Correction` burn, GMAT finds the orbital trajectory needed:

```
Correction.Element2 = 0;
...
   Vary 'Vary Prograde Burn' DC(Correction.Element1 = 0...
...
   Achieve 'Achieve Longitude' DC(Sentinel4A.Earth.Longitude = 0, {Tolerance = 0.1});
...
```

The final sequence, `Lower Apogee`, performs GEO insertion. GMAT tunes the `Insertion` burn is until the orbital period exactly matches to a sidereal day (86164.09 s):

```
   Vary 'Vary Prograde Burn' DC(Insertion.Element1 = -0.1...
...
   Achieve 'Achieve Period' DC(Sentinel4A.Earth.OrbitPeriod = 86164.09...
...
```

> NOTE: A sidereal day is not the same as a solar day (86,400 s)! We need to track the Prime Meridian with respect to distant stars, not the Sun!

Lastly, the GMAT continues the simulation to allow analysis of station-keeping drift from the Prime Meridian at GEO and Darmstadt's visibility of `Sentinel4A`.
