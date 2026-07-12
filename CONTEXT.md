# CONTEXT — goodwe fork

This is a local fork of [marcelblijleven/goodwe](https://github.com/marcelblijleven/goodwe)
(v0.4.10), used exclusively for the GoodWe GW6000-SDT-30 (DT family, Modbus-TCP).

It is **not published to PyPI**. Install from this directory:
```bash
pip install .
```
In Docker, the `solar-logger` image builds it via `additional_contexts` (no pip publish
needed). See `../solar-logger/CONTEXT.md`.

## What we changed (vs. upstream marcelblijleven/goodwe)

Based on [mityacor/goodwe commit e401a12](https://github.com/mityacor/goodwe/commit/e401a12d415f120bfecc3b4c34e51b532706a4a5).

### `goodwe/dt.py`

**`iso_resistance` sensor added to `_READ_RUNNING_DATA`:**
```python
Decimal("iso_resistance", 30173, 1, "ISO Resistance", "kΩ", Kind.PV),
```
Register 30173 stores the PV insulation resistance in **kΩ directly** (no decimal scaling).
The mityacor patch used divisor=10, which is correct for the ET/hybrid family but wrong for
DT — DT stores the raw kΩ value (verified: 8321 kΩ from the inverter, confirmed via the
GoodWe app). We use divisor=1.

**`_READ_RUNNING_DATA` register count** extended from `0x0049` (73) to `0x004A` (74) to
include the iso_resistance register.

**`active_power`, `grid_in_out`, `grid_in_out_label` stubs added to `__all_sensors_meter`:**
```python
Calculated("active_power", lambda data: None, "Active Power", "W", Kind.GRID),
Calculated("grid_in_out", lambda data: None, "On-grid Mode code", "", Kind.GRID),
Calculated("grid_in_out_label", lambda data: None, "On-grid Mode", "", Kind.GRID),
```
These sensors require a smart meter. Our installation has no meter (`meter_comm_status=0`),
so they always return `None`. The stubs prevent `KeyError` when code references them.
The real `read_runtime_data()` patching logic (±90W threshold for import/export/idle) is
still present in `read_runtime_data()` — it just always results in the "Idle" path since
meter power is always 0.

### `tests/test_dt.py` and `tests/sample/dt/*.hex`

All 10 DT hex fixture files were extended by one register (appended `0x0000` before the CRC)
and the Modbus frame headers updated:
- Length byte at offset 4: +2 (covers the new register)
- Modbus CRC-16: recomputed over `frame[2:-2]` (skipping the `aa55` header bytes), stored
  little-endian (low byte first). The CRC algorithm is table-based Modbus CRC-16 as
  implemented in `goodwe/modbus.py:_modbus_checksum()`.
- TCP fixture (`GW6000-DT-tcp.hex`): TCP length at offsets 4–5 and data length at offset 8
  both incremented by 2; no CRC in TCP frames.

`test_dt.py` assertions updated: `len(data)` counts bumped by 1 per model (45→46, 35→36,
38→39, 51→52), and `assertSensor("iso_resistance", 0.0, "kΩ", data)` added for
GW6000-DT, GW50KS-MT, GW20K-SDT-20 test cases.

## What was NOT changed

- No changes to ET, ES, EH, or any other family.
- Upstream public API unchanged — `goodwe.connect()`, `read_runtime_data()`, `sensors()`.
- `setup.cfg` / `pyproject.toml` version left at upstream value; this fork is not versioned
  separately.
