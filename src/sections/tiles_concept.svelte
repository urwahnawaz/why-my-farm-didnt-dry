<script>
  import { onMount } from "svelte";
  import rough from "roughjs/bundled/rough.esm.js";

  // slider: 0–100 prompts
  let prompts = 0;

  // derived: 0–1 dryness
  $: dryness = prompts / 100;

  let arrowSvgEl;

  onMount(() => {
    // Draw a sketchy arrow into the inline SVG
    const svg = arrowSvgEl;
    if (!svg) return;

    // clear (in case of HMR)
    while (svg.firstChild) svg.removeChild(svg.firstChild);

    const rc = rough.svg(svg);

    // Arrow from left tile → right tile
    const w = 260;
    const h = 120;

    svg.setAttribute("viewBox", `0 0 ${w} ${h}`);

    // main line
    const line = rc.line(20, h / 2, w - 40, h / 2, {
      roughness: 1.6,
      strokeWidth: 3
    });

    // arrow head (triangle-ish)
    const head = rc.polygon(
      [
        [w - 40, h / 2],
        [w - 60, h / 2 - 14],
        [w - 60, h / 2 + 14]
      ],
      { roughness: 1.6, strokeWidth: 3, fill: "none" }
    );

    svg.appendChild(line);
    svg.appendChild(head);
  });
</script>

<div class="tile-concept">
  <div class="tiles">
    <!-- LEFT: morph tile -->
    <div class="tile-stack" aria-label="Tile morph demo">
      <img class="tile base" src="/green-tile.svg" alt="Green tile" />
      <img
        class="tile top"
        src="/patch-brown.svg"
        alt="Brown tile"
        style={`opacity: ${dryness};`}
      />
    </div>

    <!-- ARROW -->
    <div class="arrow-wrap" aria-hidden="true">
      <svg bind:this={arrowSvgEl} class="arrow"></svg>
    </div>

    <!-- RIGHT: show end state (optional) -->
    <div class="tile-stack">
      <img class="tile base" src="/tile-brown.svg" alt="Brown tile (end state)" />
    </div>
  </div>

  <!-- SIDEBAR -->
  <aside class="controls">
    <div class="controls-title">ChatGPT prompts</div>

    <div class="value-row">
      <div class="value">{prompts}</div>
      <div class="unit">/ 100</div>
    </div>

    <input
      type="range"
      min="0"
      max="100"
      step="1"
      bind:value={prompts}
      aria-label="ChatGPT prompts slider"
    />

    <div class="hint">
      Slide to see the farm tile “dry out” as more prompts are used.
    </div>
  </aside>
</div>

<style>
  .tile-concept {
    display: grid;
    grid-template-columns: 1fr 260px;
    gap: 18px;
    align-items: start;
    width: 100%;
    height: 100%;
  }

  .tiles {
    display: grid;
    grid-template-columns: 1fr 260px 1fr;
    align-items: center;
    gap: 18px;
    min-height: 240px;
  }

  .tile-stack {
    position: relative;
    width: 100%;
    aspect-ratio: 1 / 1;
    border-radius: 12px;
    overflow: hidden;
    background: #fff;
    border: 1px solid rgba(0,0,0,0.08);
  }

  .tile {
    position: absolute;
    inset: 0;
    width: 100%;
    height: 100%;
    object-fit: cover;
    display: block;
  }

  .tile.top {
    transition: opacity 120ms linear;
  }

  .arrow-wrap {
    display: grid;
    place-items: center;
  }

  .arrow {
    width: 100%;
    height: auto;
  }

  .controls {
    border-radius: 14px;
    background: rgba(255,255,255,0.85);
    border: 1px solid rgba(0,0,0,0.08);
    padding: 14px;
  }

  .controls-title {
    font-size: 12px;
    letter-spacing: 0.08em;
    text-transform: uppercase;
    color: rgba(0,0,0,0.65);
    margin-bottom: 10px;
  }

  .value-row {
    display: flex;
    align-items: baseline;
    gap: 6px;
    margin-bottom: 10px;
  }

  .value {
    font-size: 28px;
    font-weight: 700;
    color: #111;
    font-variant-numeric: tabular-nums;
  }

  .unit {
    color: rgba(0,0,0,0.55);
    font-size: 12px;
  }

  input[type="range"] {
    width: 100%;
  }

  .hint {
    margin-top: 10px;
    font-size: 12px;
    color: rgba(0,0,0,0.55);
    line-height: 1.4;
  }

  @media (max-width: 900px) {
    .tile-concept {
      grid-template-columns: 1fr;
    }
    .tiles {
      grid-template-columns: 1fr;
    }
    .arrow-wrap {
      display: none;
    }
  }
</style>