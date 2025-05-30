<script>
  export let data = [];
  import { onMount, afterUpdate } from 'svelte';
  import * as d3 from 'd3';

  let svgElement;
  let width = 350;
  let height = 350;
  let radius = Math.min(width, height) / 2 - 20;
  const colors = d3.scaleOrdinal()
    .domain(["Self-Made", "Não Self-Made"])
    .range(["#22c55e", "#f59e42"]);

  let hovered = null;
  let tooltip = { show: false, x: 0, y: 0, label: '', value: 0, percent: 0 };

  function drawChart() {
    if (!svgElement || data.length === 0) return;
    d3.select(svgElement).selectAll('*').remove();

    const svg = d3.select(svgElement)
      .attr('width', width)
      .attr('height', height)
      .append('g')
      .attr('transform', `translate(${width / 2},${height / 2})`);

    const pie = d3.pie().value(d => d.value);
    const data_ready = pie(data);

    const arc = d3.arc()
      .innerRadius(0)
      .outerRadius(radius);

    svg.selectAll('path')
      .data(data_ready)
      .join('path')
      .attr('d', arc)
      .attr('fill', d => colors(d.data.label))
      .attr('stroke', '#222')
      .attr('stroke-width', 2)
      .style('opacity', 0.9)
      .on('mousemove', (event, d) => {
        const total = data.reduce((sum, v) => sum + v.value, 0);
        tooltip = {
          show: true,
          x: event.offsetX + 10,
          y: event.offsetY - 10,
          label: d.data.label,
          value: d.data.value,
          percent: ((d.data.value / total) * 100).toFixed(1)
        };
        hovered = d.data.label;
      })
      .on('mouseleave', () => {
        tooltip = { show: false, x: 0, y: 0, label: '', value: 0, percent: 0 };
        hovered = null;
      });
  }

  onMount(drawChart);
  afterUpdate(drawChart);
</script>

<div class="pie-flex-row">
  <div class="pie-chart-container">
    <svg bind:this={svgElement} style="position: relative;"></svg>
    {#if tooltip.show}
      <div class="tooltip" style="left: {tooltip.x}px; top: {tooltip.y}px;">
        <span><b>{tooltip.label}</b></span><br>
        <span>{tooltip.value} ({tooltip.percent}%)</span>
      </div>
    {/if}
  </div>
  <div class="legend-container">
    {#each data as d}
      <div class="legend-item">
        <span class="legend-color" style="background: {colors(d.label)}"></span>
        <span class="legend-label">{d.label}</span>
      </div>
    {/each}
  </div>
</div>

<style>
.pie-flex-row {
  display: flex;
  flex-direction: row;
  align-items: center;
  justify-content: center;
  width: 100%;
}
.pie-chart-container {
  width: 350px;
  height: 350px;
  display: flex;
  justify-content: center;
  align-items: center;
  position: relative;
}
svg {
  max-width: 100%;
  max-height: 100%;
}
.legend-container {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: 18px;
  margin-top: 0;
  margin-left: 32px;
}
.legend-item {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 1.1em;
  color: #e0e0e0;
}
.legend-color {
  width: 20px;
  height: 20px;
  border-radius: 4px;
  display: inline-block;
  border: 2px solid #222;
}
.tooltip {
  position: absolute;
  background: #222;
  color: #ffd700;
  padding: 10px 16px;
  border-radius: 8px;
  font-size: 1em;
  font-weight: 500;
  pointer-events: none;
  z-index: 10;
  border: 1px solid #ffd700;
  box-shadow: 0 4px 8px rgba(0,0,0,0.3);
  left: 0;
  top: 0;
  transform: translate(-50%, -100%);
  white-space: nowrap;
}
</style> 