# 3ds Max Python bridge

Drive 3ds Max from any external Python process. A small socket server runs
inside 3ds Max; a client class on your side sends it commands and gets the
result back as plain Python values.

Part of [Atlas](https://github.com/ssnfts/atlas). Extracted as a standalone
tool; nothing here imports anything outside this folder.

## What it does

- **Runs every command on the 3ds Max main thread.** `pymxs` is not
  thread-safe, so a call goes socket thread → queue → QTimer on the main
  thread → `pymxs` → reply. The caller still sees an ordinary blocking call.
- **Verifies writes.** After setting a property the bridge reads it back. A
  value 3ds Max accepted and then ignored is reported as *rejected*, not as
  applied.
- **Covers the common jobs with dedicated commands:** read and set node
  properties, list the scene, create meshes (single or batched), set
  keyframes and the animation range, build and assign materials, set up a
  V-Ray sky or HDRI environment, choose the renderer, render, capture the
  viewport and save the scene.
- **Keeps raw MAXScript off by default.** Arbitrary code runs only when you
  start 3ds Max with `ATLAS_ALLOW_MAXSCRIPT=1`.

## Prerequisites

- 3ds Max with Python 3 and `pymxs` (developed and checked on 3ds Max 2027).
- PySide6 importable inside 3ds Max. The bridge refuses to start without it
  rather than fall back to unsafe direct calls.
- Python 3.11 or later for the client side. The client uses only the
  standard library.
- V-Ray, only if you use the V-Ray sky, HDRI or material commands.

## Install

```
git clone https://github.com/ssnfts/3dsMax-Python-bridge.git
```

There is nothing to build and no package to install. To run the tests you
also need `pip install pytest`.

## How to use it

**1. Start the bridge inside 3ds Max.** In the MAXScript Listener:

```
python.ExecuteFile @"C:\path\to\3dsMax-Python-bridge\bridge\start_bridge.py"
```

The Listener prints `[atlas] bridge listening on 127.0.0.1:9879`. Running the
launcher again stops and reloads the previous bridge, so it is safe to
repeat.

**2. Talk to it from your own Python.**

```python
import sys
sys.path.insert(0, r"C:\path\to\3dsMax-Python-bridge\server")
from maxbridge import MaxBridge

bridge = MaxBridge()                      # 127.0.0.1:9879
print(bridge.ping())                      # confirms 3ds Max is answering
print(bridge.scene_list())                # what is in the scene
bridge.node_set("Teapot001", "radius", 25.0)
bridge.viewport_capture(r"C:\temp\view.png")
```

`MaxBridge` also has `get`, `set`, `properties`, `node_get`, `batch`,
`create_mesh`, `create_meshes`, `set_keys`, `animation_range`,
`assign_material`, `build_material`, `vray_sky_setup`, `vray_hdri_env`,
`set_renderer`, `render`, `save_scene` and `maxscript`.

**Settings** (environment variables, set before 3ds Max starts):

| Variable | Default | Effect |
|---|---|---|
| `ATLAS_MAX_PORT` | `9879` | Port the bridge listens on and the client connects to |
| `ATLAS_ALLOW_MAXSCRIPT` | `0` | `1` enables the raw `maxscript` command |
| `ATLAS_DEV_RELOAD` | `1` | Reloads the command handlers between requests, so handler edits apply without restarting 3ds Max |

## Getting the best result

- **Check the reply, not just the absence of an error.** Write commands
  return what was read back from the scene. Look at `rejected` before you
  trust a change.
- **Batch large jobs.** `create_meshes` and `batch` send many steps in one
  main-thread slot. Thousands of single calls freeze the 3ds Max interface
  for long enough to look like a hang.
- **Use the dedicated commands before raw MAXScript.** They verify their own
  result; raw code does not.
- **Set scene units to metres first** if the geometry you send is in metres.
  3ds Max defaults to inches and a wrong unit raises no error.
- **Pick a free port** with `ATLAS_MAX_PORT` if another tool already uses
  9879.

## Works well with

**Graphic design.** Script a batch of stills: swap a material or colourway,
move the camera, call `render`, repeat. Bring the frames into Photoshop,
Affinity or Figma for layouts and key art.

**Product visualization.** Drive turntables and variant renders from a
spreadsheet or a JSON file. Pair it with
[V-Ray material graph builder](https://github.com/ssnfts/V-Ray-material-graph-builder)
for the materials and
[Cinematic camera moves](https://github.com/ssnfts/Cinematic-camera-moves)
for the camera.

**Film and previs.** It is the transport under
[Atlas](https://github.com/ssnfts/atlas) (real sun and sky for a date and
place), [CityGen](https://github.com/ssnfts/CityGen) (real-world city
massing) and
[tyFlow script generator](https://github.com/ssnfts/tyFlow-script-generator)
(smoke, debris, sparks).

**Automation.** Put [3dsmax-mcp](https://github.com/ssnfts/3dsmax-mcp) on top
and an AI agent can call the same commands as MCP tools. A scheduler, a CI
job or a render-queue script can use the client directly.

## Layout

```
server/   maxbridge.py, the client you import
bridge/   the half that runs inside 3ds Max
tests/    offline tests
```

## Tests

```
python -m pytest tests -q
```

The tests are offline and need no 3ds Max and no network.

## Data licensing

This is source code, licensed however you choose. The *data* it fetches is
not:

- OpenStreetMap is ODbL 1.0 -- attribution, and share-alike on derived
  databases.
- Copernicus GLO-30 requires attribution.
- Poly Haven textures are CC0.

`attribution.py`, where included, generates the manifest for you.

## Compatibility

Every 3ds Max and V-Ray parameter name in this code was read off a live
3ds Max 2027 with V-Ray 7 update 3. Names are discovered from the host rather
than recalled, but they were discovered on *that* host. Treat other versions
as unverified until checked.
