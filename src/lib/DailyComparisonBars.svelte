<script>
  import { onMount } from "svelte";
  import * as d3 from "d3";

  export let crop; // e.g. "alfalfa", "tomato_processing", "onions_garlic", "grain"
  export let title = "";
  export let dailyPrompts = 2_000_000_000;
  export let dataCentreLitresPerDay = 0;

  let el;
  let err = "";

  function fmtL(x) {
    if (x == null || Number.isNaN(x)) return "—";
    if (x >= 1e9) return `${(x / 1e9).toFixed(2)}B L`;
    if (x >= 1e6) return `${(x / 1e6).toFixed(2)}M L`;
    if (x >= 1e3) return `${(x / 1e3).toFixed(2)}k L`;
    return `${Math.round(x)} L`;
  }

  onMount(async () => {
    try {
      const rows = await d3.csv("/dataset/ncro_ai_prompts_crop_equivalence.csv", d3.autoType);

      // crop per-ha litres/day
      const cropRow = rows.find((r) => r.crop === crop);
      const cropLitresPerDay = cropRow?.l_per_ha_per_day ?? 0;

      // litres per prompt from CSV (constant within scenario)
      const base = rows.find((r) => r.n_prompts && r.litres_used);
      const litresPerPrompt = base ? base.litres_used / base.n_prompts : 0;
      const chatgptLitresPerDay = litresPerPrompt * dailyPrompts;

      const data = [
        { label: "ChatGPT (prompts/day)", value: chatgptLitresPerDay },
        { label: "Data centre (est.)", value: dataCentreLitresPerDay },
        { label: `${crop} (per ha/day)`, value: cropLitresPerDay }
      ];

      const W = 520, H = 260, m = { top: 36, right: 16, bottom: 40, left: 56 };
      const innerW = W - m.left - m.right;
      const innerH = H - m.top - m.bottom;

      const svg = d3.select(el).append("svg").attr("viewBox", `0 0 ${W} ${H}`);

      svg.append("text")
        .attr("x", m.left)
        .attr("y", 22)
        .attr("font-size", 12)
        .attr("font-weight", 650)
        .attr("fill", "rgba(0,0,0,0.75)")
        .text(title || crop);

      // Log scale so "AI looks invisible" is still readable.
      const y = d3.scaleLog()
        .domain([1, Math.max(1, d3.max(data, (d) => d.value) || 1)])
        .range([m.top + innerH, m.top]);

      const x = d3.scaleBand()
        .domain(data.map((d) => d.label))
        .range([m.left, m.left + innerW])
        .padding(0.25);

      svg.append("g")
        .attr("transform", `translate(${m.left},0)`)
        .call(d3.axisLeft(y).ticks(4, "~s"))
        .call((g) => g.selectAll("text").attr("fill", "rgba(0,0,0,0.6)"))
        .call((g) => g.selectAll("path,line").attr("stroke", "rgba(0,0,0,0.15)"));

      svg.append("g")
        .attr("transform", `translate(0,${m.top + innerH})`)
        .call(d3.axisBottom(x).tickFormat(() => "")) // hide long labels
        .call((g) => g.selectAll("path,line").attr("stroke", "rgba(0,0,0,0.15)"));

      const bars = svg.append("g");

      bars.selectAll("rect")
        .data(data)
        .enter()
        .append("rect")
        .attr("x", (d) => x(d.label))
        .attr("width", x.bandwidth())
        .attr("y", (d) => y(Math.max(1, d.value)))
        .attr("height", (d) => (m.top + innerH) - y(Math.max(1, d.value)))
        .attr("rx", 10)
        .attr("fill", "rgba(0,0,0,0.18)");

      bars.selectAll("text.val")
        .data(data)
        .enter()
        .append("text")
        .attr("class", "val")
        .attr("x", (d) => (x(d.label) ?? 0) + x.bandwidth() / 2)
        .attr("y", (d) => y(Math.max(1, d.value)) - 8)
        .attr("text-anchor", "middle")
        .attr("font-size", 11)
        .attr("fill", "rgba(0,0,0,0.70)")
        .text((d) => fmtL(d.value));

      bars.selectAll("text.lab")
        .data(data)
        .enter()
        .append("text")
        .attr("class", "lab")
        .attr("x", (d) => (x(d.label) ?? 0) + x.bandwidth() / 2)
        .attr("y", m.top + innerH + 18)
        .attr("text-anchor", "middle")
        .attr("font-size", 10)
        .attr("fill", "rgba(0,0,0,0.55)")
        .text((d) => {
          if (d.label.startsWith("ChatGPT")) return "ChatGPT";
          if (d.label.startsWith("Data centre")) return "Data centre";
          return "Crop";
        });

    } catch (e) {
      console.error(e);
      err = "Chart failed to load CSV. Check /public/dataset path.";
    }
  });
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