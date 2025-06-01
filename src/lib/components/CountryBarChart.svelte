<script>
  export let data = []; 
  import { onMount, afterUpdate } from 'svelte';
  import * as d3 from 'd3';

  let svgElement, tooltip, sortBy = 'count', filterRegion = 'all';
  
  const margin = { top: 60, right: 30, bottom: 80, left: 150 }; 
  let width = 700, height = 450;
  let innerWidth = width - margin.left - margin.right;
  let innerHeight = height - margin.top - margin.bottom;

  const regionColors = {
    'North America': '#6366f1',    
    'Europe': '#06b6d4',           
    'Asia': '#f59e0b',             
    'South America': '#10b981',    
    'Africa': '#ef4444',          
    'Oceania': '#8b5cf6',       
    'Middle East': '#f97316',     
    'default': '#6b7280'        
  };

  const countryRegions = {
    'United States': 'North America', 'China': 'Asia', 'Germany': 'Europe', 'India': 'Asia',
    'United Kingdom': 'Europe', 'Russia': 'Europe', 'Hong Kong': 'Asia', 'Brazil': 'South America',
    'Canada': 'North America', 'France': 'Europe', 'Switzerland': 'Europe', 'Italy': 'Europe',
    'South Korea': 'Asia', 'Japan': 'Asia', 'Australia': 'Oceania', 'Singapore': 'Asia',
    'Israel': 'Middle East', 'Turkey': 'Europe', 'Taiwan': 'Asia', 'Thailand': 'Asia'
  };

  $: processedData = data.map(d => ({
    ...d,
    region: countryRegions[d.country] || 'Other',
    color: regionColors[countryRegions[d.country]] || regionColors.default
  }));

  $: filteredData = filterRegion === 'all' ? processedData : processedData.filter(d => d.region === filterRegion);

  $: sortedData = [...filteredData].sort((a, b) => {
    if (sortBy === 'count') return b.count - a.count;
    if (sortBy === 'alphabetical') return a.country.localeCompare(b.country);
    if (sortBy === 'region') return a.region.localeCompare(b.region);
    return 0;
  });

  $: uniqueRegions = [...new Set(processedData.map(d => d.region))].sort();

  function drawChart() {
    if (!svgElement || sortedData.length === 0) return;

    d3.select(svgElement).selectAll('*').remove();

    const svg = d3.select(svgElement).attr('width', width).attr('height', height);

    const defs = svg.append('defs');
    
    Object.entries(regionColors).forEach(([region, color]) => {
      const gradient = defs.append('linearGradient')
        .attr('id', `gradient-${region.replace(/\s+/g, '-')}`)
        .attr('x1', '0%').attr('y1', '0%')
        .attr('x2', '100%').attr('y2', '0%');
      
      gradient.append('stop')
        .attr('offset', '0%')
        .attr('stop-color', color)
        .attr('stop-opacity', 0.8);
      
      gradient.append('stop')
        .attr('offset', '100%')
        .attr('stop-color', color)
        .attr('stop-opacity', 1);
    });

    svg.append('text').attr('x', width / 2).attr('y', 30).attr('text-anchor', 'middle')
      .style('font-size', '18px').style('font-weight', 'bold').style('fill', '#e0e0e0')
      .text('Bilionários por País');

    const chartGroup = svg.append('g').attr('transform', `translate(${margin.left},${margin.top})`);

    const y = d3.scaleBand().range([0, innerHeight]).domain(sortedData.map(d => d.country)).padding(0.15);
    const x = d3.scaleLinear().domain([0, d3.max(sortedData, d => d.count) || 10]).range([0, innerWidth]);

    chartGroup.append('g').call(d3.axisLeft(y)).selectAll('text')
      .style('fill', '#e0e0e0').style('font-size', '12px');

    chartGroup.append('g').attr('transform', `translate(0,${innerHeight})`)
      .call(d3.axisBottom(x).tickFormat(d3.format('d'))).selectAll('text').style('fill', '#e0e0e0');

    const bars = chartGroup.selectAll('rect').data(sortedData).join('rect')
      .attr('y', d => y(d.country)).attr('x', 0).attr('height', y.bandwidth()).attr('width', 0)
      .attr('fill', d => `url(#gradient-${d.region.replace(/\s+/g, '-')})`)
      .attr('stroke', 'rgba(255,255,255,0.2)').attr('stroke-width', 0.5)
      .attr('rx', 4).attr('ry', 4)  // Bordas arredondadas
      .style('cursor', 'pointer');

    bars.transition().duration(800).delay((d, i) => i * 40).attr('width', d => x(d.count));

    const labels = chartGroup.selectAll('.bar-label').data(sortedData).join('text')
      .attr('class', 'bar-label').attr('x', 5).attr('y', d => y(d.country) + y.bandwidth() / 2).attr('dy', '0.35em')
      .style('font-size', '11px').style('font-weight', 'bold').style('opacity', 0);

    labels.transition().duration(800).delay((d, i) => i * 40 + 400).style('opacity', 1)
      .attr('x', d => {
        const barWidth = x(d.count);
        return barWidth < 60 ? barWidth + 8 : barWidth - 8;
      })
      .attr('text-anchor', d => x(d.count) < 60 ? 'start' : 'end')
      .style('fill', d => x(d.count) < 60 ? '#e0e0e0' : '#ffffff')
      .text(d => d.count.toLocaleString());

    bars.on('mouseover', function(event, d) {
        d3.select(this).transition().duration(200)
          .attr('opacity', 0.9)
          .attr('stroke-width', 1)
          .attr('stroke', '#ffffff');
        showTooltip(event, d);
      })
      .on('mouseout', function() {
        d3.select(this).transition().duration(200)
          .attr('opacity', 1)
          .attr('stroke-width', 0.5)
          .attr('stroke', 'rgba(255,255,255,0.2)');
        hideTooltip();
      })
      .on('mousemove', function(event) { updateTooltipPosition(event); });

    chartGroup.append('text').attr('text-anchor', 'middle').attr('x', innerWidth / 2).attr('y', innerHeight + 40)
      .text('Número de Bilionários').style('fill', '#e0e0e0').style('font-size', '14px');
    chartGroup.append('text').attr('text-anchor', 'middle').attr('transform', 'rotate(-90)')
      .attr('y', -100).attr('x', -innerHeight / 2).text('Países').style('fill', '#e0e0e0').style('font-size', '14px');
  }

  function showTooltip(event, d) {
    if (!tooltip) return;
    const percentage = ((d.count / d3.sum(processedData, d => d.count)) * 100).toFixed(1);
    tooltip.style('opacity', 1).html(`
      <div style="background: rgba(0,0,0,0.9); padding: 12px; border-radius: 8px; color: white; font-size: 13px; box-shadow: 0 4px 12px rgba(0,0,0,0.3); border: 1px solid ${d.color};">
        <div style="font-weight: bold; margin-bottom: 8px; color: ${d.color};">${d.country}</div>
        <div><strong>Bilionários:</strong> ${d.count.toLocaleString()}</div>
        <div><strong>Região:</strong> ${d.region}</div>
        <div><strong>% do Total:</strong> ${percentage}%</div>
      </div>`);
    updateTooltipPosition(event);
  }

  function updateTooltipPosition(event) {
    if (!tooltip) return;
    tooltip.style('left', (event.pageX + 10) + 'px').style('top', (event.pageY - 10) + 'px');
  }

  function hideTooltip() {
    if (!tooltip) return;
    tooltip.style('opacity', 0);
  }
  
  function updateDimensions() {
    if (svgElement && svgElement.parentElement) {
      width = svgElement.parentElement.clientWidth > 0 ? svgElement.parentElement.clientWidth : 700;
      height = svgElement.parentElement.clientHeight > 0 ? svgElement.parentElement.clientHeight : 450;
      if (width < 400) width = 400;
      if (height < 300) height = 300; 
      innerWidth = width - margin.left - margin.right;
      innerHeight = height - margin.top - margin.bottom;
      drawChart();
    }
  }

  onMount(() => {
    tooltip = d3.select('body').append('div')
      .style('position', 'absolute').style('opacity', 0).style('pointer-events', 'none').style('z-index', 1000);
    updateDimensions();
    window.addEventListener('resize', updateDimensions);
    return () => {
      window.removeEventListener('resize', updateDimensions);
      if (tooltip) tooltip.remove();
    };
  });

  afterUpdate(() => { drawChart(); });
</script>

<div class="chart-wrapper">
  <div class="controls">
    <div class="control-group">
      <label for="sort-select">Ordenar por:</label>
      <select id="sort-select" bind:value={sortBy}>
        <option value="count">Número de Bilionários</option>
        <option value="alphabetical">Ordem Alfabética</option>
        <option value="region">Região</option>
      </select>
    </div>
    
    <div class="control-group">
      <label for="region-select">Filtrar por Região:</label>
      <select id="region-select" bind:value={filterRegion}>
        <option value="all">Todas as Regiões</option>
        {#each uniqueRegions as region}
          <option value={region}>{region}</option>
        {/each}
      </select>
    </div>
  </div>

  <div class="chart-container" bind:clientWidth={width} bind:clientHeight={height}>
    <svg bind:this={svgElement}></svg>
  </div>

  <div class="legend">
    <h4>Regiões:</h4>
    <div class="legend-items">
      {#each Object.entries(regionColors) as [region, color]}
        {#if region !== 'default' && uniqueRegions.includes(region)}
          <div class="legend-item">
            <div class="legend-color" style="background-color: {color}"></div>
            <span>{region}</span>
          </div>
        {/if}
      {/each}
    </div>
  </div>
</div>

<style>
  .chart-wrapper {
    width: 100%;
    background: rgba(255, 255, 255, 0.05);
    border-radius: 12px;
    padding: 20px;
    backdrop-filter: blur(10px);
    border: 1px solid rgba(255, 255, 255, 0.1);
  }

  .controls { display: flex; gap: 20px; margin-bottom: 20px; flex-wrap: wrap; }
  .control-group { display: flex; flex-direction: column; gap: 5px; }
  .control-group label { color: #e0e0e0; font-size: 12px; font-weight: 500; }
  .control-group select {
    padding: 8px 12px; border-radius: 6px; border: 1px solid rgba(255, 255, 255, 0.2);
    background: rgba(0, 0, 0, 0.3); color: #e0e0e0; font-size: 13px; cursor: pointer;
  }
  .control-group select:focus {
    outline: none; border-color: #4ecdc4; box-shadow: 0 0 0 2px rgba(78, 205, 196, 0.2);
  }

  .chart-container {
    width: 100%; height: 450px; display: flex; justify-content: center; align-items: center;
  }
  svg { max-width: 100%; max-height: 100%; }

  .legend {
    margin-top: 15px; border-top: 1px solid rgba(255, 255, 255, 0.1); padding-top: 15px;
  }
  .legend h4 { color: #e0e0e0; margin: 0 0 10px 0; font-size: 14px; }
  .legend-items { display: flex; flex-wrap: wrap; gap: 15px; }
  .legend-item { display: flex; align-items: center; gap: 6px; font-size: 12px; color: #e0e0e0; }
  .legend-color {
    width: 16px; height: 16px; border-radius: 3px; border: 1px solid rgba(255, 255, 255, 0.2);
  }

  @media (max-width: 768px) {
    .chart-wrapper { padding: 15px; }
    .controls { flex-direction: column; gap: 15px; }
    .chart-container { height: 400px; }
  }
</style> 