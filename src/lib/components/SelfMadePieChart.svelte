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
      .style('opacity', 0.9);

    // Labels
    svg.selectAll('text')
      .data(data_ready)
      .join('text')
      .text(d => `${d.data.label}: ${d.data.value}`)
      .attr('transform', d => `translate(${arc.centroid(d)})`)
      .style('text-anchor', 'middle')
      .style('fill', '#e0e0e0')
      .style('font-size', '1.1em')
      .style('font-weight', 'bold');
  }

  onMount(drawChart);
  afterUpdate(drawChart);
</script>

<div class="pie-chart-container">
  <svg bind:this={svgElement}></svg>
</div>

<style>
.pie-chart-container {
  width: 100%;
  height: 350px;
  display: flex;
  justify-content: center;
  align-items: center;
}
svg {
  max-width: 100%;
  max-height: 100%;
}
</style> 