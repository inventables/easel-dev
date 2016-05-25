---
layout: default
---

<h1>
  <a id="executor-function" class="anchor" href="#executor-function" aria-hidden="true"><span aria-hidden="true" class="octicon octicon-link"></span></a>Executor Function
</h1>

The executor function takes 3 arguments.


```
  args:
    params: Object // The user-defined parameters your app exposes, see below
    material:
      name: String // Name of active material
      textureUrl: String // URL of material thumbnail image
      dimensions:
        x: number // Width of material in inches
        y: number // Length of material in inches
        z: number // Height of material in inches
    bitParams:
      bit:
        id: String // Identifier for the bit
        width: Number // Width of the bit
        unit: String ("mm" or "in") // Units the width is expressed in
      detailBit:   // Not present if only using 1 bit
        id: String // Identifier for the detail bit
        width: Number // Width of the detail bit
        unit: String ("mm" or "in") // Units the width is expressed in
    volumes: Array // The objects defined in the design, ordered from back to front (see below for more)
    selectedVolumeIds: Array // String ids of volumes currently selected in the design
```


The `params` object will have a shape matching the properties that your app gives the user to manipulate. For example, if you make two Number properties named `Width` and `Height` available, then you would receive a params object like this:

```
params:
  Width: Number
  Height: Number
```

A user's design is represented by `volume` objects in Easel. We call them `volumes` because they represent a three dimensional volume of material to be removed from the work piece. In the array given to your app, volumes are ordered from back to front as the user looks at their canvas. Therefore, the last volume is in front of all other volumes, and the first volume is behind all other volumes. Front-to-back ordering is important in Easel because it affects how volumes get carved out:

1.  A (filled) volume whose cut depth is shallower than a volume behind it will "lift" material out of the volume behind it
2.  A volume whose cut depth is deeper than an object behind it will "push" material down into that shape
3.  Volumes can't "raise" or "push" on volumes in front of them

Basically, in terms of cut depth, each volume in the array overrides all overlapping volumes that came before it.

Properties that are shared across all volumes are:

    volume:
      shape:
        type: String // "rectangle", "ellipse", "polygon", "path", "polyline", "line"
        flipping:
          x: Boolean // True if the shape is "flipped" horizontally
          y: Boolean // True if the shape is "flipped" vertically
        center: // Most shapes (except Lines) have a center
          x: Number // Absolute horizontal position in inches of the shape's center, relative to the zero point
          y: Number // Absolute vertical position in inches of the shape's center, relative to the zero point
        width: Number    // Width in inches of the shape
        height: Number   // Height in inches of the shape
        rotation: Number // Amount of counter-clockwise rotation in radians
      cut:
        depth: Number // Depth of cut for this shape in inches
        type: String ('outline' or 'fill') // Type of cut
        outlineStyle: String ('on-path', 'outside', 'inside') // Outline style, if outline cut
        tabPreference: Boolean // Whether the user wants to use tabs, if cut-through outline cut
        tabHeight: Number // Height (from bottom of material) of tabs in inches
        tabLength: Number // Length of tabs in inches
        tabs: Array // Objects with a shape of {center: {x: Number, y: Number}} representing the center location of the tabs

In addition, shapes often have some properties specific to their type:

Polygon (including Triangles)

    shape:
      points: Array

Where each point in the array has the following shape:

    {
      x: Number // Horizontal position of point
      y: Number // Vertical position of point
    }

Path

    shape:
      points: Array

Where each point in the array has the following shape:

    {
      x: Number // Horizontal position of point
      y: Number // Vertical position of point
      lh: // Optional "left handle"
        x: Number // Horizontal position of left handle RELATIVE to the point
        y: Number // Vertical position of left handle RELATIVE to the point
      rh: // Optional "right handle"
        x: Number // Horizontal position of right handle RELATIVE TO THE POINT
        y: Number // Vertical position of right handle RELATIVE TO THE POINT
    }

Points will have handles if the path curves around them (on one side or the other). The handles represent the (relative) locations of bezier control points. In the v2 apps API, Easel provides functions to map from an Easel path to an SVG path.

# [<span aria-hidden="true" class="octicon octicon-link"></span>](#responses)Responses

Easel passes 2 additional arguments to your app: a `success` callback and a `failure` callback.

If your app is able to function properly and produce output, it should invoke the `success` callback, passing an array of volumes having the same structure as above. The only wrinkle is with how to handle the update model--whether you want to add, replace, or remove volumes from the existing design.

- **Add volumes:** If you want to add new volumes to the design, leave the "id" field of the volumes blank in your response
- **Replace volumes:** If you want to replace a volume, pass the "id" of the existing volume with its new data
- **Remove volumes:** If you want to remove a volume, pass the "id" of the existing volume with no shape or cut data

