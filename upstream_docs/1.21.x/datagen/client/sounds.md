Sound Definition Generation
===========================

The `sounds.json` file can be generated for a mod by subclassing `SoundDefinitionsProvider` and implementing `#registerSounds`. After implementation, the provider must be [added][datagen] to the `DataGenerator`.

```java
// On the MOD event bus
@SubscribeEvent
public void gatherData(GatherDataEvent event) {
    event.getGenerator().addProvider(
        // Tell generator to run only when client assets are generating
        event.includeClient(),
        output -> new MySoundDefinitionsProvider(output, MOD_ID, event.getExistingFileHelper())
    );
}
```

Adding a Sound
--------------

A sound definition can be generated by specifying the sound name and definition via `#add`. The sound name can either be provided from a [`SoundEvent`][soundevent], `ResourceLocation`, or string.

!!! warning
    The sound name supplied will always assume the namespace is the mod id supplied to the constructor of the provider. There is no validation performed on the namespace of the sound name!

### `SoundDefinition`

The `SoundDefinition` can be created using `#definition`. The definition contains the data to define a sound instance.

A definition specifies a few methods:

Method     | Description
:---:      | :---
`with`     | Adds a sound(s) which may be played when the definition is selected.
`subtitle` | Sets the translation key of the definition.
`replace`  | When `true`, removes the sounds already defined by other `sounds.json` for this definition instead of appending to it.

### `SoundDefinition$Sound`

A sound supplied to the `SoundDefinition` can be specified using `SoundDefinitionsProvider#sound`. These methods take in the reference of the sound and a `SoundType` if specified.

The `SoundType` can be one of two values:

Sound Type | Definition
:---:      | :---
`SOUND`    | Specifies a reference to the sound located at `assets/<namespace>/sounds/<path>.ogg`.
`EVENT`    | Specifies a reference to the name of another sound defined by the `sounds.json`.

Each `Sound` created from `SoundDefinitionsProvider#sound` can specify additional configurations on how to load and play the sound provided:

Method                | Description
:---:                 | :---
`volume`              | Sets the volume scale of the sound, must be greater than `0`.
`pitch`               | Sets the pitch scale of the sound, must be greater than `0`.
`weight`              | Sets the likelihood of the sound getting played when the sound is selected.
`stream`              | When `true`, reads the sound from file instead of loading the sound into memory. Recommended for long sounds: background music, music discs, etc.
`attenuationDistance` | Sets the number of blocks the sound can be heard from.
`preload`             | When `true`, immediately loads the sound into memory as soon as the resource pack is loaded.

```java
// In some SoundDefinitionsProvider#registerSounds
this.add(EXAMPLE_SOUND_EVENT, definition()
  .subtitle("sound.examplemod.example_sound") // Set translation key
  .with(
    sound(ResourceLocation.fromNamespaceAndPath(MODID, "example_sound_1")) // Set first sound
      .weight(4) // Has a 4 / 5 = 80% chance of playing
      .volume(0.5), // Scales all volumes called on this sound by half
    sound(ResourceLocation.fromNamespaceAndPath(MODID, "example_sound_2")) // Set second sound
      .stream() // Streams the sound
  )
);

this.add(EXAMPLE_SOUND_EVENT_2, definition()
  .subtitle("sound.examplemod.example_sound") // Set translation key
  .with(
    sound(EXAMPLE_SOUND_EVENT.getLocation(), SoundType.EVENT) // Adds sounds from 'EXAMPLE_SOUND_EVENT'
      .pitch(0.5) // Scales all pitches called on this sound by half
  )
);
```

[datagen]: ../index.md#data-providers
[soundevent]: ../../gameeffects/sounds.md#creating-sound-events
