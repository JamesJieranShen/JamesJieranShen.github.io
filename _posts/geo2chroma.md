---
layout: post
title:  "RAT to Chroma geometry conversion"
date:   2022-07-01 22:00:00 -0400
categories: chroma
---

## Convert `.geo`  to `.gdml`

- SNO+ RAT does not assign surface properties to every single material. Materials like acrylic and rock has no entries in the
SURFACE.ratdb files. This leads to a segfault, as there is also no guard against accessing the
null surface property pointers. I have circled around that so far by adding a `default_surface` during surface loading,
but I am not sure if there is a better way to handle this. `RATPAC-TWO` handles surfaces differently so this may simply cease to become an issue. 


- GEANT4.10.4, which is the latest version supported by SNO+
RAT, has an arbitrary limit to the file line length in the xml files. This is length is set to 99, although memory has
been allocated for 10000 characters... This is fixed in GEANT4.11, but without forking geant4.10 or completely
re-writing the geant4 gdml parsing package outside of geant, there is no way of circumventing this issue. This is
probelematic as the reflectivity spectrum is written in a single line in the gdml files. I have forked geant4 to raise
the character limit. Given that `ratpac-two` supports geant4.11, I don't
anticipate this being a serious issue.

## Convert `gdml` to mesh
There are several issues in this... Current solution uses PyMesh, which is still
a highly popular library, but has not received a commit in more than 2 years.
Might consider hosting that as part of chroma source code?

### Adding more primary shapes
PyMesh only supports a spheres, tubes boxes, and some basic shapes, not
everything in the GDML spec. Most importantly SNO+ uses polyhedra, polycones,
torus, etc.

Solve this using Signed Distance Function. [This
library](https://github.com/fogleman/sdf) seems to deal with most of the nitty
gritties. It is super lightweight but we need some mods to the way it handles
outputs, so we want to host our fork by ourselves, perhaps directly as part of
`chroma`.

### Boolean operations
If we are using SDFs, should we just use it for boolean ops as well? It may have
worse precision than direct mesh clipping? It couldn't be that bad if we
increase triangle count. Also, we wouldn't need to store extra meshes we don't
need during runtime -> memory benefit.

### Dealing with material
Chroma's `GDMLLoader` only deals with mesh gen right now and hands off the
material assignment part to another function, which is not ideal. This can be
addressed if we improve parsing. 
