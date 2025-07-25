Model Generation
================

[Models] can be generated for models or block states by default. Each provides a method of generating the necessary JSONs (`ModelBuilder#toJson` for models and `IGeneratedBlockState#toJson` for block states). After implementation, the [associated providers][provider] must be [added][datagen] to the `DataGenerator`.

```java
// On the MOD event bus
@SubscribeEvent
public void gatherData(GatherDataEvent event) {
    DataGenerator gen = event.getGenerator();
    ExistingFileHelper efh = event.getExistingFileHelper();

    gen.addProvider(
        // Tell generator to run only when client assets are generating
        event.includeClient(),
        output -> new MyItemModelProvider(output, MOD_ID, efh)
    );
    gen.addProvider(
        event.includeClient(),
        output -> new MyBlockStateProvider(output, MOD_ID, efh)
    );
}
```

Model Files
-----------

A `ModelFile` acts as the base for all models referenced or generated by a provider. Each model file stores the location relative to the `models` subdirectory and can assert whether the file exists.

### Existing Model Files

`ExistingModelFile` is a subclass of `ModelFile` which checks via [`ExistingFileHelper#exists`][efh] whether the model already exists within the `models` subdirectory. All non-generated models are usually referenced through `ExistingModelFile`s.

### Unchecked Model Files

`UncheckedModelFile` is a subclass of `ModelFile` which assumes the specified model exists in some location.

!!! note
    There should be no cases where an `UncheckedModelFile` is used to reference a model. If there is, then the associated resources are not properly being tracked by `ExistingFileHelper`.

Model Builders
--------------

A `ModelBuilder` represents a to-be-generated `ModelFile`. It contains all the data about a model: its parent, faces, textures, transformations, lighting, and [loader].

!!! tip
    While a complex model can be generated, it is recommended that those models be constructed using a modeling software beforehand. Then, the data provider can generate the children models with specific textures applied through the defined references in the parent complex model.

The parent (via `ModelBuilder#parent`) of the builder can be any `ModelFile`: generated or existing. Generated files are added to `ModelProvider`s as soon as the builder is created. The builder itself can be passed in as a parent, or the `ResourceLocation` can supplied alternatively.

!!! warning
    If the parent is not generated before the child model when passing in a `ResourceLocation`, then an exception will be thrown.

Each element (via `ModelBuilder#element`) within a model is defined as cube using two three-dimensional points (`ElementBuilder#from` and `#to` respectively) where each axis is limited to the values `[-16,32]` (between -16 and 32 inclusive). Each face (`ElementBuilder#face`) of the cube can specify when the face is culled (`FaceBuilder#cullface`), [tint index][color] (`FaceBuilder#tintindex`), texture reference from the `textures` keys (`FaceBuilder#texture`), UV coordinate on the texture (`FaceBuilder#uvs`), and rotation in 90 degree intervals (`FaceBuilder#rotation`).

!!! note
    It recommended for block models which have elements that exceed a bound of `[0,16]` on any axis to separate into multiple blocks, such as for a multiblock structure, to avoid lighting and culling issues.

Each cube can additionally be rotated (`ElementBuilder#rotation`) around a specified point (`RotationBuilder#origin`) for a given axis (`RotationBuilder#axis`) in 22.5 degree intervals (`RotationBuilder#angle`). The cube can scale all faces in relation to the entire model as well (`RotationBuilder#rescale`). The cube can also determine whether its shadows should be rendered (`ElementBuilder#shade`).

Each model defines a list of texture keys (`ModelBuilder#texture`) which points to either a location or a reference. Each key can then be referenced in any element by prefixing using a `#` (a texture key of `example` can be referenced in an element using `#example`). A location specifies where a texture is in `assets/<namespace>/textures/<path>.png`. A reference is used by any models parenting the current model as keys to define textures for later.

The model can additionally be transformed (`ModelBuilder#transforms`) for any defined perspective (in the left hand in first person, in the gui, on the ground, etc.). For any perspective (`TransformsBuilder#transform`), the rotation (`TransformVecBuilder#rotation`), translation (`TransformVecBuilder#translation`), and scale (`TransformVecBuilder#scale`) can be set.

Finally, the model can set whether to use ambient occlusion in a level (`ModelBuilder#ao`) and from what location to light and shade the model from `ModelBuilder#guiLight`.

### `BlockModelBuilder`

A `BlockModelBuilder` represents a block model to-be-generated. In addition to the `ModelBuilder`, a transform to the entire model (`BlockModelBuilder#rootTransform`) can be generated. The root can be translated (`RootTransformBuilder#transform`), rotated (`RootTransformBuilder#rotation`, `RootTransformBuilder#postRotation`), and scaled (`RootTransformBuilder#scale`) either individually or all in one transformation (`RootTransformBuilder#transform`) around some origin (`RootTransformBuilder#origin`).

### `ItemModelBuilder`

An `ItemModelBuilder` represents an item model to-be-generated. In addition to the `ModelBuilder`, [overrides] (`OverrideBuilder#override`) can be generated. Each override applied to a model can apply conditions which represent for a given property that must be above the specified value (`OverrideBuilder#predicate`). If the conditions are met, then the specified model (`OverrideBuilder#model`) will be rendered instead of this model.

Model Providers
---------------

The `ModelProvider` subclasses are responsible for generating the constructed `ModelBuilder`s. The provider takes in the generator, mod id, subdirectory in the `models` folder to generate within, a `ModelBuilder` factory, and the existing file helper. Each provider subclass must implement `#registerModels`.

The provider contains basic methods which either create the `ModelBuilder` or provides convenience for getting texture or model references:

Method               | Description
:---:                | :---
`getBuilder`         | Creates a new `ModelBuilder` within the provider's subdirectory for the given mod id.
`withExistingParent` | Creates a new `ModelBuilder` for the given parent. Should be used when the parent is not generated by the builder.
`mcLoc`              | Creates a `ResourceLocation` for the path in the `minecraft` namespace.
`modLoc`             | Creates a `ResourceLocation` for the path in the given mod id's namespace.

Additionally, there are several helpers for easily generating common models using vanilla templates. Most are for block models with only a few being universal.

!!! note
    Although the models are within a specific subdirectory, that does **not** mean that the model cannot be referenced by a model in another subdirectory. Usually, it is indicative of that model being used for that type of object.

### `BlockModelProvider`

The `BlockModelProvider` is used for generating block models via `BlockModelBuilder` in the `block` folder. Block models should typically parent `minecraft:block/block` or one of its children models for use with item models.

!!! note
    Block models and its item model counterpart are typically not generated through a direct subclass of `BlockModelProvider` and `ItemModelProvider` but through [`BlockStateProvider`][blockstateprovider].

### `ItemModelProvider`

The `ItemModelProvider` is used for generating block models via `ItemModelBuilder` in the `item` folder. Most item models parent `item/generated` and use `layer0` to specify their texture, which can be done using `#singleTexture`.

!!! note
    `item/generated` can support five texture layers stacked on top of each other: `layer0`, `layer1`, `layer2`, `layer3`, and `layer4`.

```java
// In some ItemModelProvider#registerModels

// Will generate 'assets/<modid>/models/item/example_item.json'
// Parent will be 'minecraft:item/generated'
// For the texture key 'layer0'
//  It will be at 'assets/<modid>/textures/item/example_item.png'
this.basicItem(EXAMPLE_ITEM.get());
```

!!! note
    Item models for blocks should typically parent an existing block model instead of generating a separate model for an item.

Block State Provider
--------------------

A `BlockStateProvider` is responsible for generating [block state JSONs][blockstate] in `blockstates`, block models in `models/block`, and item models in `models/item` for said blocks. The provider takes in the data generator, mod id, and existing file helper. Each `BlockStateProvider` subclass must implement `#registerStatesAndModels`.

The provider contains basic methods for generating block state JSONs and block models. Item models must be generated separately as a block state JSON may define multiple models to use in different contexts. There are a number of common methods, however, that that the modder should be aware of when dealing with more complex tasks:

Method                | Description
:---:                 | :---
`models`              | Gets the [`BlockModelProvider`][blockmodels] used to generate the item block models.
`itemModels`          | Gets the [`ItemModelProvider`][itemmodels] used to generate the item block models.
`modLoc`              | Creates a `ResourceLocation` for the path in the given mod id's namespace.
`mcLoc`               | Creates a `ResourceLocation` for the path in the `minecraft` namespace.
`blockTexture`        | References a texture within `textures/block` which has the same name as the block.
`simpleBlockItem`     | Creates an item model for a block given the associated model file.
`simpleBlockWithItem` | Creates a single block state for a block model and an item model using the block model as its parent.

A block state JSON is made up of variants or conditions. Each variant or condition references a `ConfiguredModelList`: a list of `ConfiguredModel`s. Each configured model contains the model file (via `ConfiguredModel$Builder#modelFile`), the X and Y rotation in 90 degree intervals (via `#rotationX` and `rotationY` respectively), whether the texture can rotate when the model is rotated by the block state JSON (via `#uvLock`), and the weight of the model appearing compared to other models in the list (via `#weight`).

The builder (`ConfiguredModel#builder`) can also create an array of `ConfiguredModel`s by creating the next model using `#nextModel` and repeating the settings until `#build` is called.

### `VariantBlockStateBuilder`

Variants can be generated using `BlockStateProvider#getVariantBuilder`. Each variant specifies a list of [properties] (`PartialBlockstate`) which when matches a `BlockState` in a level, will display a model chosen from the corresponding model list. An exception is thrown if there is a `BlockState` which is not covered by any variant defined. Only one variant can be true for any `BlockState`.

A `PartialBlockstate` is typically defined using one of three methods:

Method               | Description
:---:                | :---
`partialState`       | Creates a `PartialBlockstate` to be defined.
`forAllStates`       | Defines a function where a given `BlockState` can be represented by an array of `ConfiguredModel`s.
`forAllStatesExcept` | Defines a function similar to `#forAllStates`; however, it also specifies which properties do not affect the models rendered.

For a `PartialBlockstate`, the properties defined can be specified (`#with`). The configured models can be set (`#setModels`), appended to the existing models (`#addModels`), or built (`#modelForState` and then `ConfiguredModel$Builder#addModel` once finished instead of `#ConfiguredModel$Builder#build`).

```java
// In some BlockStateProvider#registerStatesAndModels

// EXAMPLE_BLOCK_1: Has Property BlockStateProperties#AXIS
this.getVariantBuilder(EXAMPLE_BLOCK_1) // Get variant builder
  .partialState() // Construct partial state
  .with(AXIS, Axis.Y) // When BlockState AXIS = Y
    .modelForState() // Set models when AXIS = Y
    .modelFile(yModelFile1) // Can show 'yModelFile1'
    .nextModel() // Adds another model when AXIS = Y
    .modelFile(yModelFile2) // Can show 'yModelFile2'
    .weight(2) // Will show 'yModelFile2' 2/3 of the time
    .addModel() // Finalizes models when AXIS = Y
  .with(AXIS, Axis.Z) // When BlockState AXIS = Z
    .modelForState() // Set models when AXIS = Z
    .modelFile(hModelFile) // Can show 'hModelFile'
    .addModel() // Finalizes models when AXIS = Z
  .with(AXIS, Axis.X)  // When BlockState AXIS = X
    .modelForState() // Set models when AXIS = X
    .modelFile(hModelFile) // Can show 'hModelFile'
    .rotationY(90) // Rotates 'hModelFile' 90 degrees on the Y axis
    .addModel(); // Finalizes models when AXIS = X

// EXAMPLE_BLOCK_2: Has Property BlockStateProperties#HORIZONTAL_FACING
this.getVariantBuilder(EXAMPLE_BLOCK_2) // Get variant builder
  .forAllStates(state -> // For all possible states
    ConfiguredModel.builder() // Creates configured model builder
      .modelFile(modelFile) // Can show 'modelFile'
      .rotationY((int) state.getValue(HORIZONTAL_FACING).toYRot()) // Rotates 'modelFile' on the Y axis depending on the property
      .build() // Creates the array of configured models
  );

// EXAMPLE_BLOCK_3: Has Properties BlockStateProperties#HORIZONTAL_FACING, BlockStateProperties#WATERLOGGED
this.getVariantBuilder(EXAMPLE_BLOCK_3) // Get variant builder
  .forAllStatesExcept(state -> // For all HORIZONTAL_FACING states
    ConfiguredModel.builder() // Creates configured model builder
      .modelFile(modelFile) // Can show 'modelFile'
      .rotationY((int) state.getValue(HORIZONTAL_FACING).toYRot()) // Rotates 'modelFile' on the Y axis depending on the property
      .build(), // Creates the array of configured models
  WATERLOGGED); // Ignores WATERLOGGED property
```

### `MultiPartBlockStateBuilder`

Multiparts can be generated using `BlockStateProvider#getMultipartBuilder`. Each part (`MultiPartBlockStateBuilder#part`) specifies a group of conditions of properties which when matches a `BlockState` in a level, will display a model from the model list. All condition groups that match the `BlockState` will display their chosen model overlaid on each other.

For any part (obtained via `ConfiguredModel$Builder#addModel`), a condition can be added (via `#condition`) when a property is one of the specified values. Conditions must all succeed or, when `#useOr` is set, at least one must succeed. Conditions can be grouped (via `#nestedGroup`) as long as the current grouping only contains other groups and no single conditions. Groups of conditions can be left using `#endNestedGroup` and a given part can be finished via `#end`.

```java
// In some BlockStateProvider#registerStatesAndModels

// Redstone Wire
this.getMultipartBuilder(REDSTONE) // Get multipart builder
  .part() // Create part
    .modelFile(redstoneDot) // Can show 'redstoneDot'
    .addModel() // 'redstoneDot' is displayed when...
    .useOr() // At least one of these conditions are true
    .nestedGroup() // true when all grouped conditions are true
      .condition(WEST_REDSTONE, NONE) // true when WEST_REDSTONE is NONE
      .condition(EAST_REDSTONE, NONE) // true when EAST_REDSTONE is NONE
      .condition(SOUTH_REDSTONE, NONE) // true when SOUTH_REDSTONE is NONE
      .condition(NORTH_REDSTONE, NONE) // true when NORTH_REDSTONE is NONE
    .endNestedGroup() // End group
    .nestedGroup() // true when all grouped conditions are true
      .condition(EAST_REDSTONE, SIDE, UP) // true when EAST_REDSTONE is SIDE or UP
      .condition(NORTH_REDSTONE, SIDE, UP) // true when NORTH_REDSTONE is SIDE or UP
    .endNestedGroup() // End group
    .nestedGroup() // true when all grouped conditions are true
      .condition(EAST_REDSTONE, SIDE, UP) // true when EAST_REDSTONE is SIDE or UP
      .condition(SOUTH_REDSTONE, SIDE, UP) // true when SOUTH_REDSTONE is SIDE or UP
    .endNestedGroup() // End group
    .nestedGroup() // true when all grouped conditions are true
      .condition(WEST_REDSTONE, SIDE, UP) // true when WEST_REDSTONE is SIDE or UP
      .condition(SOUTH_REDSTONE, SIDE, UP) // true when SOUTH_REDSTONE is SIDE or UP
    .endNestedGroup() // End group
    .nestedGroup() // true when all grouped conditions are true
      .condition(WEST_REDSTONE, SIDE, UP) // true when WEST_REDSTONE is SIDE or UP
      .condition(NORTH_REDSTONE, SIDE, UP) // true when NORTH_REDSTONE is SIDE or UP
    .endNestedGroup() // End group
    .end() // Finish part
  .part() // Create part
    .modelFile(redstoneSide0) // Can show 'redstoneSide0'
    .addModel() // 'redstoneSide0' is displayed when...
    .condition(NORTH_REDSTONE, SIDE, UP) // NORTH_REDSTONE is SIDE or UP
    .end() // Finish part
  .part() // Create part
    .modelFile(redstoneSideAlt0) // Can show 'redstoneSideAlt0'
    .addModel() // 'redstoneSideAlt0' is displayed when...
    .condition(SOUTH_REDSTONE, SIDE, UP) // SOUTH_REDSTONE is SIDE or UP
    .end() // Finish part
  .part() // Create part
    .modelFile(redstoneSideAlt1) // Can show 'redstoneSideAlt1'
    .rotationY(270) // Rotates 'redstoneSideAlt1' 270 degrees on the Y axis
    .addModel() // 'redstoneSideAlt1' is displayed when...
    .condition(EAST_REDSTONE, SIDE, UP) // EAST_REDSTONE is SIDE or UP
    .end() // Finish part
  .part() // Create part
    .modelFile(redstoneSide1) // Can show 'redstoneSide1'
    .rotationY(270) // Rotates 'redstoneSide1' 270 degrees on the Y axis
    .addModel() // 'redstoneSide1' is displayed when...
    .condition(WEST_REDSTONE, SIDE, UP) // WEST_REDSTONE is SIDE or UP
    .end() // Finish part
  .part() // Create part
    .modelFile(redstoneUp) // Can show 'redstoneUp'
    .addModel() // 'redstoneUp' is displayed when...
    .condition(NORTH_REDSTONE, UP) // NORTH_REDSTONE is UP
    .end() // Finish part
  .part() // Create part
    .modelFile(redstoneUp) // Can show 'redstoneUp'
    .rotationY(90) // Rotates 'redstoneUp' 90 degrees on the Y axis
    .addModel() // 'redstoneUp' is displayed when...
    .condition(EAST_REDSTONE, UP) // EAST_REDSTONE is UP
    .end() // Finish part
  .part() // Create part
    .modelFile(redstoneUp) // Can show 'redstoneUp'
    .rotationY(180) // Rotates 'redstoneUp' 180 degrees on the Y axis
    .addModel() // 'redstoneUp' is displayed when...
    .condition(SOUTH_REDSTONE, UP) // SOUTH_REDSTONE is UP
    .end() // Finish part
  .part() // Create part
    .modelFile(redstoneUp) // Can show 'redstoneUp'
    .rotationY(270) // Rotates 'redstoneUp' 270 degrees on the Y axis
    .addModel() // 'redstoneUp' is displayed when...
    .condition(WEST_REDSTONE, UP) // WEST_REDSTONE is UP
    .end(); // Finish part
```

Model Loader Builders
---------------------

Custom model loaders can also be generated for a given `ModelBuilder`. Custom model loaders subclass `CustomLoaderBuilder` and can be applied to a `ModelBuilder` via `#customLoader`. The factory method passed in creates a new loader builder to which configurations can be made. After all the changes have been finished, the custom loader can return back to the `ModelBuilder` via `CustomLoaderBuilder#end`.

Model Builder                       | Factory Method | Description
:---:                               | :---:          | :---
`DynamicFluidContainerModelBuilder` | `#begin`       | Generates a bucket model for the specified fluid.
`CompositeModelBuilder`             | `#begin`       | Generates a model composed of models.
`ItemLayersModelBuilder`            | `#begin`       | Generates a Forge implementation of an `item/generated` model.
`SeparateTransformsModelBuilder`    | `#begin`       | Generates a model which changes based on the specified [transform].
`ObjModelBuilder`                   | `#begin`       | Generates an [OBJ model][obj].

```java
// For some BlockModelBuilder builder
builder.customLoader(ObjModelBuilder::begin) // Custom loader 'forge:obj'
  .modelLocation(modLoc("models/block/model.obj")) // Set the OBJ model location
  .flipV(true) // Flips the V coordinate in the supplied .mtl texture
  .end() // Finish custom loader configuration
.texture("particle", mcLoc("block/dirt")) // Set particle texture to dirt
.texture("texture0", mcLoc("block/dirt")); // Set 'texture0' texture to dirt
```

Custom Model Loader Builders
----------------------------

Custom loader builders can be created by extending `CustomLoaderBuilder`. The constructor can still have a `protected` visibility with the `ResourceLocation` hardcoded to the loader id registered via `ModelEvent$RegisterGeometryLoaders#register`. The builder can then be initialized via a static factory method or the constructor if made `public`.

```java
public class ExampleLoaderBuilder<T extends ModelBuilder<T>> extends CustomLoaderBuilder<T> {
  public static <T extends ModelBuilder<T>> ExampleLoaderBuilder<T> begin(T parent, ExistingFileHelper existingFileHelper) {
    return new ExampleLoaderBuilder<>(parent, existingFileHelper);
  }

  protected ExampleLoaderBuilder(T parent, ExistingFileHelper existingFileHelper) {
    super(ResourceLocation.fromNamespaceAndPath(MOD_ID, "example_loader"), parent, existingFileHelper);
  }
}
```

Afterwards, any configurations specified by the loader should be added as chainable methods.

```java
// In ExampleLoaderBuilder
public ExampleLoaderBuilder<T> exampleInt(int example) {
  // Set int
  return this;
}

public ExampleLoaderBuilder<T> exampleString(String example) {
  // Set string
  return this;
}
```

If any additional configuration is specified, `#toJson` should be overridden to write the additional properties.

```java
// In ExampleLoaderBuilder
@Override
public JsonObject toJson(JsonObject json) {
  json = super.toJson(json); // Handle base loader properties
  // Encode custom loader properties
  return json;
}
```

Custom Model Providers
----------------------

Custom model providers require a `ModelBuilder` subclass, which defines the base of the model to generate, and a `ModelProvider` subclass, which generates the models.

The `ModelBuilder` subclass contains any special properties to which can be applied specifically to those types of models (item models can have overrides). If any additional properties are added, `#toJson` needs to be overridden to write the additional information.

```java
public class ExampleModelBuilder extends ModelBuilder<ExampleModelBuilder> {
  // ...
}
```

The `ModelProvider` subclass requires no special logic. The constructor should hardcode the subdirectory within the `models` folder and the `ModelBuilder` to represent the to-be-generated models.

```java
public class ExampleModelProvider extends ModelProvider<ExampleModelBuilder> {

  public ExampleModelProvider(PackOutput output, String modid, ExistingFileHelper existingFileHelper) {
    // Models will be generated to 'assets/<modid>/models/example' if no 'modid' is specified in '#getBuilder'
    super(output, modid, "example", ExampleModelBuilder::new, existingFileHelper);
  }
}
```

Custom Model Consumers
----------------------

Custom model consumers like [`BlockStateProvider`][blockstateprovider] can be created by manually generating the models themselves. The `ModelProvider` subclass used to generate the models should be specified and made available.

```java
public class ExampleModelConsumerProvider implements IDataProvider {

  public ExampleModelConsumerProvider(PackOutput output, String modid, ExistingFileHelper existingFileHelper) {
    this.example = new ExampleModelProvider(output, modid, existingFileHelper);
  }
}
```

Once the data provider is running, the models within the `ModelProvider` subclass can be generated using `ModelProvider#generateAll`.

```java
// In ExampleModelConsumerProvider
@Override
public CompletableFuture<?> run(CachedOutput cache) {
  // Populate the model provider
  CompletableFuture<?> exampleFutures = this.example.generateAll(cache); // Generate the models

  // Run logic and create CompletableFuture(s) for writing files
  // ...

  // Assume we have a new CompletableFuture providerFuture
  return CompletableFuture.allOf(exampleFutures, providerFuture);
}
```

[provider]: #model-providers
[models]: ../../resources/client/models/index.md
[datagen]: ../index.md#data-providers
[efh]: ../index.md#existing-files
[loader]: #custom-model-loader-builders
[color]: ../../resources/client/models/tinting.md#blockcoloritemcolor
[overrides]: ../../resources/client/models/itemproperties.md
[blockstateprovider]: #block-state-provider
[blockstate]: https://minecraft.wiki/w/Tutorials/Models#Block_states
[blockmodels]: #blockmodelprovider
[itemmodels]: #itemmodelprovider
[properties]: ../../blocks/states.md#implementing-block-states
[transform]: ../../rendering/modelloaders/transform.md
[obj]: ../../rendering/modelloaders/index.md#wavefront-obj-models
