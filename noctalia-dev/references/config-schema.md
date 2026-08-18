# Config schema changes

Noctalia's user config is TOML (`example.toml` is the documented, ready-to-use default), backed by a typed
schema engine under `src/config/schema/` and consumed by `src/config/config_service.cpp` and friends
(`config_overrides.cpp`, `config_merge.cpp`, `state_store.cpp`).

## Where things live

| File | Role |
|---|---|
| `src/config/schema/field.h` | `Field<T>` machinery: parsing a TOML node into a typed value, `Range<T>` for min/max/step clamping |
| `src/config/schema/config_sections.h/.cpp` | Top-level config sections (bar, widgets, notifications, ...) |
| `src/config/schema/config_schema.h/.cpp` | Assembles sections into the full schema |
| `src/config/schema/widget_schema.h` | Widget-specific field schema (shared with `WidgetDefinition` - see `references/adding-a-widget.md`) |
| `src/config/schema/diagnostics.h` | Structured parse warnings/errors surfaced to the user |
| `src/config/config_migrations.h/.cpp` | Versioned migrations applied to old config files on load |
| `src/config/config_types.h` | Plain config value types shared across the schema |
| `src/config/config_validate.cpp` | `noctalia config validate` CLI validation path |

## Adding a new config key

1. Add the typed field to the relevant section struct/schema (`config_sections.*` or the widget's own
   `WidgetDefinition` if it's widget-scoped - most widget options don't need a change here at all, see
   `references/adding-a-widget.md` instead).
2. Give it a sensible default so existing configs that predate the key keep working unchanged.
3. If the key has bounds, use `Range<T>` rather than hand-rolled clamping so GUI sliders and validation agree.
4. Add or extend coverage in `tests/config_schema_roundtrip_test.cpp` - this test exists specifically to catch
   fields that serialize one way and deserialize another, or that lose defaults on round-trip.
5. Document the new key/section in `example.toml` if it's something a user would reasonably set.

## Renaming or changing the meaning of an existing key - migrations

Do **not** just change how an existing key is read - existing users' config files on disk still have the old
shape. Add a migration instead. Migrations are versioned, applied in order, and recorded via `config_version` in
the config file (`kConfigVersionKey` in `config_migrations.h`).

Pattern (see `src/config/config_migrations.cpp`):

```cpp
// 1. A named version constant, incremented from the current highest:
constexpr int kMyNewMigrationVersion = 13;  // one higher than the current max in the file

// 2. A migration function that mutates the toml::table in place and reports what changed:
bool migrateMyThing(toml::table& table, std::string_view path, OnChanged& onChanged) {
  // read old shape, write new shape, call onChanged(pathString) for each touched key
}

// 3. Register it in configMigrations():
{
    .toVersion = kMyNewMigrationVersion,
    .summary = "short human-readable description of what this migration does",
    .apply = migrateMyThingSidecar,
},
```

Migrations run in ascending `toVersion` order against whatever version a loaded file claims, bringing it up to
current. Keep each migration narrowly scoped to one behavioral change - the `summary` field is what a user or
future contributor reads to understand config history, so make it stand alone.

## Checklist for schema/migration changes

- [ ] New key has a default that preserves old-config behavior
- [ ] Bounds expressed via `Range<T>`, not hand-rolled clamps
- [ ] `tests/config_schema_roundtrip_test.cpp` covers the new/changed field
- [ ] Renames/behavior changes to *existing* keys go through a migration, not a silent reinterpretation
- [ ] `example.toml` updated if user-facing
- [ ] `just build && just test debug --print-errorlogs && just lint && just format`
- [ ] Canonical key names used consistently (per the PR checklist in the main SKILL.md) - grep for the old name
      across `src/`, `assets/templates/`, and `example.toml` before renaming anything
