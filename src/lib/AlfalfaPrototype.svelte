<script>
  import { onMount } from "svelte";
  import * as d3 from "d3";

  // -------- config --------
  export let jsonPath = "/datasets/ncro_ai_prompts_crop_equivalence.json"; // public/datasets/...
  export let cropKey = "alfalfa";
  export let tileHref = "/tiles-concept/green-tile-final.svg"; // swap to /basic-green-tile-1.png if you want

  const COLS = 25;
  const ROWS = 16;
  const TOTAL = COLS * ROWS;
  const width = 1200;
  const height = 800;

  let svgEl;

  // -------- data --------
  let data = [];
  let scenarios = [];
  let promptsByScenario = {};

  // -------- UI --------
  let selectedScenario = "";
  let promptIndex = 0;

  // -------- grid state --------
  let rects;
  let cellW = 0;
  let cellH = 0;

  // derived
  $: promptValues = promptsByScenario[selectedScenario] || [];
  $: currentPrompts = promptValues[promptIndex];

  $: currentRow =
    data.find(
      (d) =>
        d.scenario_label === selectedScenario &&
        d.crop === cropKey &&
        d.n_prompts === currentPrompts
    );

  function drawGrid() {
    const svg = d3
      .select(svgEl)
      .attr("viewBox", `0 0 ${width} ${height}`)
      .attr("width", "100%")
      .attr("height", "100%");

    svg.selectAll("*").remove();

    cellW = width / COLS;
    cellH = height / ROWS;

    const indices = d3.range(TOTAL);
    const g = svg.append("g").attr("class", "grid");

    const cell = g
      .selectAll("g.cell")
      .data(indices)
      .enter()
      .append("g")
      .attr("class", "cell");

    cell
      .append("image")
      .attr("class", "tile")
      .attr("href", tileHref)
      .attr("xlink:href", tileHref)
      .attr("x", (d) => (d % COLS) * cellW)
      .attr("y", (d) => Math.floor(d / COLS) * cellH)
      .attr("width", cellW)
      .attr("height", cellH)
      .attr("preserveAspectRatio", "none");

    cell
      .append("rect")
      .attr("class", "dry")
      .attr("x", (d) => (d % COLS) * cellW)
      .attr("y", (d) => Math.floor(d / COLS) * cellH)
      .attr("width", cellW)
      .attr("height", cellH)
      .attr("fill", "transparent");

    rects = g.selectAll("rect.dry");
  }

  function updateGrid() {
    if (!rects) return;

    // clear if no match
    if (!currentRow) {
      rects
        .attr("fill", "transparent")
        .attr("y", (d) => Math.floor(d / COLS) * cellH)
        .attr("height", cellH);
      return;
    }

    const frac = Math.max(0, Math.min(Number(currentRow.ha_equiv_day || 0), 1));
    const exact = frac * TOTAL;
    const full = Math.floor(exact);
    const rem = exact - full;

    rects
      .attr("fill", "transparent")
      .attr("y", (d) => Math.floor(d / COLS) * cellH)
      .attr("height", cellH);

    rects.filter((d) => d < full).attr("fill", "rgba(155,107,67,0.75)");

    if (rem > 0 && full < TOTAL) {
      rects
        .filter((d) => d === full)
        .attr("fill", "rgba(155,107,67,0.75)")
        .attr("y", (d) => Math.floor(d / COLS) * cellH + (1 - rem) * cellH)
        .attr("height", rem * cellH);
    }
  }

  function resetSlider() {
    promptIndex = 0;
  }

  onMount(async () => {
    drawGrid();

    const res = await fetch(jsonPath);
    if (!res.ok) throw new Error(`Failed to fetch ${jsonPath} (${res.status})`);

    data = await res.json();

    // scenario labels (use cases)
    scenarios = [...new Set(data.map((d) => d.scenario_label))].filter(Boolean);

    // prompt values per scenario
    promptsByScenario = {};
    scenarios.forEach((s) => {
      promptsByScenario[s] = [
        ...new Set(data.filter((d) => d.scenario_label === s).map((d) => d.n_prompts))
      ]
        .filter((x) => x != null)
        .sort((a, b) => a - b);
    });

    selectedScenario = scenarios[0] ?? "";
    resetSlider();
    updateGrid();

    // super useful 30-min debug
    console.log("Loaded rows:", data.length);
    console.log("Scenarios:", scenarios);
    console.log("First scenario prompts:", promptsByScenario[selectedScenario]);
  });

  // reactive updates: whenever scenario or slider changes, recolor
  $: if (selectedScenario) {
    // ensure slider index stays valid for new scenario
    if (promptIndex > (promptValues.length - 1)) promptIndex = 0;
    updateGrid();
  }
  $: if (data.length) updateGrid();
</script>

<div class="proto-wrap">
  <div class="panel">
    <div class="row">
      <div class="label">Use case</div>
      <select bind:value={selectedScenario} on:change={resetSlider}>
        {#each scenarios as s}
          <option value={s}>{s}</option>
        {/each}
      </select>
    </div>

    {#if promptValues.length}
      <div class="row" style="margin-top:10px;">
        <div class="label">Prompts</div>
        <input
          type="range"
          min="0"
          max={promptValues.length - 1}
          bind:value={promptIndex}
        />
        <div class="note">{currentPrompts} prompts</div>
      </div>
    {:else}
      <div class="note" style="margin-top:10px;">No prompt values found.</div>
    {/if}

    <div class="row" style="margin-top:10px;">
      {#if currentRow}
        <div class="note">ha/day: {currentRow.ha_equiv_day}</div>
      {:else}
        <div class="note">No matching row for alfalfa + this use case + this prompt value.</div>
      {/if}
    </div>
  </div>

  <svg bind:this={svgEl}></svg>
</div>

<style>
  .proto-wrap { margin-top: 14px; }

  .panel {
    background: white;
    padding: 12px;
    border: 1px solid #ddd;
    font-size: 12px;
    border-radius: 12px;
    max-width: 520px;
  }

  .row { display: grid; gap: 6px; }
  .label { font-weight: 650; color: rgba(0,0,0,0.75); }
  .note { font-size: 11px; color: rgba(0,0,0,0.6); }

  svg {
    display: block;
    width: 100%;
    height: auto;
    margin-top: 12px;
  }
</style>