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

### Updating `/remote-control` without reflashing (A)

This fork supports updating the `/remote-control` page **without** reflashing firmware by using WLED’s filesystem editor:

- The route handler for `/remote-control` will first try to serve a file from the device filesystem at:
  - **`/remote-control.htm`** (or **`/remote-control.htm.gz`**)
- If that file does not exist, it falls back to the built-in (compiled-in) `remote-control` page.

Practical workflow:

- **1) Ensure settings are unlocked (if you use a PIN)**:
  - In WLED UI: `Config → Security & Updates` and unlock settings if needed.
- **2) Open the filesystem editor**:
  - `http://<wled-ip>/edit`
- **3) Upload your updated page**:
  - Upload the file **named exactly** `remote-control.htm` to the filesystem root (so it becomes `/remote-control.htm`).
- **4) Reload**:
  - Open `http://<wled-ip>/remote-control` and it should now serve your uploaded file.

Notes:

- If you upload a gzipped page as `remote-control.htm.gz`, WLED will serve it as well.
- To revert to the built-in page, delete `/remote-control.htm` (and `/remote-control.htm.gz`) in `/edit`.

### iOS “Not Secure” note (home screen web app)

If you add `http://.../remote-control` to the iOS home screen, iOS may still warn that it is not HTTPS.
That warning cannot be removed by changing the WLED page alone; it requires serving the page over **HTTPS with a certificate iOS trusts**.

Practical solution: put an HTTPS reverse proxy (Apache/Nginx/Caddy/Nginx Proxy Manager) in front of WLED and access:

- `https://<your-hostname>/remote-control`

#### Practical LAN setup (ioBroker Apache on high port)

One working approach is to run an Apache reverse proxy on your ioBroker host and expose WLED over HTTPS on a **high port** (example: `9443`) so you do not need ports 80/443.

Example URLs (after proxy setup):

- Remote UI: `https://192.168.0.50:9443/remote-control`
- JSON API: `https://192.168.0.50:9443/json/state`

If you use a local/self-signed CA, iOS will only stop warning after you install and trust that CA certificate on the phone.

### What the remote UI controls do

- **Daniel / Gabriela selector**: chooses which segment is being controlled:
  - Daniel = `seg.id = 0` and uses `fx = 219` (`Lightbar Right`)
  - Gabriela = `seg.id = 1` and uses `fx = 218` (`Lightbar Links`)
- **ON**: restores last-used state for that segment:
  - If the segment is on the Lightbar effect for that side, sends `"on": true` + `"o1": true`.
  - If the segment is on a different effect, sends only `"on": true` (no effect override).
- **OFF**:
  - If the segment is on the Lightbar effect, sends `"on": true` + `"o1": false` (OFF-with-animation; effect later turns `seg.on=false`)
  - Otherwise sends `"on": false` (normal WLED OFF)
- **ON/OFF indication in the remote UI**: a status bar at the top of the control card shows Daniel + Gabriela state (ON / OFF / OFF (anim)).
- **Brightness +/- and slider**: updates `seg.bri`
- **More LEDs / Less LEDs**: steps through a **fixed ordered list of inclusive LED ranges** per bedside side and per mode (Top/Side).
  - **More LEDs** = next step; **Less LEDs** = previous step
  - The list is **cyclic** (wrap-around): past max → min, past min → max
  - This matches the “canonical step list” spec (two-phase expansion) you provided.
- **Top / Side**:
  - Applies the per-side anchor windows above
  - If **Tight mode** is enabled (Advanced), uses the `min` window; otherwise uses `max`
  - Per bedside side, the remote also remembers **last-used** window (`c1/c2`), brightness, and color **separately for Top and Side**:
    - When you switch Top↔Side, it restores the last-used state for the target mode.
    - This memory is stored in the browser’s `localStorage` (per phone/browser), not on the WLED device.
- **Warmer / Colder**: nudges the segment’s primary color step-by-step towards a warmer or colder reference white (keeps the effect).
- **Color picker “Apply”**: applies only the selected RGB color to the segment’s primary color (does not change brightness/window/effect).
- **Advanced**:
  - `sx`, `ix`, `c3`, `o2`, `o3`, and Tight mode
  - Changes are applied to the selected segment via `/json/state` (sliders apply on release; checkboxes apply immediately).


