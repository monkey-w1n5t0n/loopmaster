Rendering Technology

  All widgets are plain Canvas 2D — no Two.js, no WebGL, no SVG, no DOM elements for visualizations. Every widget receives a draw(ctx: CanvasRenderingContext2D, x, y,
  w, h) callback and paints directly onto a shared canvas managed by the editor package.

  The editor package (a custom CodeMirror-like editor at github:loopmaster-xyz/editor) owns a single <canvas> that renders both the code text and the inline widgets.
  Widget types are defined by the editor as a simple interface:

  type Widget = {
    type: 'above' | 'full' | 'before'  // placement relative to a code line
    pos: { x?: number | [number, number], y: number, width?: number }
    draw: (ctx: CanvasRenderingContext2D, x: number, y: number, w: number, h: number) => void
    onMouseDown?: (event: MouseEvent) => void
  }

  - above — draws above the code line, spanning a column range (e.g. waveforms, ADSR, filters)
  - full — full-width widget above the line (e.g. Mix waveform)
  - before — small inline widget before a token (e.g. knob dials before number literals)

  Widget Structure

  Each widget file (src/widgets/*.ts) exports a create*Widget() factory that:

  1. Locates itself in the source — uses target.source.line / target.source.column from the DSP history to map back to the code position
  2. Creates a cache key — makeWidgetCacheKey('Adsr', startIndex, endIndex) keyed by source position so widgets survive re-renders without losing state
  3. Wraps a history reader — createHistoryReader() from the engine provides latency-compensated reads of audio thread data
  4. Returns a Widget object with the draw() callback

  The widget cache (src/widgets/cache-key.ts) handles source text edits — when the user types, relocateWidgetCacheKeys() adjusts all cache keys' start/end positions
  based on the splice delta, so widgets don't get recreated on every keystroke.

  Audio Thread → Main Thread Data Flow

  This is the most interesting part. There are two distinct mechanisms:

  1. Ring Buffers (for waveform/audio data)

  Used by wave.ts for waveform, spectrum, and amplitude visualizations:

  - The engine's WASM audio worklet writes audio samples into a SharedArrayBuffer ring buffer (outputRing)
  - WaveformBuffer (src/lib/waveform-buffer.ts) reads chunks from this ring on the main thread using ring[chunkIdx] — this is lock-free because SharedArrayBuffer is
  shared between the worklet and main threads
  - The chunkPos write cursor is updated by the audio worklet; the main thread reads it and maintains a readChunkPtr with a safety lag of 4 chunks to avoid reading
  data that's still being written
  - The read data gets downsampled into a display buffer (Float32Array of 8192 samples) for drawing

  2. History Readers (for parameter/state data)

  Used by ADSR, filter, compressor, LFO, piano, every, reverb, etc:

  - Each DSP generator (e.g. Adsr, Biquad, Compressor) writes its parameter values into typed history arrays — also backed by SharedArrayBuffer
  - createHistoryReader() from the engine creates a reader that:
    - Knows the ring buffer size and mask (power-of-2 ring buffer)
    - Has a run(epoch) method that reads the latest values, compensating for audio latency
    - Uses writeIndex (written by audio thread) and sampleCounts to determine which history entry is current
    - Reads individual parameters via .at(index) on the typed arrays
  - The reader maintains a state object (e.g. { stage, attack, decay, sustain, release, exponent, env } for ADSR) that gets updated each frame

  3. The Animation Pump

  In App.tsx:
  const tick = () => {
    tickCount.value++  // signal increment triggers all reactive effects
    draw()             // editor repaints canvas (text + widgets)
    rafId = requestAnimationFrame(tick)
  }

  tickCount is a Preact signal. Incrementing it triggers reactive effect() chains in dsp.ts that:
  - Call program.refreshHistories() to pick up new history data from the audio thread
  - Update the histories signal, which triggers widget recreation via doc.widgets = createWidgets(...)
  - The editor then calls each widget's draw() during its canvas paint pass

  Key Design Decisions Worth Noting

  1. Single canvas for everything — the editor renders code text AND widgets on one canvas. No DOM overlay, no separate canvases per widget. This avoids layout
  thrashing and compositing costs.
  2. SharedArrayBuffer for zero-copy audio data — enabled via COOP/COEP headers (vite-plugin-coop-coep). The audio worklet and main thread share memory directly. No
  postMessage copying.
  3. Latency compensation — createHistoryReader accounts for audio pipeline latency so the visualization shows what you're hearing, not what the audio thread is
  currently processing.
  4. Widget caching with position tracking — rather than recreating widgets every frame, they're cached by source position and relocated when text edits happen. The
  historyRef pattern (a mutable { current } ref) lets cached widgets point to fresh history data without recreating the widget object.
  5. Reactive signals as the glue — @preact/signals connects the animation loop (tickCount), DSP state, and widget creation. Effects react to signal changes and
  propagate updates without manual wiring.
  6. No framework for the hot path — widgets are pure imperative Canvas 2D drawing. Preact/signals handle the orchestration (when to redraw, which widgets exist), but
  the actual rendering is just ctx.beginPath(), ctx.lineTo(), ctx.stroke() etc.

  For your CodeMirror + Two.js setup: the closest analog would be CodeMirror decorations/widgets that contain Two.js canvases, reading from SharedArrayBuffers filled
  by your audio worklet, with some kind of rAF loop triggering redraws. The main complexity this codebase solves elegantly is the latency-compensated history reader
  pattern and the cache-key relocation on text edits.
