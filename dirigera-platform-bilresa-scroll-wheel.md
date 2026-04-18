# dirigera_platform: BILRESA scroll wheel support

**Status:** working patch, not yet upstreamed  
**Tested with:** dirigera_platform 0.2.12, HA OS 2026.4.3, BILRESA scroll wheel fw 1.8.7

## Problem

The BILRESA scroll wheel remote pairs with Dirigera and appears in HA, but button presses are silently dropped. No trigger events fire.

Two root causes:

1. `"BILRESA scroll wheel"` is absent from `CONTROLLER_BUTTON_MAP` in `base_classes.py`, so the integration treats each sub-device as having 1 button but never registers it correctly for event dispatch.

2. The scroll wheel exposes three sub-devices with IDs `<uuid>_3`, `<uuid>_6`, `<uuid>_9` — one per group. The `hub_event_listener` normalizes any `_N` suffix to `_1` for registry lookup, but `_1` doesn't exist for this device, so every press event logs "not found in registry, ignoring" and returns.

The dual-button variant (`_1`, `_2`) works fine because the normalization to `_1` lands on a real registry entry. The scroll wheel's non-standard suffix step of 3 breaks this assumption.

## Patch

Both files live at `/config/custom_components/dirigera_platform/` on the Pi.

### base_classes.py

Add scroll wheel to `CONTROLLER_BUTTON_MAP`:

```python
CONTROLLER_BUTTON_MAP = {
    ...
    "BILRESA dual button": 2,
    "BILRESA scroll wheel": 1,  # each sub-device (_3/_6/_9) represents one group
}
```

Setting `1` treats each sub-device as a single-button controller, producing triggers `single_click`, `long_press`, `double_click` per group — no `buttonN_` prefix.

### hub_event_listener.py

After the `_1` registry lookup fails, fall back to the original device ID and strip the `buttonN_` prefix from the trigger type:

```python
# original code:
registry_value = hub_event_listener.get_registry_entry(device_id_for_registry)

if registry_value is None:
    # Fallback: some remotes (e.g. BILRESA scroll wheel) register sub-devices under
    # their actual IDs (_3/_6/_9) rather than the normalized _1 base.
    registry_value = hub_event_listener.get_registry_entry(device_id)
    if registry_value is None:
        logger.debug(f"remotePressEvent: Controller {device_id_for_registry} not found in registry, ignoring...")
        return
    # Strip the buttonN_ prefix — sub-device is a single-button entity
    if trigger_type.startswith("button") and "_" in trigger_type:
        trigger_type = trigger_type.split("_", 1)[1]
    logger.debug(f"remotePressEvent: Found sub-device directly at {device_id}, trigger: {trigger_type}")
```

## Result

Each BILRESA scroll wheel group fires HA device triggers on its own entity:

| Sub-device | HA entity | Triggers |
|---|---|---|
| `<uuid>_3` | `sensor.bilresa_green` | `single_click`, `double_click`, `long_press` |
| `<uuid>_6` | `sensor.bilresa_green_2` | `single_click`, `double_click`, `long_press` |
| `<uuid>_9` | `sensor.bilresa_green_3` | `single_click`, `double_click`, `long_press` |

## Scroll wheel gesture

The scroll gesture sends Matter `LevelControl` commands directly to the bound IKEA device over Thread, bypassing Dirigera's event system. No scroll events reach HA. Only click events (single/double/long press) are available.

Workaround: assign a dummy IKEA device (e.g. a bulb in storage) to each group in the IKEA app — this activates the group LEDs and keeps the IKEA-side scene binding alive, without affecting HA control.

## TODO

- Fork `sanjoyg/dirigera_platform`, apply on a branch, install via HACS from fork
- Open upstream PR
