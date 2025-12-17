<script>
  import { onMount } from "svelte";
  import * as d3 from "d3";
  import * as XLSX from "xlsx";

  import CropVsChatgptBars from "./lib/CropVsChatgptBars.svelte";
  import DailyComparisonBars from "./lib/DailyComparisonBars.svelte";
  import TotalComparisonBars from "./lib/TotalComparisonBars.svelte";
  import AlfalfaPrototype from "./lib/AlfalfaPrototype.svelte";

  // ---------------------------
  // SETTINGS YOU CONTROL
  // ---------------------------
  const DAILY_PROMPTS = 2_000_000_000;
  const LITRES_PER_PROMPT = 0.00032176;
  const CHATGPT_L_PER_DAY = DAILY_PROMPTS * LITRES_PER_PROMPT;

  const CROP_AREA_HA = {
    alfalfa: 55689.0,
    tomatoes: 20317.0,
    onions: 10318.0,
    wheat: 0
  };

  // ---------------------------
  // TILE CONCEPT (wipe)
  // ---------------------------
  let promptValue = 0;

  const GREEN_TILE = "/tiles-concept/green-tile.svg";
  const BROWN_TILE = "/tiles-concept/patch-tile.svg";

  function onSliderInput(e) {
    promptValue = +e.currentTarget.value;
  }

  $: t = Math.max(0, Math.min(promptValue / 100, 1));
  $: rightInset = (1 - t) * 100;
  $: bottomInset = (1 - t) * 100;
  $: clipStyle = `inset(0% ${rightInset}% ${bottomInset}% 0%)`;

  // ---------------------------
  // TILE GALLERY (4 tiles)
  // ---------------------------
  const tiles = [
    { id: "alf", label: "Alfalfa", img: "/tiles-concept/green-tile-final.svg", daily: "", weekly: "", annual: "" },
    { id: "tom", label: "Tomatoes", img: "/tiles-concept/tomatoes-concept.svg", daily: "", weekly: "", annual: "" },
    { id: "oni", label: "Onions", img: "/tiles-concept/onions.svg", daily: "", weekly: "", annual: "" },
    { id: "whe", label: "Wheat", img: "/tiles-concept/wheat-concept.svg", daily: "", weekly: "", annual: "" }
  ];
  let hovered = null;

  // ---------------------------
  // CSV PREVIEW TABLE
  // ---------------------------
  let previewRows = [];
  let previewCols = [];
  let previewError = "";

  onMount(async () => {
    try {
      const rows = await d3.csv("/dataset/ncro_ai_prompts_crop_equivalence.csv", d3.autoType);
      previewRows = rows.slice(0, 20);
      previewCols = rows.columns ?? (rows.length ? Object.keys(rows[0]) : []);
    } catch (e) {
      console.error(e);
      previewError =
        "Couldn’t load the CSV preview. Make sure the file is at public/dataset/ncro_ai_prompts_crop_equivalence.csv";
    }
  });

  // ---------------------------
  // EXCEL PREPROCESS (AW + AREA)
  // ---------------------------
  const CROPS = [
    { key: "alfalfa", label: "Alfalfa" },
    { key: "tomatoes", label: "Tomatoes" },
    { key: "onions", label: "Onions" },
    { key: "wheat", label: "Wheat" }
  ];

  let cropMetrics = {
    alfalfa: { label: "Alfalfa", lPerHaPerDay: 0, areaHa: 0 },
    tomatoes: { label: "Tomatoes", lPerHaPerDay: 0, areaHa: 0 },
    onions: { label: "Onions", lPerHaPerDay: 0, areaHa: 0 },
    wheat: { label: "Wheat", lPerHaPerDay: 0, areaHa: 0 }
  };

  let excelError = "";

  const ACRE_FT_TO_L = 1233481.84;
  const ACRE_TO_HA = 0.404686;

  function norm(s) {
    return String(s ?? "").trim().toLowerCase();
  }

  function pick(obj, candidates) {
    const keys = Object.keys(obj || {});
    for (const c of candidates) {
      const k = keys.find((kk) => norm(kk) === norm(c));
      if (k != null) return obj[k];
    }
    return undefined;
  }

  onMount(async () => {
    try {
      excelError = "";

      const res = await fetch("/dataset/california-DAUCo-dataset.xlsx");
      if (!res.ok) throw new Error("Failed to fetch XLSX from /public/dataset");

      const buf = await res.arrayBuffer();
      const wb = XLSX.read(buf, { type: "array" });

      const sheetName = wb.SheetNames[0];
      const ws = wb.Sheets[sheetName];

      const rows = XLSX.utils.sheet_to_json(ws, { defval: null });

      const CROP_COLS = ["crop", "Crop", "Crop Name", "CROP"];
      const AW_COLS = ["AW", "aw", "AW (acre-ft/acre)", "AW_acreft_per_acre"];
      const AREA_ACRES_COLS = ["Acres", "acres", "AREA_ACRES", "Area (acres)", "Area_Acres"];
      const AREA_HA_COLS = ["Hectares", "hectares", "AREA_HA", "Area (ha)", "Area_Ha"];

      const CROPNAME_MAP = {
        alfalfa: ["alfalfa"],
        tomatoes: ["tomato", "tomatoes", "tomato_processing", "processing tomatoes"],
        onions: ["onion", "onions", "onions_garlic", "garlic"],
        wheat: ["wheat", "grain", "small grains"]
      };

      const next = { ...cropMetrics };

      for (const { key, label } of CROPS) {
        const aliases = CROPNAME_MAP[key].map(norm);

        const r = rows.find((row) => {
          const cropName = norm(pick(row, CROP_COLS));
          return aliases.some((a) => cropName.includes(a));
        });

        if (!r) continue;

        const aw = Number(pick(r, AW_COLS) ?? 0) || 0;

        const haDirect = Number(pick(r, AREA_HA_COLS) ?? 0) || 0;
        const acres = Number(pick(r, AREA_ACRES_COLS) ?? 0) || 0;
        const areaHa = haDirect > 0 ? haDirect : (acres > 0 ? acres * ACRE_TO_HA : 0);

        const lPerHaPerYear = aw * ACRE_FT_TO_L / ACRE_TO_HA;
        const lPerHaPerDay = lPerHaPerYear / 365;

        next[key] = { label, lPerHaPerDay, areaHa };
      }

      cropMetrics = next;
      console.log("Parsed cropMetrics:", cropMetrics);
    } catch (e) {
      console.error(e);
      excelError =
        "Couldn’t parse california-DAUCo-dataset.xlsx. Check it is in public/dataset and column headers match.";
    }
  });

  // ---------------------------
  // FINAL PROTOTYPE (tile grid)
  // ---------------------------

  const GRID_COLS = 25;
  const GRID_ROWS = 16;
  const GRID_TOTAL = GRID_COLS * GRID_ROWS;

  let protoSvgEl;

  let protoData = [];
  let protoActivities = [];
  let protoUnitsByActivity = {};

  const CROP_OPTIONS = [
    { key: "alfalfa", label: "Alfalfa", tile: "/tiles-concept/green-tile-final.svg" },
    { key: "tomato_processing", label: "Tomatoes", tile: "/tiles-concept/tomatoes-concept.svg" },
    { key: "onions_garlic", label: "Onions", tile: "/tiles-concept/onions.svg" },
    { key: "grain", label: "Wheat", tile: "/tiles-concept/wheat-concept.svg" }
  ];

  // aliases so your dropdown can match whatever the JSON uses
  const CROP_ALIASES = {
    alfalfa: ["alfalfa"],
    tomato_processing: ["tomato_processing", "tomatoes", "tomato"],
    onions_garlic: ["onions_garlic", "onions", "onion", "garlic"],
    grain: ["grain", "wheat", "small grains"]
  };

  let protoSelectedActivity = "";
  let protoSelectedCrop = CROP_OPTIONS[0].key;
  let protoUnitIndex = 0;

  let protoRects;
  let protoCellW = 0;
  let protoCellH = 0;
  const protoW = 1200;
  const protoH = 800;

  const norm2 = (s) => String(s ?? "").trim().toLowerCase();

  $: protoUnitValues = protoUnitsByActivity[protoSelectedActivity] || [];
  $: protoCurrentUnits = protoUnitValues[protoUnitIndex];

  $: protoCurrentTile =
    CROP_OPTIONS.find((c) => c.key === protoSelectedCrop)?.tile ||
    "/tiles-concept/green-tile-final.svg";

  function cropMatches(rowCrop, selectedKey) {
    const rc = norm2(rowCrop);
    const aliases = (CROP_ALIASES[selectedKey] || [selectedKey]).map(norm2);
    return aliases.some((a) => rc === a || rc.includes(a));
  }

  function findCurrentRow() {
    if (!protoData.length) return null;
    const act = protoSelectedActivity;
    const unitsNum = Number(protoCurrentUnits);

    return protoData.find((d) => {
      const okAct = norm2(d.activity_type) === norm2(act);
      const okCrop = cropMatches(d.crop, protoSelectedCrop);
      const okUnits = Number(d.n_units) === unitsNum;
      return okAct && okCrop && okUnits;
    });
  }

  $: protoCurrentRow = findCurrentRow();

  function drawProtoGrid() {
    if (!protoSvgEl) return;

    const svg = d3
      .select(protoSvgEl)
      .attr("viewBox", `0 0 ${protoW} ${protoH}`)
      .attr("width", "100%")
      .attr("height", "100%");

    svg.selectAll("*").remove();

    protoCellW = protoW / GRID_COLS;
    protoCellH = protoH / GRID_ROWS;

    const indices = d3.range(GRID_TOTAL);
    const g = svg.append("g").attr("class", "grid");

    const cell = g
      .selectAll("g.cell")
      .data(indices)
      .enter()
      .append("g")
      .attr("class", "cell");

    // base tile
    cell
      .append("image")
      .attr("class", "tile")
      .attr("href", protoCurrentTile)
      .attr("xlink:href", protoCurrentTile)
      .attr("x", (d) => (d % GRID_COLS) * protoCellW)
      .attr("y", (d) => Math.floor(d / GRID_COLS) * protoCellH)
      .attr("width", protoCellW)
      .attr("height", protoCellH)
      .attr("preserveAspectRatio", "none");

    // dry overlay
    cell
      .append("rect")
      .attr("class", "dry")
      .attr("x", (d) => (d % GRID_COLS) * protoCellW)
      .attr("y", (d) => Math.floor(d / GRID_COLS) * protoCellH)
      .attr("width", protoCellW)
      .attr("height", protoCellH)
      .attr("fill", "transparent");

    protoRects = g.selectAll("rect.dry");
  }

  function updateProtoGrid() {
    if (!protoCurrentRow || !protoRects) return;

    const frac = Math.max(0, Math.min(Number(protoCurrentRow.ha_equiv_day || 0), 1));
    const exact = frac * GRID_TOTAL;
    const full = Math.floor(exact);
    const rem = exact - full;

    protoRects
      .attr("fill", "transparent")
      .attr("y", (d) => Math.floor(d / GRID_COLS) * protoCellH)
      .attr("height", protoCellH);

    protoRects.filter((d) => d < full).attr("fill", "rgba(155,107,67,0.75)");

    if (rem > 0 && full < GRID_TOTAL) {
      protoRects
        .filter((d) => d === full)
        .attr("fill", "rgba(155,107,67,0.75)")
        .attr("y", (d) => Math.floor(d / GRID_COLS) * protoCellH + (1 - rem) * protoCellH)
        .attr("height", rem * protoCellH);
    }
  }

  // load JSON + init defaults
  onMount(async () => {
    try {
      drawProtoGrid();

      // try both locations (your old code used the non-/dataset path)
      const candidates = [
        "/ncro_ai_full_crop_equivalence.json",
        "/dataset/ncro_ai_full_crop_equivalence.json"
      ];

      let loaded = null;
      for (const url of candidates) {
        const r = await fetch(url);
        if (r.ok) {
          loaded = await r.json();
          console.log("Loaded prototype JSON from:", url);
          break;
        }
      }
      if (!loaded) throw new Error("Could not fetch prototype JSON from either location.");

      protoData = loaded;

      // debug helpers
      console.log("Prototype rows:", protoData.length);
      console.log("Unique crops:", [...new Set(protoData.map((d) => d.crop))].slice(0, 50));
      console.log("Unique activities:", [...new Set(protoData.map((d) => d.activity_type))]);

      protoActivities = [...new Set(protoData.map((d) => d.activity_type))];
      protoSelectedActivity = protoActivities[0] ?? "";

      protoUnitsByActivity = {};
      protoActivities.forEach((act) => {
        protoUnitsByActivity[act] = [
          ...new Set(protoData.filter((d) => d.activity_type === act).map((d) => Number(d.n_units)))
        ]
          .filter((x) => Number.isFinite(x))
          .sort((a, b) => a - b);
      });

      protoUnitIndex = Math.floor((protoUnitsByActivity[protoSelectedActivity]?.length || 1) * 0.7);

      // pick first crop that exists (using aliases)
      const firstCropThatExists =
        CROP_OPTIONS.find((c) => protoData.some((d) => cropMatches(d.crop, c.key)))?.key ||
        CROP_OPTIONS[0].key;

      protoSelectedCrop = firstCropThatExists;

      // redraw now that everything is real
      drawProtoGrid();
      updateProtoGrid();
    } catch (e) {
      console.error("Prototype failed to load:", e);
    }
  });

  // When crop changes → redraw (this is what makes tile swapping reliable)
  $: if (protoSvgEl && protoData.length) {
    drawProtoGrid();
    updateProtoGrid();
  }

  // When slider/activity changes → update overlay
  $: if (protoData.length) updateProtoGrid();
</script>

<main class="case-study">
  <section class="hero">
    <h1>Why my farm didn’t dry out</h1>
    <p class="subtitle"></p>
    <p class="mentor">Mentor: Chris Knox</p>
  </section>

  <section class="section">
    <h2>Water usage in data centres</h2>
    <p>
      Water is becoming increasingly scarce as climate change, population growth, and
      rising demand put pressure on rivers and groundwater. At the same time, digital
      infrastructure, including cloud computing and data centres, is quietly emerging as a new water user
      as our online activity continues to grow. This expansion is happening alongside
      agriculture, which already uses the largest share of freshwater and is essential for
      producing food. Both farming and digital infrastructure depend on reliable access to water
      and are often located in the same regions, drawing from shared water supplies. Comparing them
      helps put the water footprint of AI into context and highlights how growing digital services
      fit into an already stretched water system.
    </p>
  </section>

  <section class="section">
    <h2>Vision board and concept</h2>
    <p>I wanted to represent blocks of farms as measuring cups.</p>

    <div class="vision-board">
      <div class="concept-row" aria-label="Tile concept row: green → dry">
        <div class="concept-tileStack" aria-label="Reference tile">
          <img src={GREEN_TILE} alt="Green farm tile" class="concept-tileImg" />
        </div>

        <div class="concept-arrowWrap" aria-hidden="true">
          <svg viewBox="0 0 160 60" class="concept-arrowSvg">
            <line x1="10" y1="30" x2="140" y2="30" stroke="currentColor" stroke-width="5" stroke-linecap="round" />
            <polyline
              points="130,20 150,30 130,40"
              fill="none"
              stroke="currentColor"
              stroke-width="5"
              stroke-linecap="round"
              stroke-linejoin="round"
            />
          </svg>
        </div>

        <div class="concept-tileStack" aria-label="Tile dries with prompts">
          <img src={GREEN_TILE} alt="Green tile base" class="concept-tileImg" style={`opacity:${1 - t};`} />
          <img
            src={BROWN_TILE}
            alt="Dry tile reveal"
            class="concept-tileImg concept-wipe"
            style={`clip-path:${clipStyle}; opacity:${t};`}
          />
        </div>

        <div class="concept-controls">
          <label class="control-label">
            ChatGPT prompts
            <input type="range" min="0" max="100" step="1" value={promptValue} on:input={onSliderInput} />
          </label>

          <div class="prompt-readout">
            <span class="prompt-num">{promptValue}</span>
            <span class="prompt-unit">prompts</span>
          </div>

          <p class="hint">Move the slider → the tile dries.</p>
        </div>
      </div>
    </div>
  </section>

  <section class="section">
    <h2>Learnings and final visualisation</h2>

    <p><strong>Lesson 1:</strong> Don’t fall in love with a visualisation concept before you <u>find</u> the relevant dataset.</p>

    <ul>
      <li>There is currently no openly available, reliable, and well-processed dataset that reports water usage across data centres in a consistent way.</li>
      <li>Estimations had to be derived from secondary sources such as blogs, industry articles, and calculator-based assumptions.</li>
      <li>Agricultural irrigation data and data-centre water usage are reported using fundamentally different units, time scales, and accounting frameworks, making direct comparison non-trivial.</li>
      <li>In the end, I used a California dataset with irrigated crop metrics + a per-prompt water claim to prototype the comparison.</li>
    </ul>

    <div class="data-peek">
      <div class="data-peek__header">
        <div class="data-peek__title">Data peek</div>
        <div class="data-peek__meta">
          First {previewRows.length} rows from <code>ncro_ai_prompts_crop_equivalence.csv</code>
        </div>
      </div>

      {#if previewError}
        <div class="data-peek__error">{previewError}</div>
      {:else if previewRows.length === 0}
        <div class="data-peek__loading">Loading preview…</div>
      {:else}
        <div class="data-peek__tableWrap" role="region" aria-label="CSV preview table">
          <table class="data-peek__table">
            <thead>
              <tr>
                {#each previewCols as col}
                  <th>{col}</th>
                {/each}
              </tr>
            </thead>
            <tbody>
              {#each previewRows as row}
                <tr>
                  {#each previewCols as col}
                    <td>{row[col] ?? ""}</td>
                  {/each}
                </tr>
              {/each}
            </tbody>
          </table>
        </div>
      {/if}
    </div>

    <ul>
      <li>The farm dataset comes from California Land and Water Usage data that has been curated from 2016-2020 .</li>
      <li>California’s farms produce more than 400 different crops and generate over $50 billion in annual revenue, but they also use around 40% of the state’s total water supply, making agriculture by far the largest water user.</li>
      <li>Agricultural irrigation data and data-centre water usage are reported using fundamentally different units, time scales, and accounting frameworks, making direct comparison non-trivial.</li>
      <li>Limiting this to the North coast region, which is water rich relative to central valley.</li>
    </ul>

    <p><strong>Lesson 2:</strong> Don’t fall in love with a visualisation concept before you <u>explore</u> the dataset.</p>

    <ul>
      <li>I assumed daily prompt usage would be larger than farm irrigation water usage.</li>
      <li>Turns out farming uses a LOT of water.</li>
    </ul>

    <p><strong>Lesson 3:</strong> Don’t fall in love with a visualisation concept before you <u>understand</u> the dataset.</p>

    <p>
      Data-centres use water re-looping and irrigation is not as simple as "water used".
      The water sources used are also different — farming doesn't necessarily use fresh water.
    </p>
  </section>

  <section class="section">
    <h2>Tiles & aesthetics</h2>
    <p>Tiles which look like parcel of land that could change state.</p>

    <div class="gallery-grid">
      {#each tiles as tt}
        <figure class="gallery-card" on:mouseenter={() => (hovered = tt.id)} on:mouseleave={() => (hovered = null)}>
          <div
            class="gallery-imgWrap"
            role="button"
            tabindex="0"
            aria-label={`Show details for ${tt.label}`}
            on:focus={() => (hovered = tt.id)}
            on:blur={() => (hovered = null)}
            on:keydown={(e) => {
              if (e.key === "Enter" || e.key === " ") hovered = tt.id;
              if (e.key === "Escape") hovered = null;
            }}
          >
            <img class="gallery-img" src={tt.img} alt={tt.label} />
          </div>

          <div class={"gallery-tip " + (hovered === tt.id ? "show" : "")}>
            <div class="tip-title">{tt.label}</div>
            <div class="tip-row"><span class="k">Daily</span><span class="v">{tt.daily || "—"}</span></div>
            <div class="tip-row"><span class="k">Weekly</span><span class="v">{tt.weekly || "—"}</span></div>
            <div class="tip-row"><span class="k">Annual</span><span class="v">{tt.annual || "—"}</span></div>
          </div>

          <figcaption class="gallery-caption">{tt.label}</figcaption>
        </figure>
      {/each}
    </div>
  </section>

  <section class="section">
    <h2>Final prototype</h2>
    <p>
      The final prototype explored how much of a one-hectare field could be irrigated using
      the same water consumed by AI activities.
    </p>

    <div class="panel">
      <div>
        Activity:
        <select bind:value={protoSelectedActivity}>
          {#each protoActivities as a}
            <option value={a}>{a}</option>
          {/each}
        </select>
      </div>

      <div style="margin-top:8px;">
        Crop:
        <select bind:value={protoSelectedCrop}>
          {#each CROP_OPTIONS as c}
            <option value={c.key}>{c.label}</option>
          {/each}
        </select>
      </div>

      {#if protoUnitValues.length}
        <div style="margin-top:8px;">
          Scale:
          <input type="range" min="0" max={protoUnitValues.length - 1} bind:value={protoUnitIndex} />
        </div>
      {/if}

      <div style="margin-top:8px;">
        {#if protoCurrentRow}
          ha/day: {protoCurrentRow.ha_equiv_day}
        {:else}
          no matching row
        {/if}
      </div>
    </div>

    <svg bind:this={protoSvgEl}></svg>
  </section>
</main>

<style>
  :global(body) {
    margin: 0;
    background: #f5f3f1;
    color: #111;
    font-family: system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial, sans-serif;
  }

  .case-study {
    max-width: 980px;
    margin: 0 auto;
    padding: 56px 20px 96px 20px;
  }

  .hero {
    padding: 28px 0 20px 0;
  }

  h1 {
    margin: 0 0 10px 0;
    font-size: clamp(34px, 4vw, 54px);
    line-height: 1.05;
    letter-spacing: -0.02em;
  }

  .subtitle {
    margin: 0;
    font-size: 16px;
    color: rgba(0, 0, 0, 0.62);
    max-width: 70ch;
    line-height: 1.5;
  }

  .mentor {
    margin: 10px 0 0;
    font-size: 13px;
    color: rgba(0, 0, 0, 0.55);
  }

  .section {
    padding: 28px 0;
    border-top: 1px solid rgba(0, 0, 0, 0.08);
  }

  .section:first-of-type {
    border-top: none;
  }

  h2 {
    margin: 0 0 10px 0;
    font-size: 18px;
    letter-spacing: 0.01em;
  }

  p {
    margin: 0 0 10px 0;
    max-width: 75ch;
    line-height: 1.6;
    color: rgba(0, 0, 0, 0.72);
  }

  ul {
    margin: 10px 0 18px 20px;
    color: rgba(0, 0, 0, 0.72);
    line-height: 1.7;
    max-width: 78ch;
  }

  .vision-board {
    margin-top: 16px;
  }

  :global(:root) {
    --tileSize: 220px;
  }

  .concept-row {
    display: grid;
    grid-template-columns: var(--tileSize) 120px var(--tileSize) minmax(240px, 1fr);
    gap: 14px;
    align-items: center;
    overflow-x: auto;
    padding-bottom: 6px;
  }

  .concept-tileStack {
    position: relative;
    width: var(--tileSize);
    height: var(--tileSize);
    flex: 0 0 auto;
  }

  .concept-tileImg {
    position: absolute;
    inset: 0;
    width: 100%;
    height: 100%;
    display: block;
    object-fit: contain;
    transition: opacity 220ms ease;
  }

  .concept-wipe {
    transition: clip-path 220ms ease-out, opacity 220ms ease;
    will-change: clip-path, opacity;
  }

  .concept-arrowWrap {
    display: grid;
    place-items: center;
    min-width: 120px;
  }

  .concept-arrowSvg {
    width: 110px;
    height: 44px;
    color: rgba(0, 0, 0, 0.6);
  }

  .concept-controls {
    display: flex;
    flex-direction: column;
    gap: 10px;
    min-width: 260px;
  }

  .control-label {
    display: flex;
    flex-direction: column;
    gap: 8px;
    font-size: 13px;
    color: rgba(0, 0, 0, 0.7);
  }

  input[type="range"] {
    width: 100%;
  }

  .prompt-readout {
    display: flex;
    align-items: baseline;
    gap: 8px;
  }

  .prompt-num {
    font-variant-numeric: tabular-nums;
    font-size: 28px;
    font-weight: 650;
    color: #111;
  }

  .prompt-unit {
    color: rgba(0, 0, 0, 0.6);
  }

  .hint {
    margin: 0;
    font-size: 12px;
    color: rgba(0, 0, 0, 0.55);
  }

  .panel {
    position: relative;
    margin-top: 14px;
    background: white;
    padding: 12px;
    border: 1px solid #ddd;
    font-size: 12px;
    border-radius: 12px;
  }

  svg {
    display: block;
    width: 100%;
    height: auto;
    margin-top: 12px;
  }

  .gallery-grid {
    display: grid;
    grid-template-columns: repeat(4, minmax(0, 1fr));
    gap: 16px;
    margin-top: 18px;
  }

  .gallery-card {
    position: relative;
    margin: 0;
    outline: none;
    cursor: pointer;
  }

  .gallery-imgWrap {
    width: 100%;
    aspect-ratio: 1 / 1;
    background: rgba(255, 255, 255, 0.7);
    border: 1px solid rgba(0, 0, 0, 0.10);
    border-radius: 14px;
    overflow: hidden;
    transition: transform 180ms ease, box-shadow 180ms ease;
  }

  .gallery-card:hover .gallery-imgWrap,
  .gallery-card:focus .gallery-imgWrap {
    transform: translateY(-6px) scale(1.03);
    box-shadow: 0 16px 34px rgba(0, 0, 0, 0.12);
    animation: gallery-bounce 420ms cubic-bezier(.2,.9,.2,1);
  }

  @keyframes gallery-bounce {
    0% { transform: translateY(0) scale(1); }
    55% { transform: translateY(-8px) scale(1.04); }
    85% { transform: translateY(-5px) scale(1.02); }
    100% { transform: translateY(-6px) scale(1.03); }
  }

  .gallery-img {
    width: 100%;
    height: 100%;
    display: block;
    object-fit: contain;
  }

  .gallery-caption {
    margin-top: 8px;
    font-size: 12px;
    color: rgba(0, 0, 0, 0.65);
  }

  .gallery-tip {
    position: absolute;
    top: 10px;
    left: calc(100% + 12px);
    width: 220px;
    padding: 10px 12px;
    border-radius: 12px;
    background: rgba(255, 255, 255, 0.98);
    border: 1px solid rgba(0, 0, 0, 0.10);
    box-shadow: 0 18px 50px rgba(0, 0, 0, 0.14);
    opacity: 0;
    transform: translateY(6px) scale(0.98);
    pointer-events: none;
    transition: opacity 120ms ease, transform 120ms ease;
    z-index: 50;
  }

  .gallery-tip.show {
    opacity: 1;
    transform: translateY(0) scale(1);
  }

  .gallery-tip::before {
    content: "";
    position: absolute;
    left: -7px;
    top: 18px;
    width: 12px;
    height: 12px;
    background: rgba(255, 255, 255, 0.98);
    border-left: 1px solid rgba(0, 0, 0, 0.10);
    border-top: 1px solid rgba(0, 0, 0, 0.10);
    transform: rotate(45deg);
  }

  .tip-title { font-weight: 650; margin-bottom: 6px; }

  .tip-row {
    display: flex;
    justify-content: space-between;
    font-size: 12px;
    padding: 2px 0;
  }

  .tip-row .k { color: rgba(0, 0, 0, 0.60); }
  .tip-row .v { font-variant-numeric: tabular-nums; }

  .tip-note {
    margin-top: 6px;
    font-size: 11px;
    color: rgba(0, 0, 0, 0.55);
  }

  .data-peek {
    margin-top: 14px;
    background: rgba(255, 255, 255, 0.72);
    border: 1px solid rgba(0, 0, 0, 0.10);
    border-radius: 14px;
    overflow: hidden;
  }

  .data-peek__header {
    padding: 12px 14px;
    border-bottom: 1px solid rgba(0, 0, 0, 0.08);
    display: grid;
    gap: 4px;
  }

  .data-peek__title { font-weight: 700; font-size: 13px; }

  .data-peek__meta {
    font-size: 12px;
    color: rgba(0, 0, 0, 0.62);
  }

  .data-peek__meta code {
    font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", monospace;
    font-size: 11px;
  }

  .data-peek__loading,
  .data-peek__error {
    padding: 12px 14px;
    font-size: 12px;
    color: rgba(0, 0, 0, 0.65);
  }

  .data-peek__error { color: #8b1d1d; }

  .data-peek__tableWrap {
    max-height: 320px;
    overflow: auto;
  }

  .data-peek__table {
    width: 100%;
    border-collapse: collapse;
    font-size: 12px;
  }

  .data-peek__table thead th {
    position: sticky;
    top: 0;
    background: rgba(245, 243, 241, 0.95);
    backdrop-filter: blur(6px);
    border-bottom: 1px solid rgba(0, 0, 0, 0.10);
    text-align: left;
    padding: 10px 10px;
    white-space: nowrap;
  }

  .data-peek__table tbody td {
    border-bottom: 1px solid rgba(0, 0, 0, 0.06);
    padding: 8px 10px;
    vertical-align: top;
    white-space: nowrap;
  }

  .data-peek__table tbody tr:nth-child(even) {
    background: rgba(255, 255, 255, 0.55);
  }

  @media (max-width: 820px) {
    .gallery-grid { grid-template-columns: repeat(2, minmax(0, 1fr)); }
    .gallery-tip { left: 0; top: calc(100% + 10px); }
    .gallery-tip::before { left: 16px; top: -7px; }
  }

  @media (max-width: 720px) {
    .concept-row {
      grid-template-columns: var(--tileSize) 96px var(--tileSize) minmax(220px, 1fr);
    }
    .concept-arrowSvg { width: 84px; height: 36px; }
  }
</style>