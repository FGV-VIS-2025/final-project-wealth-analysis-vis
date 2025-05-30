<script>
  export let data = [];
  export let animate = false;
  import { onMount, afterUpdate } from 'svelte';
  import * as d3 from 'd3';
  import { tick } from 'svelte';

  let svgElement;
  const margin = { top: 20, right: 30, bottom: 70, left: 60 };
  let width = 500;
  let height = 300;
  let innerWidth = width - margin.left - margin.right;
  let innerHeight = height - margin.top - margin.bottom;

  let tooltip = { show: false, x: 0, y: 0, label: '', value: 0, percent: 0 };
  let lastDataKey = '';

  async function drawChart() {
    if (!svgElement || data.length === 0) return;

    d3.select(svgElement).selectAll('*').remove();

    const svg = d3.select(svgElement)
      .attr('width', width)
      .attr('height', height)
      .append('g')
      .attr('transform', `translate(${margin.left},${margin.top})`);

    const x = d3.scaleBand()
      .range([0, innerWidth])
      .domain(data.map(d => d.gender))
      .padding(0.2);

    svg.append('g')
      .attr('transform', `translate(0,${innerHeight})`)
      .call(d3.axisBottom(x))
      .selectAll('text')
        .style('text-anchor', 'middle')
        .style('fill', '#e0e0e0');

    const y = d3.scaleLinear()
      .domain([0, d3.max(data, d => d.count) || 10])
      .range([innerHeight, 0]);

    svg.append('g')
      .call(d3.axisLeft(y))
      .selectAll('text')
        .style('fill', '#e0e0e0');

    const total = data.reduce((sum, d) => sum + d.count, 0);

    const bars = svg.selectAll('rect')
      .data(data, d => d.gender);

    const barsEnter = bars.enter().append('rect')
      .attr('x', d => x(d.gender))
      .attr('width', x.bandwidth())
      .attr('fill', d => {
        if (d.gender === 'M') return '#2563eb';
        if (d.gender === 'F') return '#dc2626';
        return '#6b7280';
      });

    barsEnter
      .attr('y', d => y(d.count))
      .attr('height', d => innerHeight - y(d.count));

    // Tooltip sempre instantâneo, independente de animação
    barsEnter
      .on('mousemove', (event, d) => {
        tooltip = {
          show: true,
          x: event.offsetX + 10,
          y: event.offsetY - 10,
          label: d.gender,
          value: d.count,
          percent: ((d.count / total) * 100).toFixed(1)
        };
      })
      .on('mouseleave', () => {
        tooltip = { show: false, x: 0, y: 0, label: '', value: 0, percent: 0 };
      });

    // Só remove transições se não for animação de entrada
    if (!animate) {
      d3.select(svgElement).selectAll('rect')
        .interrupt()
        .style('transition', 'none');
    }

    let newDataKey = data.map(d => d.gender + d.count).join('-');
    if (lastDataKey !== newDataKey) {
      tooltip = { show: false, x: 0, y: 0, label: '', value: 0, percent: 0 };
      lastDataKey = newDataKey;
    }

    svg.append('text')
        .attr('text-anchor', 'middle')
        .attr('x', innerWidth / 2)
        .attr('y', innerHeight + margin.bottom - 20)
        .text('Gender')
        .style('fill', '#e0e0e0');

    svg.append('text')
        .attr('text-anchor', 'middle')
        .attr('transform', 'rotate(-90)')
        .attr('y', -margin.left + 20)
        .attr('x', -innerHeight / 2)
        .text('Number of Billionaires')
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
        width = svgElement.parentElement.clientWidth > 0 ? svgElement.parentElement.clientWidth : 500;
        height = svgElement.parentElement.clientHeight > 0 ? svgElement.parentElement.clientHeight : 300;
        if (width < 200) width = 200; 
        if (height < 150) height = 150;
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
    height: 400px;
    display: flex;
    justify-content: center;
    align-items: center;
    position: relative;
  }
  svg {
    max-width: 100%;
    max-height: 100%;
  }
  svg rect {
    transition: none !important;
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