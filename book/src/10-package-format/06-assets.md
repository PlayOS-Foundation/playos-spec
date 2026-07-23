# Assets

The `assets/` directory contains game assets — textures, audio, scripts,
data files, and any other content the application needs at runtime.

## Layout

The `assets/` directory has no prescribed internal structure. The
developer organizes assets as they see fit:

```text
assets/
├── textures/
│   ├── player.png
│   └── background.png
├── audio/
│   ├── music.ogg
│   └── sfx.wav
├── data/
│   ├── levels.json
│   └── strings/
│       ├── en.json
│       └── jp.json
└── shaders/
    └── bloom.glsl
```

## Access at runtime

The runtime sets the current working directory to the package root before
launching the application. Assets are accessed via relative paths:

```cpp
// Assets are relative to the package root.
LoadTexture("assets/textures/player.png");
LoadAudio("assets/audio/music.ogg");
```

## Recommended conventions

- Use lowercase filenames with underscores or hyphens as separators.
- Group by type: `textures/`, `audio/`, `data/`, `shaders/`.
- Keep paths short (under 256 characters total).
- Use standard formats: PNG for textures, OGG/Vorbis for audio, JSON/TOML
  for data files.

## Size limits

The package format imposes no size limit on `assets/`, but practical
guidelines apply:

- Keep total package size reasonable (recommended: under 8 GB for store
  distribution).
- Compress textures and audio appropriately.
- The marketplace may reject packages that exceed size limits.

## Related

- [Package Anatomy](03-package-anatomy.md)
- [Storage API](../06-platform-api/10-storage-api.md) — save data, which
  lives outside the package
