# Changelog

## 2026-07-20 - DHW override and manual today cleanup

Branch: `codex-dhw-override-manual-today`

Commits:
- `93a89b3` - `chore(dhw): note manual today cleanup follow-up`
- `29eaef0` - `feat(dhw): add timed temperature override`
- `a0da271` - `fix(dhw): make manual today slots self-enable`

### Timed DHW override

Added a timer-based DHW override path for short manual heating periods, such as "heat water for the next hour".

New/updated controls:
- `input_number.dhw_boost_minutes` controls how long the override runs.
- `input_number.dhw_boost_temp` controls the temporary DHW target temperature.
- `timer.dhw_boost` represents the active override period.
- `input_button.dhw_boost_start` starts the override.
- `input_button.dhw_boost_cancel` cancels it.

Behavior:
- When the boost starts, `timer.dhw_boost` starts for the configured duration.
- Boost does not press the Viessmann/Vicare one-time charge buttons.
- While the timer is active, `binary_sensor.dhw_should_heat_now` treats boost as an active heating reason.
- `automation.dhw_temperature_control_auto` now also reacts to `timer.dhw_boost` state changes.
- While boost is active, DHW target temperature is set from `input_number.dhw_boost_temp`.
- When boost ends or is cancelled, temperature is restored to normal target or eco target depending on the rest of the DHW logic.

Why:
- The old manual slot flow was too clumsy for "heat now for one hour".
- A timer-based override is easier to use from the dashboard and does not require editing start/end clock values.

Important note:
- This override should be a compressor-friendly target-temperature override, not Viessmann/Vicare "one-time charge".
- The Viessmann one-time charge can enable high-consumption electric resistance heating and may make the tank temperature look high before the whole tank is actually heated.

### Manual today slots

Changed the manual-today flow so today's manual slots can be used directly from the dashboard.

Before:
- `input_datetime.dhw_manual_today_start_slot*` and `end_slot*` did nothing unless `input_boolean.dhw_manual_today_enable` was also on.
- The enable helper was easy to miss in the UI.
- `dhw_actuator_control` could clear today's manual slots when `binary_sensor.dhw_should_heat_now` changed, which made manual today feel unreliable.

After:
- Added `automation.dhw_manual_today_enable_from_slots`.
- When today slot values are changed and at least one slot is valid, `input_boolean.dhw_manual_today_enable` is turned on automatically.
- If all today slots are cleared/invalid, the enable helper is turned off.
- Added `input_boolean.dhw_manual_today_enable` to the dashboard so the current enable state is visible.
- Repurposed `automation.dhw_actuator_control` into `DHW manual today end-of-day cleanup`, running at `23:55` instead of on every `dhw_should_heat_now` state change.

Why:
- Manual tomorrow already worked because midnight copy also enabled manual today.
- Manual today needed the same "make it active when a valid slot exists" behavior.
- Cleanup should happen at the end of the day, not when heating decisions change.

### Validation notes

Checked:
- `git diff --check` passed.
- The boost duration template was evaluated against live HA state and returned `01:00:00` for 60 minutes.

Not checked yet:
- Full Home Assistant config check after deploying these files to the HA server.
- Runtime behavior after reloading packages/templates/automations on the live instance.
