# Layer Base Class

The `Layer` class is the base class of all deck.gl layers, and it provides
a number of **base properties** availabe in all layers.

## Static Members

#### `layerName` (static, string, required)

This static property should contain the name of the layer,
typically the name of layer's class (it cannot reliably be autodeduced in
minified code). It is used as the default Layer id as well as for debugging
and profiling.

#### `defaultProps` (static, object, optional)

All deck.gl layers define a `defaultProps` static member listing their props and
default values. Using `defaultProps` improves both the code readability and
performance during layer instance construction.

Remarks:
* For developer that would like to write new layers, it's highly recommended
to follow core deck.gl layers and define `defaultProps`. In the future version
of deck.gl, a check will be in place to enforce this

## Properties

### Basic Properties

##### `id` (String, optional*)

Default: layers `layerName` static property.

The `id` must be unique among all your layers at a given time.
The default value of `id` is the `Layer`'s "name". If more than one instance of
a specific type of layer exist at the same time, they must possess different
`id` strings for deck.gl to properly distinguish them.

Remarks:
* For React users: `id` is similar to the `key` property used in React to match
  components between rendering calls, with the difference that deck.gl requires
  the `id` whereas in React the `key` is optional but recommended.
* A layer's name is defined by the `Layer.layerName` static member, or inferred
  from the constructor name (which is not robust in minified code).
* Note that for sublayers (generated by composite layers), a composite layer id
  is generated by appending the sub layer's id to the parent layer's
  `id`, so there are no `id` collisions.

##### `data` (Array or Iterable, optional)

- Default: `[]`

The data prop should contain an iterable JavaScript container,
please see JavaScript `[Symbol.iterator]`.

##### `visible` (Boolean, optional)

- Default: `true`

Whether the layer is visible.

Under most circumstances, using `visible` prop to control the visibility
of layers are recommended over doing conditional rendering. Compare:
```js
const layers = [
  new MyLayer({data: ..., visible: showMyLayer})
];
```
with
```js
const layers = [];
if (showMyLayer) {
  layers.push(new MyLayer({data: ...}));
}
```
In the second example (conditional rendering) the layer state will
be destroyed and regenerated every time the showMyLayer flag changes.

##### `opacity` (Number, required)

The opacity of the layer. deck.gl automatically applies gamma to the opacity
in an attempt to make opacity changes appear linear (i.e. the opacity is
visually proportional to the value of the prop)

Remarks
* While it is a recommended convention that all deck.gl layers should
  support the `opacity` prop, it is up to each layer's fragment shader
  to implement support for opacity.

### Interaction Properties

Layers can be interacted with using these properties.

##### `pickable` (Boolean, optional)

- Default: `false`

Whether the layer responds to mouse pointer picking events.

##### `onHover` (Function, optional)

This callback will be called when the mouse enters/leaves an object of this
deck.gl layer with a single parameter
[`info`](/docs/interactivity.md#the-picking-info-object).

If this callback returns a truthy value, the `hover` event is marked as handled
and will not bubble up to the
[`onLayerHover`](/docs/using-with-react.md#-onlayerhover-function-optional-)
callback of the `DeckGL` canvas.

Requires `pickable` to be true.**

##### `onClick` (Function, optional)

This callback will be called when the mouse clicks over an object of this
deck.gl layer with a single parameter
[`info`](/docs/interactivity.md#the-picking-info-object).

If this callback returns a truthy value, the `hover` event is marked as handled
and will not bubble up to the
[`onLayerHover`](/docs/using-with-react.md#-onlayerhover-function-optional-)
callback of the `DeckGL` canvas.

Requires `pickable` to be true.**

### Coordinate System Properties

Normally only used when the application wants to work with coordinates
that are not Web Mercator projected longitudes/latitudes.

##### `projectionMode` (Number, optional)

Specifies how layer positions and offsets should be geographically interpreted.

The default is to interpret positions as latitude and longitude, however it
is also possible to interpret positions as meter offsets added to projection
center specified by the `projectionCenter` prop.

See the article on Coordinate Systems for details.

##### `projectionCenter` ([Number, Number], optional)

Required when the `projectionMode` is set to `COORDINATE_SYSTEM.METER_OFFSETS`.

Specifies a longitude and a latitude from which meter offsets are calculated.

See the article on Coordinate Systems for details

TODO: This section needs to be rewritten
##### `modelMatrix` (Number[16], optional)

An optional 4x4 matrix that is multiplied into the affine projection
matrices used by shader `project` GLSL function and the
Viewport's `project` and `unproject` JavaScript function.

Allows local coordinate system transformations to be applied to a layer,
which is useful when composing data from multiple sources that use
different coordinate systems.

Note that the matrix projection is applied after the non-linear mercator
projection calculations are resolved, so be careful when using model matrices
with lng/lat encoded coordinates. They normally work best with non-mercator
viewports or meter offset based mercator layers


### Data Properties

There are a number of additional properties that provide extra control over data
iteration, comparison and update.

##### `dataComparator` (Function, optional)

This prop causes the `data` prop to be compared using a custom comparison
function. The comparison function is called with the old data and the new
data objects, and is expected to return true if they compare equally.

Used to override the default shallow comparison of the `data` object.

As an illustration, the app could set this to e.g. 'lodash.isequal',
enabling deep comparison of the data structure. This particular examples
would obviously have considerable performance impact and should only
be used as a temporary solution for small data sets until the application
can be refactored to avoid the need.

##### `numInstances` (Number, optional)

deck.gl automatically derives the number of drawing instances from the `data` prop by
counting the number of objects in `data`. However, the developer might want to
maunally override it using this prop.

##### `updateTriggers` (Object, optional)

This prop expects an object of which the keys matching the accessor names of a layer.
If any values in this object are changed between props update, the attribute corresponding
to the accessors named by the key will be invalidated.

Using this method, the attribute update can happen not only when the content of `data` prop
changed, but also when the developer would like to manually force an attribute update.

Note: shallow comparision of the `data` prop has higher priority than the `updateTriggers`. So
if the app to mint a new object on every render, all attributes will be automatically updated.
updateTriggers cannot block attribute updates, just trigger them. To block the attribute updates,
developers need to override the updateState.


## Methods

> Layer methods are designed to support the creation of new layers through
layer sub-classing and are not intended to be called by applications.

### General Methods

##### setState()

Used to update the layers state object, which allows a layer to store
information that will be available to the next matching layer.

Calling this method will also mark the layer as needing a rerender.

### Layer Lifecycle Methods

Not to be called by the user.
See [layer lifecycle](/docs/layer-lifecycle.md).

### Layer Projection Methods

While most projection is handled "automatically" in the layers vertex
shader, it is occasionally useful to be able to work in the projected
coordinates in JavaScript while calculating uniforms etc.

##### `project(lngLatZ[, options])`

- `lngLatZ`: an array of `[longitude, latitude, altitude]`.
- `options.topLeft`: whether to use screen coordinates (start from top
left) or clipspace-like coordinates (start from bottom left).
Default to `true`.

Projects a map coordinate using the current viewport settings.

##### `unproject(xyz[, options])`

- `xyz`: an array of `[x, y, depth]`.
- `options.topLeft`: whether to use screen coordinates (start from top
left) or clipspace-like coordinates (start from bottom left).
Default to `true`.

Projects a pixel coordinate using the current viewport settings.

##### `projectFlat(lngLatZ[, options])`

- `lngLatZ`: an array of `[longitude, latitude, altitude]`.
- `options.topLeft`: whether to use screen coordinates (start from top
left) or clipspace-like coordinates (start from bottom left).
Default to `true`.

Projects a map coordinate using the current viewport settings, ignoring any
perspective tilt. Can be useful to calculate screen space distances.

##### `unprojectFlat(xyz[, options])`

- `xyz`: an array of `[x, y, depth]`.
- `options.topLeft`: whether to use screen coordinates (start from top
left) or clipspace-like coordinates (start from bottom left).
Default to `true`.

Unrojects a pixel coordinate using the current viewport settings, ignoring any
perspective tilt (meaning that the pixel was projected).

##### `screenToDevicePixels(pixels)`

- `pixels`: a number in screen pixels to scale to device pixels.

Simply multiplies `pixels` parameter with `window.devicePixelRatio` if
available.

Useful to adjust e.g. line widths to get more consistent visuals between
low and high resolution displays.

### Layer Picking Methods

Not to be called by the user.
See [implement picking](/docs/advanced/picking.md#layer-picking-methods).
