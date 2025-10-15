sequence store.py started life as a barebones Dorothy sketch: a window, a background, and a simple grid of booleans. The earliest seed was nothing more than the data structure we still use today:

```
grid = [[False] * STEP_COUNT for _ in GRID_ROWS]
```

That array of False values was the heart of the idea—each row would become a drum voice, each column a beat. Once we could toggle those booleans, the next milestone was teaching Dorothy to render them. The first draw loop that felt like progress wrapped rectangles over the data:

```
dot.fill(colour)
dot.rectangle((cell_x, row_y), (cell_x + CELL_SIZE, row_y + CELL_SIZE))
```

Clicking a cell flipped the boolean and immediately repainted the square. That interaction proved the seed: we had a living step grid and, more importantly, a visible sequence that reacted as we advanced through it.

From there, the project grew layer by incremental layer. Audio playback was the next big leap. Dorothy ships with a DSP hook but no turnkey sampler, so we wrapped the raw stream with a tiny mixer able to replay overlapping hits:

```
def get_frame(size: int) -> np.ndarray:
    audio = np.zeros(size, dtype=np.float32)
    for idx, pos in enumerate(self.positions):
        if pos < 0:
            continue
        sample = self.samples[idx]
        end = min(pos + size, len(sample))
        chunk = sample[pos:end] * self.gains[idx]
        audio[: len(chunk)] += chunk
        self.positions[idx] = -1 if end >= len(sample) else end
        if self.positions[idx] == -1:
            self.gains[idx] = 0.0
    return np.clip(audio, -1.0, 1.0)
```

With that in place, toggling a square finally triggered a sound on the next step. Sequencing means little without distinct voices, so the project immediately bumped into another Dorothy constraint: there is no ready-made palette of drum samples. We built our own library on the fly, using numpy envelopes to give each row its own personality:

```
env = _exp_env(0.46, sr, 7.0)
t = np.arange(env.size) / sr
kit.append(_normalize(np.sin(2 * np.pi * 140 * t) * (0.6 + 0.4 * np.sin(2 * np.pi * 2.2 * t)) * env, 0.82))
```

That line is representative of the whole `build_sample_pack` function. Nine such blocks sculpt kicks, snares, hats, claps, cymbals, and more, so a single boolean in `grid[row][col]` maps directly to an audible instrument.

Timing came next. We used Dorothy’s millisecond clock to advance the step pointer, ensuring each beat landed where it should:

```
while now - last_step_time >= step_millis:
    last_step_time += step_millis
    current_step = (current_step + 1) % STEP_COUNT
    trigger_step(current_step)
```

This loop is the backbone of sequence control. Every cycle we compare real time against `step_millis`, then rotate through the columns. The `trigger_step` function makes the sequencing explicit:

```
def trigger_step(step_index: int) -> None:
    for row_index, row in enumerate(grid):
        if row[step_index]:
            sampler.trigger(row_index)
```

There is no Dorothy helper that does this for us; we iterate row by row, check the state we stored in the sequence data structure, and fire the appropriate action. That is sequencing logic in plain Python.

Because Dorothy lacks built-in fonts or buttons, we layered those ourselves. The display text you see is drawn pixel by pixel:

```
def draw_text(message: str, pos: Tuple[int, int], colour: Tuple[int, int, int] = (220, 220, 220), scale: int = 4) -> None:
    x_cursor = int(pos[0])
    y_cursor = int(pos[1])
    dot.fill(colour)
    chars = message.upper()
    for idx, ch in enumerate(chars):
        glyph = FONT_5X7.get(ch, FONT_5X7[" "])
        glyph_width = len(glyph[0])
        for row, row_data in enumerate(glyph):
            for col, bit in enumerate(row_data):
                if bit != "1":
                    continue
                x0 = x_cursor + col * scale
                y0 = y_cursor + row * scale
                dot.rectangle((x0, y0), (x0 + scale, y0 + scale))
        x_cursor += glyph_width * scale
        if idx < len(chars) - 1:
            x_cursor += scale
```

Even the layout had to be engineered by hand; we treat every UI element as part of another sequence, computing their positions one after another:

```
for idx, (key, _) in enumerate(BUTTON_ORDER):
    x1 = start_x + idx * (BUTTON_WIDTH + BUTTON_GAP)
    button_rects[key] = (x1, BUTTON_TOP, x1 + BUTTON_WIDTH, BUTTON_TOP + BUTTON_HEIGHT)
```

Mouse handling proved that Dorothy’s APIs vary between builds, so we pooled every known attribute name and returned a unified event:

```
def read_mouse() -> Tuple[bool, float, float]:
    down = any(bool(getattr(dot, attr, 0)) for attr in MOUSE_DOWN_ATTRS)
    for nx, ny in (("mouse_x", "mouse_y"), ("mouseX", "mouseY")):
        if hasattr(dot, nx) and hasattr(dot, ny):
            return down, float(getattr(dot, nx)), float(getattr(dot, ny))
    return down, 0.0, 0.0
```

Transport controls followed—start, stop, clear—and after living with the app we swapped the BPM buttons for a tap-tempo workflow. Users can now set tempo in the most musical way possible:

```
def tap_tempo() -> None:
    now = read_millis()
    tap_times.append(now)
    while tap_times and now - tap_times[0] > TAP_WINDOW:
        tap_times.pop(0)
    if len(tap_times) < 2:
        return
    intervals = [t2 - t1 for t1, t2 in zip(tap_times, tap_times[1:])]
    avg = sum(intervals) / len(intervals)
    if avg <= 0:
        return
    set_bpm(60000.0 / avg)
```

`set_bpm` keeps the loop anchored to musical time, rounding to the nearest five BPM for readability and recalculating the step length in milliseconds:

```
def set_bpm(value: float) -> None:
    global bpm, step_millis, last_step_time
    bpm = max(40, min(240, int(round(value / 5.0) * 5)))
    step_millis = 60000.0 / bpm
    last_step_time = read_millis()
```

Tap tempo exists because Dorothy only exposes an immediate millisecond clock—no higher-level timing helpers—so we gather the tap sequence ourselves, average the intervals, and update `step_millis`. It is sequencing in its purest form: turn a series of user events into tempo data and feed it straight back into the loop that advances `current_step`.

Every enhancement grew out of that original grid seed: start with toggling booleans, add visible feedback, give them sound, control the timing, label the interface, and finally dial in the musician-friendly UX. At each step we ran into Dorothy’s lightweight nature—no font helpers, no UI widgets, limited mouse APIs—and solved those gaps with sequencing knowledge: arrays of states, time-step loops, per-voice triggering, averaged tap intervals. The result is a compact Dorothy sequencer that still exposes its incremental evolution in the code. Whether you look at `grid`, `trigger_step`, `set_bpm`, or `tap_tempo`, each piece demonstrates an explicit command of how to represent, manipulate, and perform sequences in Python.
