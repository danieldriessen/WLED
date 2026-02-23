# Bedside “Lightbar” custom WLED effects

This fork adds two custom effects intended to run on **two separate segments** of a single physical strip (left + right bedside “virtual lamps”):

- `Lightbar Links` (left bedside): **ground is segment-local `len-1`**
- `Lightbar Right` (right bedside): **ground is segment-local `0`**

Both effects run all animations **on-device** and are controlled via standard WLED per-segment JSON fields.

## Effect IDs (this firmware)

On your device build the effect IDs are:

- **`Lightbar Links` = `218`**
- **`Lightbar Right` = `219`**

## Parameter contract (per segment)

These map to WLED’s standard segment effect parameters:

- **`sx` (0–255)**: Expand/fill speed (adds pixels)
- **`ix` (0–255)**: Retract/off speed (removes pixels)
- **`c1` (0–255)**: Target bound A (segment-local, scaled to `0..SEGLEN-1`)
- **`c2` (0–255)**: Target bound B (segment-local, scaled to `0..SEGLEN-1`)
- **`c3` (0–31)**: Early-retract overlap (WLED stores `c3` as 5-bit)
  - Effect maps this to **0..80 LEDs**:
  - `earlyRetractLeds ≈ round(c3 * 80 / 31)`
  - For ~35 LEDs overlap, use **`c3 = 14`**.
- **`o1`** (`check1`): Desired power (virtual) — `true` = ON, `false` = OFF-with-animation
- **`o2`** (`check2`): OFF realism — progressive speed-up + brightness swell near ground
- **`o3`** (`check3`): No-anim/debug — jumps immediately to steady ON/OFF (no transitions)

## Behavior summary

- **Steady ON**: only the window `[A..B]` is lit (using the segment’s primary color).
- **ON (o1 false→true)**: fill from ground toward the far bound, then retract from ground to the near bound.
- **Target change while ON**: window transitions smoothly; when expanding, retract begins early based on `c3`.
- **OFF (o1 true→false)**: window falls to ground, then shrinks/fades into ground; effect then sets the segment `on=false`.

## Segment setup (typical for your 262 LED strip)

Physical indices: `0..261`

One common segmentation for “two lamps on one strip”:

- **Right bedside segment**: `start=0`, `stop=131` (covers 0..130), use effect **`Lightbar Right`**
- **Left bedside segment**: `start=131`, `stop=262` (covers 131..261), use effect **`Lightbar Links`**

Keep segments **non-overlapping** and command them independently by `seg.id`.

Note: WLED’s segment **Reverse** option (`"rev": true`) flips segment-local coordinates. For predictable “ground” orientation, keep `"rev": false`. If you do reverse a segment, use the opposite Lightbar effect (or invert your `c1/c2` mapping).

## Anchor windows (Top/Side, min/max)

Your physical topology (global indices):

- Total: `0..261`
- Right wall: `0..77` (bottom → ceiling)
- Ceiling top-right: `78..130` (right → center)
- Ceiling top-left: `131..183` (center → left)
- Left wall: `184..261` (ceiling → bottom)

Right-side anchor ranges you provided (global, and also segment-local for Daniel because his segment is `0..130`):

- **Right Top min**: `108..116`
- **Right Top max**: `78..130`
- **Right Side min**: `23..31`
- **Right Side max**: `15..77`

Left-side anchors are mirrored with `LAST=261`:

`[a..b]right → [LAST-b .. LAST-a]left`

Converted to **Gabriela segment-local** (`local = global - 131`):

- **Left Top min**: global `145..153` → local **`14..22`**
- **Left Top max**: global `131..183` → local **`0..52`**
- **Left Side min**: global `230..238` → local **`99..107`**
- **Left Side max**: global `184..246` → local **`53..115`**

## JSON API usage examples

All calls are `POST http://<wled-ip>/json/state`.

### Turn ON (pick window + tuning)

#### Right bedside (Daniel / `seg.id = 0`): `Lightbar Right` (`fx = 219`)

```json
{
  "seg": [
    {
      "id": 0,
      "on": true,
      "fx": 219,
      "sx": 90,
      "ix": 120,
      "c1": 140,
      "c2": 180,
      "c3": 14,
      "o1": true,
      "o2": true,
      "o3": false
    }
  ]
}
```

#### Left bedside (Gabriela / `seg.id = 1`): `Lightbar Links` (`fx = 218`)

```json
{
  "seg": [
    {
      "id": 1,
      "on": true,
      "fx": 218,
      "sx": 90,
      "ix": 120,
      "c1": 140,
      "c2": 180,
      "c3": 14,
      "o1": true,
      "o2": true,
      "o3": false
    }
  ]
}
```

### Switch “mode” while ON (just update bounds)

```json
{
  "seg": [
    { "id": 0, "c1": 108, "c2": 130 }
  ]
}
```

### Turn OFF with animation (important)

Do **not** send `"on": false` directly (that stops the effect immediately).

Instead keep `"on": true` and set the virtual desired power **`o1=false`**:

```json
{
  "seg": [
    { "id": 1, "on": true, "o1": false }
  ]
}
```

The effect will finish the OFF animation and then set the segment `on=false` internally so the UI reflects OFF.

## Build notes (local)

This repo builds the WLED web UI as part of PlatformIO builds (runs `npm ci` / `npm run build`).

Example compile command:

```bash
npm_config_cache="$(pwd)/.npm-cache" pio run -e esp32dev
```

Pick the PlatformIO environment that matches your controller/board.

## Update/flash process (practical)

- **Build** the firmware (command above).
- Flash via WLED OTA:
  - Open `http://<wled-ip>/update`
  - Upload `build_output/release/WLED_16.0-alpha_ESP32.bin`
- After flashing, verify:
  - `/json/info` shows `repo: danieldriessen/WLED`
  - `/json/effects` includes `Lightbar Links` and `Lightbar Right`

## Built-in bedside remote UI

This fork also includes a mobile-first remote page served directly from the device:

- `http://<wled-ip>/remote-control`

It provides a Daniel/Gabriela selector and sends the correct per-segment `/json/state` commands (including OFF-with-animation via `o1=false`).

### What the remote UI controls do

- **Daniel / Gabriela selector**: chooses which segment is being controlled:
  - Daniel = `seg.id = 0` and uses `fx = 219` (`Lightbar Right`)
  - Gabriela = `seg.id = 1` and uses `fx = 218` (`Lightbar Links`)
- **ON**: sends `"on": true` + `"o1": true` (desired power ON) and ensures correct `fx`
- **OFF**: sends `"on": true` + `"o1": false` (OFF-with-animation; effect later turns `seg.on=false`)
- **Brightness +/- and slider**: updates `seg.bri`
- **More LEDs / Less LEDs**: grows/shrinks the window **away from ground** (by nudging the window’s far bound)
- **Top / Side**:
  - Applies the per-side anchor windows above
  - If **Tight mode** is enabled (Advanced), uses the `min` window; otherwise uses `max`
- **Warm / Cool**: sets the segment’s primary color to a warm-ish or cool-ish RGB value (keeps the effect)
- **Advanced**:
  - `sx`, `ix`, `c3`, `o2`, `o3`, and Tight mode


