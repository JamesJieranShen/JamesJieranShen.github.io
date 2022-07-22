---
layout: post
title:  "Two bugs in SNO+ Geometry"
date:   2022-07-18 00:00:00 -0400
categories: chroma, sno+, geant4
---

# Description

Weird Belly Plates are seen in the geometry viewer for ROOT:

![root viewer](/assets/snoplus_geo_bug/snoplus_bug.png)

Reason of this seems to be that translation is incorrectly done twice.

Plate is at position (X, Y, Z) = (6000, 0, 0) mm. A translation is done  (X-1149.6, Y+3545.81) to move the plate to a $$\phi$$ =
36$$^{\circ}$$.  A rotation of 36 deg is separately specified. GDML standard procedure is rotation first, translation
second. This means that the translation will move the plate to the wrong place.

## Rat Code

In [`src/geo/solids/BellyPlate.cc`](https://github.com/snoplus/rat/blob/ba923dbacc8fd51475d47cbaeb61af31a540c868/src/geo/solids/BellyPlate.cc#L48-L78), belly plate is returned as a `G4DisplacedSolid`:
```c++
G4VSolid*
BellyPlate::ConstructBellyPlate( const std::string& name,
                                 const double halfHeight,
                                 const double rPolyNominal,
                                 const double rPolyInner,
                                 const double rPolyOuter,
                                 const double rSphereInner,
                                 const double rSphereOuter,
                                 const double offset ) const
{
  // Logic:
  // The belly plate is an intersection of two polycones and a sphere.
  // The two polycones create the conal edges of the belly plate whilst the sphere creates the face.

  double z[] = { -halfHeight, 0.0, halfHeight};
  double rInner[] = { rPolyNominal, rPolyInner, rPolyNominal };
  double rOuter[] = { rPolyNominal, rPolyOuter, rPolyNominal };

  G4Sphere* sphere = new G4Sphere( name + "_sphere_solid", rSphereInner, rSphereOuter, -pi / 10.0, pi / 5.0, 4.0 * pi / 10.0, pi / 5.0 );
  G4Polycone* polycone = new G4Polycone( name + "_polycone_solid", -pi / 10.0, pi / 5.0, 3, z, rInner, rOuter );
  // The first step is too create two polycones : horizontal and vertical
  G4RotationMatrix* rotation = new G4RotationMatrix();
  rotation->rotateX( pi/2.0 );
  G4IntersectionSolid* polycones = new G4IntersectionSolid( name + "_intersected_polycones_solid", polycone, polycone, rotation, G4ThreeVector( 0.0, 0.0, 0.0 ) );

  // Then to intersect that volume with a sphere
  G4VSolid* bellyPlate = new G4IntersectionSolid( name + "_displaced_belly_plate_solid", sphere, polycones, NULL, G4ThreeVector( 0.0, 0.0, 0.0 ) );

  // Finally move the net solid back to the origin
  return new G4DisplacedSolid( name + "_belly_plate", bellyPlate, NULL, G4ThreeVector( -offset, 0.0, 0.0 ) );
}
```
Further down the pipeline, this object is then used for [subtraction](https://github.com/snoplus/rat/blob/ba923dbacc8fd51475d47cbaeb61af31a540c868/src/geo/AcrylicVesselFactory.cc#L68-L77):
```c++
      if( addBellyPlates )
        {
          for( size_t uPlate = 0; uPlate < bellyX.size(); uPlate++ )
            {
              G4RotationMatrix* rotation = new G4RotationMatrix();
              rotation->rotateZ( bellyRoll[uPlate] * deg );
              G4ThreeVector position( bellyX[uPlate], bellyY[uPlate], bellyZ[uPlate] );
              av = new G4SubtractionSolid( name + "_av_solid", av, bellyPlate, rotation, position );
            }
        }
```
Also, the return object's name for belly plate is `*_belly_plate`, however, in the GDML it is
`inner_av_belly_plate_displaced_belly_plate_solid0x55f5cf7bb160`, which is the intermediate intersection solid. 

This is because `G4DisplacedSolid` is not supposed to be used as a normal solid, but implemented as an [intermediate step
for boolean
operations](https://gitlab.cern.ch/geant4/geant4/-/blob/master/source/geometry/solids/Boolean/src/G4DisplacedSolid.cc#L26-27).

So likely, translation in the belly construction has been consolidated as part of the subtraction operation. Meaning
that the operational order carried out is (rotation->translation1->translation2), where as the desired order is
translation1->rotation->translation2.

This might be happening to other `G4DisplacedSolid` in SNO+ RAT (needs confirmation):
- HoldUpRope
- BellyPlate
- BellyGrove
- HoldDownRope
- AVPipe
- InternalRope

## Solution
The issue was also previously seen in ATLAS geometry, reported here:
<https://indico.cern.ch/event/727112/contributions/3104480/attachments/1706029/2749574/Investigations_Atlas.pdf>, page 8.

Bug report was patched in Geant4 10.4.p02
<https://gitlab.cern.ch/geant4/geant4/-/blob/master/source/geometry/solids/Boolean/src/G4DisplacedSolid.cc#L30>,
however, we are stuck on ancient 10.0.p04.



## Another Bug

Belly groove objects are not continuous because of an offset issue: <https://github.com/snoplus/rat/blob/ba923dbacc8fd51475d47cbaeb61af31a540c868/src/geo/solids/BellyGroove.cc#L83>

![Belly Groove](/assets/snoplus_geo_bug/belly_groove.png)
