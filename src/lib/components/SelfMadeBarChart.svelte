<script>
  export let data = [];
  import { onMount, afterUpdate } from 'svelte';
  import * as d3 from 'd3';

  let svgElement;
  const margin = { top: 20, right: 30, bottom: 70, left: 60 };
  let width = 500;
  let height = 300;
  let innerWidth = width - margin.left - margin.right;
  let innerHeight = height - margin.top - margin.bottom;

  let tooltip = { show: false, x: 0, y: 0, label: '', value: 0, percent: 0 };

  function drawChart() {
    if (!svgElement || data.length === 0) return;
    d3.select(svgElement).selectAll('*').remove();

    const svg = d3.select(svgElement)
      .attr('width', width)
      .attr('height', height);

    const defs = svg.append('defs');
    
    const selfMadeGradient = defs.append('linearGradient')
      .attr('id', 'gradient-selfmade')
      .attr('x1', '0%').attr('y1', '0%')
      .attr('x2', '100%').attr('y2', '0%');
    
    selfMadeGradient.append('stop')
      .attr('offset', '0%')
      .attr('stop-color', '#10b981')
      .attr('stop-opacity', 0.8);
    
    selfMadeGradient.append('stop')
      .attr('offset', '100%')
      .attr('stop-color', '#10b981')
      .attr('stop-opacity', 1);

    const notSelfMadeGradient = defs.append('linearGradient')
      .attr('id', 'gradient-notselfmade')
      .attr('x1', '0%').attr('y1', '0%')
      .attr('x2', '100%').attr('y2', '0%');
    
    notSelfMadeGradient.append('stop')
      .attr('offset', '0%')
      .attr('stop-color', '#8b5cf6')
      .attr('stop-opacity', 0.8);
    
    notSelfMadeGradient.append('stop')
      .attr('offset', '100%')
      .attr('stop-color', '#8b5cf6')
      .attr('stop-opacity', 1);

    const chartGroup = svg.append('g')
      .attr('transform', `translate(${margin.left},${margin.top})`);

    const x = d3.scaleBand()
      .range([0, innerWidth])
      .domain(data.map(d => d.label))
      .padding(0.3);

    chartGroup.append('g')
      .attr('transform', `translate(0,${innerHeight})`)
      .call(d3.axisBottom(x))
      .selectAll('text')
        .style('text-anchor', 'middle')
        .style('fill', '#e0e0e0')
        .style('font-size', '12px');

    const y = d3.scaleLinear()
      .domain([0, d3.max(data, d => d.value) || 10])
      .range([innerHeight, 0]);

    chartGroup.append('g')
      .call(d3.axisLeft(y))
      .selectAll('text')
        .style('fill', '#e0e0e0')
        .style('font-size', '12px');

    const total = data.reduce((sum, d) => sum + d.value, 0);

    const bars = chartGroup.selectAll('rect')
      .data(data)
      .join('rect')
      .attr('x', d => x(d.label))
      .attr('y', d => y(d.value))
      .attr('width', x.bandwidth())
      .attr('height', d => innerHeight - y(d.value))
      .attr('fill', d => {
        if (d.label === 'Self-Made') return 'url(#gradient-selfmade)';
        if (d.label === 'Não Self-Made') return 'url(#gradient-notselfmade)';
        return '#6b7280';
      })
      .attr('stroke', 'rgba(255,255,255,0.2)')
      .attr('stroke-width', 0.5)
      .attr('rx', 6).attr('ry', 6)  // Bordas arredondadas
      .style('cursor', 'pointer')
      .on('mouseover', function(event, d) {
        d3.select(this).transition().duration(200)
          .attr('opacity', 0.9)
          .attr('stroke-width', 1)
          .attr('stroke', '#ffffff');
        tooltip = {
          show: true,
          x: event.offsetX + 10,
          y: event.offsetY - 10,
          label: d.label,
          value: d.value,
          percent: ((d.value / total) * 100).toFixed(1)
        };
      })
      .on('mouseout', function() {
        d3.select(this).transition().duration(200)
          .attr('opacity', 1)
          .attr('stroke-width', 0.5)
          .attr('stroke', 'rgba(255,255,255,0.2)');
        tooltip = { show: false, x: 0, y: 0, label: '', value: 0, percent: 0 };
      });

    chartGroup.selectAll('.bar-label')
      .data(data)
      .join('text')
      .attr('class', 'bar-label')
      .attr('x', d => x(d.label) + x.bandwidth() / 2)
      .attr('y', d => y(d.value) - 8)
      .attr('text-anchor', 'middle')
      .style('font-size', '12px')
      .style('font-weight', 'bold')
      .style('fill', '#e0e0e0')
      .text(d => d.value.toLocaleString());

    chartGroup.append('text')
      .attr('text-anchor', 'middle')
      .attr('x', innerWidth / 2)
      .attr('y', innerHeight + margin.bottom - 10)
      .text('Origem da Fortuna')
      .style('fill', '#e0e0e0')
      .style('font-size', '14px');

    chartGroup.append('text')
      .attr('text-anchor', 'middle')
      .attr('transform', 'rotate(-90)')
      .attr('y', -margin.left + 20)
      .attr('x', -innerHeight / 2)
      .text('Número de Bilionários')
      .style('fill', '#e0e0e0')
      .style('font-size', '14px');
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
    background: rgba(255, 255, 255, 0.05);
    border-radius: 12px;
    padding: 20px;
    backdrop-filter: blur(10px);
    border: 1px solid rgba(255, 255, 255, 0.1);
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
    background: rgba(0, 0, 0, 0.95);
    color: #e0e0e0;
    padding: 12px 16px;
    border-radius: 8px;
    font-size: 13px;
    font-weight: 500;
    pointer-events: none;
    z-index: 10;
    border: 1px solid rgba(255, 255, 255, 0.3);
    box-shadow: 0 4px 12px rgba(0,0,0,0.3);
    left: 0;
    top: 0;
    transform: translate(-50%, -100%);
    white-space: nowrap;
  }
  
  .tooltip span {
    color: #ffd700;
  }
</style> 