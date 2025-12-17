<script>
  import { onMount } from "svelte";
  import * as d3 from "d3";

  export let title = "";
  export let chatgptLitresPerDay = 0;

  // mode: "perHa" or "total"
  export let mode = "perHa";

  // crop metrics passed in from App
  // crop = { label, lPerHaPerDay, areaHa }
  export let crop = { label: "Crop", lPerHaPerDay: 0, areaHa: 0 };

  let el;
  let err = "";

  function fmtL(x) {
    const v = Number(x);
    if (!Number.isFinite(v)) return "—";
    if (v >= 1e12) return `${(v / 1e12).toFixed(2)}T L`;
    if (v >= 1e9) return `${(v / 1e9).toFixed(2)}B L`;
    if (v >= 1e6) return `${(v / 1e6).toFixed(2)}M L`;
    if (v >= 1e3) return `${(v / 1e3).toFixed(2)}k L`;
    return `${Math.round(v)} L`;
  }

  function clear() {
    if (el) el.innerHTML = "";
  }

  function render() {
    try {
      err = "";
      clear();

      const cropValue =
        mode === "total"
          ? (Number(crop.lPerHaPerDay || 0) * Number(crop.areaHa || 0))
          : Number(crop.lPerHaPerDay || 0);

      const data = [
        { key: "chatgpt", label: "ChatGPT (daily)", value: Number(chatgptLitresPerDay || 0) },
        {
          key: "crop",
          label: mode === "total" ? `${crop.label} (TOTAL daily)` : `${crop.label} (per ha/day)`,
          value: Number(cropValue || 0)
        }
      ];

      // If crop value is 0, still render (you’ll notice missing parsing quickly)
      const W = 520;
      const H = 240;
      const m = { top: 36, right: 14, bottom: 44, left: 66 };
      const innerW = W - m.left - m.right;
      const innerH = H - m.top - m.bottom;

      const svg = d3.select(el).append("svg").attr("viewBox", `0 0 ${W} ${H}`);

      svg.append("text")
        .attr("x", m.left)
        .attr("y", 18)
        .attr("font-size", 12)
        .attr("font-weight", 650)
        .attr("fill", "rgba(0,0,0,0.78)")
        .text(title);

      const maxV = Math.max(1, d3.max(data, (d) => d.value) || 1);

      // Log scale makes “AI becomes invisible” very clear on total plots
      const y = d3.scaleLog()
        .domain([1, maxV])
        .range([m.top + innerH, m.top]);

      const x = d3.scaleBand()
        .domain(data.map((d) => d.key))
        .range([m.left, m.left + innerW])
        .padding(0.35);

      svg.append("g")
        .attr("transform", `translate(${m.left},0)`)
        .call(d3.axisLeft(y).ticks(4, "~s"))
        .call((g) => g.selectAll("text").attr("fill", "rgba(0,0,0,0.6)"))
        .call((g) => g.selectAll("path,line").attr("stroke", "rgba(0,0,0,0.15)"));

      const bars = svg.append("g");

      bars.selectAll("rect")
        .data(data)
        .enter()
        .append("rect")
        .attr("x", (d) => x(d.key))
        .attr("width", x.bandwidth())
        .attr("y", (d) => y(Math.max(1, d.value)))
        .attr("height", (d) => (m.top + innerH) - y(Math.max(1, d.value)))
        .attr("rx", 12)
        .attr("fill", "rgba(0,0,0,0.18)");

      bars.selectAll("text.val")
        .data(data)
        .enter()
        .append("text")
        .attr("x", (d) => (x(d.key) ?? 0) + x.bandwidth() / 2)
        .attr("y", (d) => y(Math.max(1, d.value)) - 8)
        .attr("text-anchor", "middle")
        .attr("font-size", 12)
        .attr("fill", "rgba(0,0,0,0.72)")
        .text((d) => fmtL(d.value));

      const xLabels = { chatgpt: "ChatGPT", crop: mode === "total" ? "Crop total" : "Crop /ha" };

      bars.selectAll("text.lab")
        .data(data)
        .enter()
        .append("text")
        .attr("x", (d) => (x(d.key) ?? 0) + x.bandwidth() / 2)
        .attr("y", m.top + innerH + 26)
        .attr("text-anchor", "middle")
        .attr("font-size", 11)
        .attr("fill", "rgba(0,0,0,0.55)")
        .text((d) => xLabels[d.key]);
    } catch (e) {
      console.error(e);
      err = "Chart render failed.";
    }
  }

  onMount(render);
  $: (title, chatgptLitresPerDay, mode, crop), render();
</script>

<div class="card">
  {#if err}
    <div class="err">{err}</div>
  {:else}
    <div bind:this={el}></div>
  {/if}
</div>

<style>
  .card{
    background: rgba(255,255,255,0.72);
    border: 1px solid rgba(0,0,0,0.10);
    border-radius: 14px;
    padding: 10px;
    overflow: hidden;
  }
  .err{
    font-size: 12px;
    color: #8b1d1d;
    padding: 8px 10px;
  }
</style>