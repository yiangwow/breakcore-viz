# breakcore-viz

A music visualizer in WebGL2. Drop in an audio file and fly through a tunnel of
shapes toward a black hole. Kicks, snares and hats each drive different things.
It can also drive Philips Hue lights.

It was built around breakcore. The hit effects and the style meter borrow from
ULTRAKILL.

## Run

Open `static/index.html` in a browser. It needs no server.

With Docker:

```bash
docker-compose up        # http://localhost:8086
```

With the Rust server:

```bash
cargo build --release
./target/release/breakcore-viz    # http://localhost:3000
```

## Use

1. Drop an audio file on the page, or click to pick one.
2. Pick a mode:
   - **Real-time** starts at once and only reacts to what it hears.
   - **Analyzed** scans the whole track first, so it can see drops coming.

| Key | Does |
|---|---|
| `Space` | Pause |
| `F` | Fullscreen |
| `D` | Debug readout |
| Scroll wheel | Volume |

URL flags, for testing:

| Flag | Does |
|---|---|
| `?debug` | Start with the debug readout on |
| `?demo` | Fake audio, no file needed |
| `?minimal` | Turn the 2D overlay off |
| `?nopost` | Skip post-processing |

## How it works

Everything is in one file, `static/index.html`.

**Audio.** The sound is split into four bands:

| Band | Range | Mostly |
|---|---|---|
| sub | 20 to 100 Hz | kicks |
| low | 100 to 500 Hz | snares, bass |
| mid | 500 to 2000 Hz | melody, vocals |
| high | 2000 to 16000 Hz | hats |

Sub counts the most toward the overall energy, so drums drive the big reactions.

**Song sections.** The track is sorted into breakdown, buildup, drop and sustain.
The camera slows in a breakdown, speeds up through a buildup and surges on a drop.

**Picture.** Three layers, back to front:

1. A raymarched scene shader, drawn at 0.4x resolution.
2. A post-processing shader: bloom, colour fringing, feedback trails.
3. A 2D canvas overlay: stars, spectrum ring, particles, style meter.

## Hue lights

`hue-service/` is a small Node service with no dependencies. The page posts the
band levels to it, and it sets three lights. This only happens when the page is
served from a host other than `localhost`.

Set these in `.env`:

```env
HUE_BRIDGE_IP=<bridge address>
HUE_API_KEY=<your api key>
HUE_LIGHT_BIG=1       # room light
HUE_LIGHT_PC=2        # desk light
HUE_LIGHT_SCREEN=3    # light behind the screen
```

Inside Docker the visualizer is on port 8086 and the Hue service on 8087.

## Deploy

```bash
bash deploy.sh [user@host]
```

## Files

| Path | What |
|---|---|
| `static/index.html` | The visualizer: shaders, audio, overlay |
| `src/main.rs` | Rust server that serves the page |
| `hue-service/server.js` | Hue lights service |
| `docker-compose.yml` | Runs both |
| `deploy.sh` | Copies it to a server and starts it |

## License

MIT
