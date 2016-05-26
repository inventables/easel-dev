---
layout: default
---

# <a name="overview"></a> Overview

At a high level, apps make it easier for users to make things. They do this by providing `properties` (e.g., sliders, input boxes, dropdowns) that the user may manipulate. Each time the user changes one of these `properties`, your app's `executor` function will be invoked. Easel will provide the new values of all the properties, along with several other data elements representing the context of the user's project. Your app will use this data to create new design objects, or to modify existing ones.

# <a name="volumes"></a> Volumes

A core data type in the API is the `volume` object. A user's design in Easel is represented by a collection of `volume` objects. We call them `volumes` because they represent a three dimensional _volume_ of material to be removed from the work piece. When a user places a circle on the canvas in Easel and assigns it a cut depth of say 0.25 inches, what the user is really saying is that they want to remove a cylinder of material.

![Cylinder](/img/cylinder.png)

Things get more interesting when the user places overlapping shapes on the canvas. Suppose the user drops a triangle with a cut depth of 0 inches somewhere on top of but not completely covering the circle. Easel tracks each volume separately, but the _order_ in which those shapes are placed on top of each other is important. Because the triangle is on top of the circle, its cut depth _overrides_ the cut depth of the circle in the regions where the two shapes overlap. The result is that the two volumes will combine to carve a "pie" volume looking like this:

![Pie](/img/pie.png)

Your app will send `volume` objects back to Easel, telling Easel what objects to add, modify, or remove from the design.

# <a name="anatomy"></a> Anatomy of an App

Every Easel app is a single javascript file that must export the following:

- A `properties` array
- An `executor` function

## <a name="properties"></a> Properties

When your app is first launched by the user, Easel will invoke it and look for the `properties` array that must be exposed. This array defines the properties that the user may manipulate when using your app. Each element in the array must be an object with the following attributes:

```
id: String   // Doubles as an identifier and the label to show next to the property
type: String // One of 'range', 'boolean', 'file-input', 'text', 'list'
```

Each type of property carries its own specific attributes:

**Range** (slider)

```
min: number   // The minimum value for the range
max: number   // The maximum value for the range
step: number  // How much to step the value by when dragging the slider
value: number // The initial value
```

**Boolean** (checkbox)

```
value: boolean // The initial value
```

**File Picker**

(No additional attributes)

**Text** (input)

```
value: String // The initial value
```

**List** (select)

```
options: Array of Strings // The options for the dropdown list
value: String // The initial value
```


## <a name="executor-function"></a> Executor Function

Whenever the user changes the value of a property, Easel will invoke your app's `executor` function.  The signature of the `executor` function is:

`function executor(args, success, failure)`

The `args` parameter is an object structured as follows:


```
  args:
    params: Object // The user-defined parameters your app exposes, see below
    volumes: Array // The objects defined in the design, ordered from back to front (see below for more)
    selectedVolumeIds: Array // String ids of volumes currently selected in the design
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
```


The `params` object will have a shape matching the properties that your app gives the user to manipulate. For example, if you make two Number properties named `Width` and `Height` available, then you would receive a params object like this:

```
params:
  Width: Number
  Height: Number
```

The `volumes` property represents the current state of the user's entire design in the active project. In the array given to your app, volumes are ordered from back to front as the user looks at their canvas. Therefore, the last volume is in front of all other volumes, and the first volume is behind all other volumes. Front-to-back ordering is important in Easel because it affects how volumes get carved out:

1.  A (filled) volume whose cut depth is shallower than a volume behind it will "lift" material out of the volume behind it
2.  A volume whose cut depth is deeper than an object behind it will "push" material down into that shape
3.  Volumes can't "raise" or "push" on volumes in front of them

Basically, in terms of cut depth, each volume in the array overrides all overlapping volumes that came before it.

Properties that are shared across all volumes **except `lines`** are:

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

**Volumes of type `line` do not have a `center`, `width`, `height`, or `rotation`.**

In addition to the shared properties, shapes often have some properties specific to their type:

**Polygon** (including Triangles)

```
shape:
  points: Array
```

Where each point in the array has the following shape:

```
{
  x: Number // Horizontal position of point
  y: Number // Vertical position of point
}
```

**Path**

```
shape:
  points: Array
```

Where each point in the array has the following shape:

```
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
```

Points will have handles if the path curves around them (on one side or the other). The handles represent the (relative) locations of bezier control points. In the v2 apps API, Easel provides functions to map from an Easel path to an SVG path.

**Text**

```
font: String     // Name of font, must be one of the fonts listed in Easel
fontSize: Number // Must be 80
text: String     // The content of the text object
```

**Line**

```
point1:
  x: Number
  y: Number
point2:
  x: Number
  y: Number
```

# <a name="responses"></a> Responses

Easel passes 2 additional arguments to your app: a `success` callback and a `failure` callback.

If your app is able to function properly and produce output, it should invoke the `success` callback, passing an array of volumes having the same structure as above. The only wrinkle is with handling the update model--whether you want to add, replace, or remove volumes from the existing design.

**Adding volumes:** If you want to add new volumes to the design, leave the "id" field of the volumes blank in your response

**Replacing volumes:** If you want to replace a volume in the original design, pass the `id` of the existing volume with its new data

**Removing volumes:** If you want to remove a volume from the original design, pass the `id` of the existing volume with no shape or cut data

