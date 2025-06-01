<script>
  export let data = []; 
  import { onMount, afterUpdate } from 'svelte';
  import * as d3 from 'd3';

  let svgElement;
  const margin = { top: 20, right: 30, bottom: 100, left: 150 }; 
  let width = 600;
  let height = 400;
  let innerWidth = width - margin.left - margin.right;
  let innerHeight = height - margin.top - margin.bottom;

  let tooltip = { show: false, x: 0, y: 0, label: '', value: 0, percent: 0 };

  function drawChart() {
    if (!svgElement || data.length === 0) return;

    d3.select(svgElement).selectAll('*').remove();

    const svg = d3.select(svgElement)
      .attr('width', width)
      .attr('height', height)
      .append('g')
      .attr('transform', `translate(${margin.left},${margin.top})`);

    const y = d3.scaleBand()
      .range([0, innerHeight])
      .domain(data.map(d => d.country))
      .padding(0.1);

    svg.append('g')
      .call(d3.axisLeft(y))
      .selectAll('text')
        .style('fill', '#e0e0e0');

    const x = d3.scaleLinear()
      .domain([0, d3.max(data, d => d.count) || 10])
      .range([0, innerWidth]);

    svg.append('g')
      .attr('transform', `translate(0,${innerHeight})`)
      .call(d3.axisBottom(x))
      .selectAll('text')
        .attr('transform', 'translate(-10,0)rotate(-45)')
        .style('text-anchor', 'end')
        .style('fill', '#e0e0e0');

    const total = data.reduce((sum, d) => sum + d.count, 0);

    svg.selectAll('rect')
      .data(data)
      .join('rect')
      .attr('y', d => y(d.country))
      .attr('x', 0) 
      .attr('height', y.bandwidth())
      .attr('width', d => x(d.count))
      .attr('fill', '#ffd700')
      .on('mousemove', (event, d) => {
        tooltip = {
          show: true,
          x: event.offsetX + 10,
          y: event.offsetY - 10,
          label: d.country,
          value: d.count,
          percent: ((d.count / total) * 100).toFixed(1)
        };
      })
      .on('mouseleave', () => {
        tooltip = { show: false, x: 0, y: 0, label: '', value: 0, percent: 0 };
      });

    svg.append('text')
        .attr('text-anchor', 'middle')
        .attr('x', innerWidth / 2)
        .attr('y', innerHeight + margin.bottom - 30)
        .text('Número de Bilionários')
        .style('fill', '#e0e0e0');

    svg.append('text')
        .attr('text-anchor', 'middle')
        .attr('transform', 'rotate(-90)')
        .attr('y', -margin.left + 50)
        .attr('x', -innerHeight / 2)
        .text('País')
        .style('fill', '#e0e0e0');
  }
  
  onMount(() => {
    drawChart();
  });

  afterUpdate(() => {
    drawChart();
  });

  function updateDimensions() {
    if (svgElement && svgElement.parentElement) {
        width = svgElement.parentElement.clientWidth > 0 ? svgElement.parentElement.clientWidth : 600;
        height = svgElement.parentElement.clientHeight > 0 ? svgElement.parentElement.clientHeight : 400;
        if (width < 250) width = 250;
        if (height < 200) height = 200; 
        innerWidth = width - margin.left - margin.right;
        innerHeight = height - margin.top - margin.bottom;
        drawChart();
    }
  }

  onMount(() => {
    updateDimensions();
    window.addEventListener('resize', updateDimensions);
    return () => {
        window.removeEventListener('resize', updateDimensions);
    };
  });

</script>

<div class="chart-container" bind:clientWidth={width} bind:clientHeight={height} style="position:relative;">
    <svg bind:this={svgElement}></svg>
    {#if tooltip.show}
      <div class="tooltip" style="left: {tooltip.x}px; top: {tooltip.y}px;">
        <span><b>{tooltip.label}</b></span><br>
        <span>{tooltip.value} ({tooltip.percent}%)</span>
      </div>
    {/if}
</div>

<style>
  .chart-container {
    width: 100%;
    height: 450px; 
    display: flex;
    justify-content: center;
    align-items: center;
    position: relative;
  }
  svg {
    max-width: 100%;
    max-height: 100%;
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