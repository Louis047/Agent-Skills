# Adding or editing a widget

Bar, dock, and desktop widgets in Noctalia all follow the same three-piece pattern: an implementation class, a
declarative field **definition**, and a **factory** registration that ties a config `type` string to both.

Worked example: the battery widget (`src/shell/bar/widgets/battery_widget*`).

## 1. The widget implementation

`battery_widget.h` / `battery_widget.cpp` - the actual rendering/behavior class. It exposes an `Options` struct
(`BatteryWidget::Options`) holding every user-configurable field as a plain typed member (bool, enum, string,
ColorSpec, etc.). This is the thing the definition below fills in from config.

## 2. The field definition

`battery_widget_definition.h` / `.cpp` declares, once, how each `Options` member maps to a config key, its
schema type, and (optionally) its GUI presentation:

```cpp
static const noctalia::bar::WidgetDefinition<Options, BatteryWidgetDefinitionContext> definition{
    .type = "battery",  // the string used in TOML / IPC to name this widget
    .fields = {
        field<&Options::displayMode>({
            .key = "display_mode",
            .choices = {
                {.value = BatteryDisplayMode::None,    .configValue = "none",    .labelKey = "settings.widgets.options.none"},
                {.value = BatteryDisplayMode::Glyph,   .configValue = "glyph",   .labelKey = "settings.widgets.options.glyph"},
                {.value = BatteryDisplayMode::Graphic, .configValue = "graphic", .labelKey = "settings.widgets.options.graphic"},
            },
        }),
        field<&Options::showLabel>({.key = "show_label"}),
        field<&Options::labelContent>({
            .key = "label_content",
            .choices = { /* ... */ },
            .presentation = settings::WidgetSettingPresentation{
                .visibleWhen = settings::WidgetSettingVisibility{"show_label", {"true"}},  // GUI conditional visibility
                .horizontalBarOnly = true,
            },
        }),
        // ...
    },
    .finalize = [](Options& options, const BatteryWidgetDefinitionContext& context) {
        // Post-process fields that depend on external services/config, run after all fields resolve.
    },
};
```

Key points:
- `field<&Options::member>({...})` binds one config key to one `Options` member. The schema type (bool/int/
  double/enum/...) is inferred from the member's C++ type unless `choices` makes it an enum.
- `.presentation` controls how the settings GUI renders/hides the field (e.g. `visibleWhen`, `horizontalBarOnly`,
  `optionSource` for dynamically-populated choices like device lists).
- `labelKey`/`descriptionKey`-style strings are i18n keys, not literal text - they must exist in
  `assets/translations/en.json` (see the i18n rules in the main SKILL.md).
- `.finalize` is for values that can't be derived from a single config key alone (e.g. computing a threshold
  from both the widget's own config and a shared service).
- The `Context` struct (`BatteryWidgetDefinitionContext` here) carries whatever external pointers `.finalize`
  or field resolution needs - keep it minimal, just what this widget actually requires.

## 3. Registering in the factory

`src/shell/bar/widget_factory.cpp` maps the `type` string to a construction lambda:

```cpp
{"battery", [](const WidgetFactory& f, const BuiltinWidgetContext& context) {
    return /* construct BatteryWidget, calling */
        batteryWidgetDefinition().resolve(/* ..., */ .batteryConfig = &f.m_config.battery, /* ... */);
}},
```

This is the map new widgets must be added to. Missing this step means the widget compiles but is never
reachable from config.

## 4. Wiring it into the build

Add both new `.cpp` files to `_noctalia_sources` in the root `meson.build` (see the meson gotcha in the main
SKILL.md - there is no glob, this is a hard requirement or the widget silently isn't compiled in).

## 5. Checklist for a new widget

- [ ] `Options` struct with every configurable field, plain typed members
- [ ] `WidgetDefinition` mapping each field to a config key + schema type + presentation
- [ ] Any new choice/enum labels or descriptions added to `assets/translations/en.json`
- [ ] Registered under a unique `type` string in `widget_factory.cpp`
- [ ] New `.cpp` files added to `meson.build`
- [ ] `example.toml` updated if this widget should appear there as a documented default
- [ ] `just build && just test debug --print-errorlogs && just lint && just format`
- [ ] If it's a bar widget, sanity-check it in both horizontal and vertical bar orientations if `presentation`
      options like `horizontalBarOnly` are relevant
