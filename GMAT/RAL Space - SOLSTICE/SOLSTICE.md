# GMAT Tutorial - SOLSTICE Solar Occultation

## Introduction

SOLSTICE consists of three spacecraft flying in Low Earth Orbit (LEO) to observe the layers of the atmosphere during sunrises and sunsets, formally called occultation. By measuring IR light transmission through different layers of the atmosphere, the mission can infer atmospheric gas composition and layer structure. A trio of satellites allow measurements every 20 minutes at dawn or dusk for most locations on Earth.

---

## 1. User-Defined Celestial Bodies

The script begins by creating a planetary body named `Atmos` with a duplicate of Earth's SPICE kernel ID and frame, which locks it to Earth's location in space.

```
Create Planet Atmos;
Atmos.NAIFId = 399;
Atmos.SpiceFrameId = 'ITRF93';
...
Atmos.EquatorialRadius = 6458.1363;
...
```

`Atmos` is radially 80 km larger than Earth to represent the atmosphere layers for occultation analysis. All other data is duplicated from the built-in `CelestialBody` definition for Earth.

---

## 2. Spacecraft

Three spacecraft are defined: `SOLSTICE1`, `SOLSTICE2` and `SOLSTICE3`. All spacecraft start at same initial epoch of `23 Sep 2027 01:00:00.000` in the `EarthMJ2000Eq` coordinate system. The definitions for SOLSTICE1 show the initial orbit:

```
Create Spacecraft SOLSTICE1;
SOLSTICE1.DateFormat = UTCGregorian;
SOLSTICE1.Epoch = '23 Sep 2027 01:00:00.000';
SOLSTICE1.CoordinateSystem = EarthMJ2000Eq;
SOLSTICE1.DisplayStateType = ModifiedKeplerian;
SOLSTICE1.RadPer = 6936;
SOLSTICE1.RadApo = 6936;
SOLSTICE1.INC = 40;
SOLSTICE1.RAAN = 90;
SOLSTICE1.AOP = 0;
SOLSTICE1.TA = 0;
...
```

A circular LEO orbit at 40 degree inclination from the Equator provides coverage for any point within the +/- 40 latitude bound. `SOLSTICE1.RAAN = 90` orients the orbit to face the Sun, maximising the occultation time. All satellites are created at the same orbital location at the start of the simulation.

---

## 3. Formation

A 3-point constellation is assembled in the Formation section with `Formation1.Add = {SOLSTICE1, SOLSTICE2, SOLSTICE3}`. A `Formation` allows the entire constellation to maintain their relative positions as time advances in the simulation.

## 4. Ground Stations

Two ground stations are defined for communication and visibility analysis:

```
Create GroundStation GSChilbolton;
...
GSChilbolton.Location1 = 51.14578123015045;
GSChilbolton.Location2 = 358.5612293965133;
...
Create GroundStation GSAzores;
...
GSAzores.Location1 = 36.985;
GSAzores.Location2 = 334.874;
...
```

Their locations are specified using geographic latitude and longitude entered in the fields `Location1` and `Location2`. `Chilbolton` is RAL Space's satellite operations centre in England, while `Azores` is LeafSpace's relay used by OpenCosmos in the Atlantic Ocean.

---

## 5. Force Models

The model uses Earth as the primary gravitational body while also including lunar and solar perturbations with `PointMasses = {Luna, Sun}`. Solar radiation pressure is enabled using `SRP = On`. These perturbations increase realism but remain minor at the speeds `SOLSTICE1` and the other satellites orbit in LEO. All other settings are left on default.

---

## 6. Propagators

The propagator controls numerical integration of the spacecraft trajectories. The script creates `EarthOrbitProp` using default settings with the above `EarthOrbitProp_ForceModel` force model.

---

## 7. Coordinate Systems

A custom coordinate system called `SolarVector` is created to support occultation visualisation:

```
Create CoordinateSystem SolarVector;
SolarVector.Origin = SOLSTICE1;
SolarVector.Axes = ObjectReferenced;
...
SolarVector.Primary = SOLSTICE1;
SolarVector.Secondary = Sun;
```

This frame fixes the view from `SOLSTICE1` to always see the centre of the Sun as it moves in LEO, ensuring repeatable measurement of occultation lengths and timings.

---

## 8. Event Locators

Event locators use NASA's SPICE kernels to automatically calculate when and for how long ground stations and `CelestialBody` objects can "see" each other.

`OccultationLocator` is setup to identify occultation intervals using Earth's penumbra, the region when the Sun is partially blocked by the Earth's surface:

```
Create EclipseLocator OccultationLocator;
OccultationLocator.Spacecraft = SOLSTICE1;
...
OccultationLocator.OccultingBodies = {Earth};
...
OccultationLocator.StepSize = 1;
...
OccultationLocator.UseEntireInterval = true;
OccultationLocator.EclipseTypes = {'Penumbra'};
```

These times, coupled with the `SolarView` frame sunset/sunrise timings define the occultation length.

> NOTE: `Atmos` represents the Earth's atmosphere which isn't a SPICE object. Eclipse location in GMAT requires all involved bodies to have a SPICE kernel.

`ContactLocator` checks visibility between `SOLSTICE1` and the two ground stations for downlinking the science data:

```
Create ContactLocator ContactLocator;
ContactLocator.Target = SOLSTICE1;
...
ContactLocator.Observers = {GSChilbolton, GSAzores};
ContactLocator.LightTimeDirection = Transmit;
...
```

---

## 9. Subscribers

`SOLSTICESolarView` displays the Sun, `SOLSTICE1` (Any spacecraft is okay) and `Atmos`, which represents the Earth's atmospheric boundary to find occultation times:

```
Create OpenFramesInterface SOLSTICESolarView;
...
SOLSTICESolarView.Add = {SOLSTICE1, Sun, Atmos};
SOLSTICESolarView.View = {SolarView};
SOLSTICESolarView.CoordinateSystem = SolarVector;
...
```

The `EarthOrbitView` provides a wider field of view including all spacecraft, Earth, the Sun and ground stations:

```
Create OpenFramesInterface EarthOrbitView;
...
EarthOrbitView.Add = {SOLSTICE1, GSChilbolton, GSAzores, SOLSTICE2, SOLSTICE3, Earth, Sun};
EarthOrbitView.View = {OrbitView};
EarthOrbitView.Vector = {StationVector};
EarthOrbitView.CoordinateSystem = EarthMJ2000Eq;
...
```

`GroundStationTrackPlot` produces a default ground-track display showing spacecraft motion as seen from Earth's surface to prove the +/- 40 latitude bound is covered by the constellation. The ground stations are included to show that `SOLSTICE1` can see them, as visualised by the `StationVector` that is defined below.

> NOTE: `GSChilbolton` and `GSAzores` need to be added manually to `GroundStationTrackPlot`

---

## 10. User Objects

`SolarView` creates a spacecraft-centred camera directed toward the Sun:

```
Create OpenFramesView SolarView;
SolarView.ViewFrame = SOLSTICE1;
...
SolarView.InertialFrame = On;
SolarView.LookAtFrame = Sun;
...
SolarView.DefaultEye = [ 0 -1000 300 ];
...
```

> NOTE: The `DefaultEye` pulls the camera behind and slightly above `SOLSTICE1`, so the viewer can see the Sun too!

`OrbitView` provides a global isometric view of the systems from the sunlit side of Earth in the Equatorial reference frame:

```
Create OpenFramesView OrbitView;
OrbitView.ViewFrame = CoordinateSystem;
...
OrbitView.DefaultEye = [ 30000 -30000 30000 ];
...
```

The `StationVector` starts at `SOLSTICE1` and points to `GSAzores` using `VectorType = Relative Position`:

```
Create OpenFramesVector StationVector;
...
StationVector.SourceObject = SOLSTICE1;
StationVector.VectorType = Relative Position;
StationVector.DestinationObject = GSAzores;
...
StationVector.VectorLengthType = 'Manual';
StationVector.VectorLength = 300;
```
> NOTE: `VectorLength` needs to be increased manually to actually see it in `EarthOrbitView`

---

## 11. Mission Sequence

At simulation start, `SOLSTICE1`, `SOLSTICE2` and `SOLSTICE3` all have the same orbital locations, doubly ensured by copying the states. Then, `SOLSTICE2` is placed 120 degrees ahead of `SOLSTICE1`, while `SOLSTICE3` is positioned another 120 degrees forward:

```
SOLSTICE2 = SOLSTICE1;
SOLSTICE2.TA = SOLSTICE1.TA + 120;
SOLSTICE3 = SOLSTICE2;
SOLSTICE3.TA = SOLSTICE2.TA + 120;
Propagate 'Prop 2 Days' EarthOrbitProp(Formation1) {SOLSTICE1.ElapsedDays = 2};
```

Finally, the formation is propagated for two days to analyse orbital geometry, occultation opportunities and ground-station visibility over time.
