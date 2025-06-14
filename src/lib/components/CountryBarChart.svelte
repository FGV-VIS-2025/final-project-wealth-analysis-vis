<script>
  export let data = []; 
  export let allData = []; // Dados completos para calcular riqueza total
  import { onMount } from 'svelte';
  import * as d3 from 'd3';

  let svgElement, tooltip, filterRegion = 'all', filterIndustry = 'all';
  
  const margin = { top: 40, right: 20, bottom: 60, left: 120 }; 
  let width = 800, height = 400; // Reduzido para caber melhor
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

  // Extrair indústrias dos dados
  $: uniqueIndustries = allData && allData.length > 0 
    ? ['all', ...new Set(allData.map(d => d.industries).filter(Boolean)).values()].sort()
    : ['all'];

  const maxVisibleIndustries = 8;
  let showAllIndustries = false;
  $: displayedIndustries = uniqueIndustries.slice(1); // remove "all"
  $: visibleIndustries = showAllIndustries ? displayedIndustries : displayedIndustries.slice(0, maxVisibleIndustries);
  $: hiddenCount = displayedIndustries.length - visibleIndustries.length;

  $: processedData = data.map(d => {
    // Calcular riqueza total para este país, filtrada por indústria se selecionada
    let countryWealth = 0;
    if (allData && allData.length > 0) {
      let countryPeople = allData.filter(person => person.country === d.country);
      if (filterIndustry !== 'all') {
        countryPeople = countryPeople.filter(person => person.industries === filterIndustry);
      }
      countryWealth = d3.sum(countryPeople, person => +person.finalWorth || 0);
    }
    
    // Recalcular contagem por país considerando filtro de indústria
    let actualCount = d.count;
    if (filterIndustry !== 'all' && allData && allData.length > 0) {
      actualCount = allData.filter(person => 
        person.country === d.country && person.industries === filterIndustry
      ).length;
    }
    
    return {
      ...d,
      count: actualCount,
      region: countryRegions[d.country] || 'Other',
      color: regionColors[countryRegions[d.country]] || regionColors.default,
      totalWealth: countryWealth
    };
  });

  $: filteredData = processedData.filter(d => {
    const regionMatch = filterRegion === 'all' || d.region === filterRegion;
    const hasData = d.count > 0; // Só mostrar países com dados após filtro de indústria
    return regionMatch && hasData;
  });

  // Ordenação padrão por número de bilionários (decrescente)
  $: sortedData = [...filteredData].sort((a, b) => b.count - a.count);

  $: uniqueRegions = [...new Set(processedData.map(d => d.region))].sort();

  // Reagir a mudanças nos dados ou filtros para redesenhar o gráfico
  $: if (svgElement && sortedData.length > 0) {
    drawChart();
  }

  // Também reagir a mudanças nas dimensões
  $: if (svgElement && (width || height)) {
    innerWidth = width - margin.left - margin.right;
    innerHeight = height - margin.top - margin.bottom;
    if (sortedData.length > 0) {
      drawChart();
    }
  }

  function drawChart() {
    if (!svgElement || sortedData.length === 0) return;

    // Limpar completamente o SVG antes de redesenhar
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

    // Título menor
    svg.append('text').attr('x', width / 2).attr('y', 25).attr('text-anchor', 'middle')
      .style('font-size', '14px').style('font-weight', 'bold').style('fill', '#e0e0e0')
      .text('Concentração Global de Bilionários');

    const chartGroup = svg.append('g').attr('transform', `translate(${margin.left},${margin.top})`);

    // Recalcular todas as escalas com base nos dados filtrados atuais
    const y = d3.scaleBand()
      .range([0, innerHeight])
      .domain(sortedData.map(d => d.country))
      .padding(0.15);
    
    const maxValue = d3.max(sortedData, d => d.count) || 10;
    const x = d3.scaleLinear()
      .domain([0, maxValue])
      .range([0, innerWidth]);

    // Criar eixo Y (países) com novo domínio - fonte menor
    const yAxis = chartGroup.append('g')
      .call(d3.axisLeft(y));
    
    yAxis.selectAll('text')
      .style('fill', '#e0e0e0')
      .style('font-size', '10px') // Reduzido de 12px
      .style('text-anchor', 'end')
      .each(function(d) {
        const text = d3.select(this);
        const words = d.split(' ');
        if (words.length > 1 && d.length > 12) {
          text.text('');
          text.append('tspan').attr('x', -10).attr('dy', '-0.2em').text(words[0]);
          text.append('tspan').attr('x', -10).attr('dy', '1.2em').text(words.slice(1).join(' '));
        }
      });

    // Gerar ticks mais proporcionais incluindo o valor máximo
    const tickCount = Math.min(6, Math.floor(maxValue / 50) + 1);
    const tickValues = x.ticks(tickCount);
    
    // Sempre incluir o valor máximo se não estiver muito próximo do último tick
    if (!tickValues.includes(maxValue) && (maxValue - tickValues[tickValues.length - 1]) > maxValue * 0.1) {
      tickValues.push(maxValue);
    }

    // Criar eixo X (números) com novos ticks - fonte menor
    const xAxis = chartGroup.append('g')
      .attr('transform', `translate(0,${innerHeight})`)
      .call(d3.axisBottom(x).tickFormat(d3.format('d')).tickValues(tickValues));
    
    xAxis.selectAll('text')
      .style('fill', '#e0e0e0')
      .style('font-size', '10px'); // Reduzido de 12px

    // Criar barras com posicionamento correto
    const bars = chartGroup.selectAll('rect')
      .data(sortedData)
      .join('rect')
      .attr('y', d => y(d.country))
      .attr('x', 0)
      .attr('height', y.bandwidth())
      .attr('width', 0)
      .attr('fill', d => `url(#gradient-${d.region.replace(/\s+/g, '-')}` + ")")
      .attr('stroke', 'rgba(255,255,255,0.2)')
      .attr('stroke-width', 0.5)
      .attr('rx', 4)
      .attr('ry', 4)
      .style('cursor', 'pointer')
      .style('pointer-events', 'none');

    // Animar as barras
    bars.transition()
      .duration(800)
      .delay((d, i) => i * 40)
      .attr('width', d => x(d.count))
      .on('end', function() {
        d3.select(this).style('pointer-events', 'auto');
      });

    // Criar labels das barras - fonte menor
    const labels = chartGroup.selectAll('.bar-label')
      .data(sortedData)
      .join('text')
      .attr('class', 'bar-label')
      .attr('x', 5)
      .attr('y', d => y(d.country) + y.bandwidth() / 2)
      .attr('dy', '0.35em')
      .style('font-size', '9px') // Reduzido de 11px
      .style('font-weight', 'bold')
      .style('opacity', 0);

    labels.transition()
      .duration(800)
      .delay((d, i) => i * 40 + 400)
      .style('opacity', 1)
      .attr('x', d => {
        const barWidth = x(d.count);
        return barWidth < 60 ? barWidth + 8 : barWidth - 8;
      })
      .attr('text-anchor', d => x(d.count) < 60 ? 'start' : 'end')
      .style('fill', d => x(d.count) < 60 ? '#e0e0e0' : '#ffffff')
      .text(d => d.count.toLocaleString());

    // Adicionar interações de hover
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

    // Adicionar labels dos eixos - fonte menor
    chartGroup.append('text')
      .attr('text-anchor', 'middle')
      .attr('x', innerWidth / 2)
      .attr('y', innerHeight + 35) // Ajustado
      .text('Número de Bilionários')
      .style('fill', '#e0e0e0')
      .style('font-size', '11px'); // Reduzido de 14px
      
    chartGroup.append('text')
      .attr('text-anchor', 'middle')
      .attr('transform', 'rotate(-90)')
      .attr('y', -80) // Ajustado
      .attr('x', -innerHeight / 2)
      .text('Países')
      .style('fill', '#e0e0e0')
      .style('font-size', '11px'); // Reduzido de 14px
  }

  function showTooltip(event, d) {
    if (!tooltip) return;
    const percentage = ((d.count / d3.sum(processedData, d => d.count)) * 100).toFixed(1);
    const wealthFormatted = d.totalWealth >= 1000 
      ? `$${(d.totalWealth / 1000).toFixed(1)}T` 
      : `$${d.totalWealth.toFixed(1)}B`;
    
    tooltip.style('opacity', 1).html(`
      <div style="background: rgba(0,0,0,0.9); padding: 12px; border-radius: 8px; color: white; font-size: 13px; box-shadow: 0 4px 12px rgba(0,0,0,0.3); border: 1px solid ${d.color};">
        <div style="font-weight: bold; margin-bottom: 8px; color: ${d.color};">${d.country}</div>
        <div><strong>Bilionários:</strong> ${d.count.toLocaleString()}</div>
        <div><strong>Riqueza Total:</strong> ${wealthFormatted}</div>
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
      const containerWidth = svgElement.parentElement.clientWidth > 0 ? svgElement.parentElement.clientWidth : 800;
      const containerHeight = svgElement.parentElement.clientHeight > 0 ? svgElement.parentElement.clientHeight : 400;
      
      // Limitar dimensões máximas para caber na tela
      width = Math.min(containerWidth, 800);
      height = Math.min(containerHeight, 400);
      
      if (width < 300) width = 300;
      if (height < 250) height = 250; 
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
</script>

<div class="chart-wrapper">
  <div class="controls">
    <div class="filters-row">
      <div class="control-group">
        <label for="region-select">Região:</label>
        <select id="region-select" bind:value={filterRegion}>
          <option value="all">Todas</option>
          {#each uniqueRegions as region}
            <option value={region}>{region}</option>
          {/each}
        </select>
      </div>
      
      <div class="control-group">
        <label>Indústria:</label>
        <div class="chips">
          <button 
            class="chip {filterIndustry === 'all' ? 'selected' : ''}"
            on:click={() => filterIndustry = 'all'}
          >
            Todos
          </button>

          {#each visibleIndustries as industry}
            <button 
              class="chip {filterIndustry === industry ? 'selected' : ''}"
              on:click={() => filterIndustry = industry}
            >
              {industry}
            </button>
          {/each}

          {#if !showAllIndustries && hiddenCount > 0}
            <button class="chip" on:click={() => showAllIndustries = true}>
              Mais +{hiddenCount}
            </button>
          {/if}
        </div>
      </div>
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
    padding: 15px; /* Reduzido de 20px */
    backdrop-filter: blur(10px);
    border: 1px solid rgba(255, 255, 255, 0.1);
  }

  .controls { 
    display: flex; 
    justify-content: center; 
    margin-bottom: 15px; /* Reduzido de 20px */
  }
  
  .filters-row {
    display: flex;
    gap: 20px;
    align-items: flex-end;
    flex-wrap: wrap;
  }
  
  .control-group { 
    display: flex; 
    flex-direction: column; 
    gap: 4px; /* Reduzido de 5px */
  }
  
  .control-group label { 
    color: #e0e0e0; 
    font-size: 10px; /* Reduzido de 12px */
    font-weight: 500; 
    margin: 0;
  }
  
  .control-group select {
    padding: 6px 10px; /* Reduzido de 8px 12px */
    border-radius: 6px; 
    border: 1px solid rgba(255, 255, 255, 0.2);
    background: rgba(0, 0, 0, 0.3); 
    color: #e0e0e0; 
    font-size: 11px; /* Reduzido de 13px */
    cursor: pointer;
    min-width: 100px;
    transition: all 0.2s ease;
  }
  
  .control-group select:hover {
    background: rgba(0, 0, 0, 0.4);
    border-color: rgba(255, 255, 255, 0.3);
  }
  
  .control-group select:focus {
    outline: none; 
    border-color: #4ecdc4; 
    box-shadow: 0 0 0 2px rgba(78, 205, 196, 0.2);
  }

  .chart-container {
    width: 100%; 
    height: 400px; /* Reduzido de 450px */
    display: flex; 
    justify-content: center; 
    align-items: center;
  }
  
  svg { 
    max-width: 100%; 
    max-height: 100%; 
  }

  .legend {
    margin-top: 12px; /* Reduzido de 15px */
    border-top: 1px solid rgba(255, 255, 255, 0.1); 
    padding-top: 12px; /* Reduzido de 15px */
  }
  
  .legend h4 { 
    color: #e0e0e0; 
    margin: 0 0 8px 0; /* Reduzido de 10px */
    font-size: 12px; /* Reduzido de 14px */
  }
  
  .legend-items { 
    display: flex; 
    flex-wrap: wrap; 
    gap: 12px; /* Reduzido de 15px */
  }
  
  .legend-item { 
    display: flex; 
    align-items: center; 
    gap: 5px; /* Reduzido de 6px */
    font-size: 10px; /* Reduzido de 12px */
    color: #e0e0e0; 
  }
  
  .legend-color {
    width: 14px; /* Reduzido de 16px */
    height: 14px; /* Reduzido de 16px */
    border-radius: 3px; 
    border: 1px solid rgba(255, 255, 255, 0.2);
  }

  .chips {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
    margin-top: 4px;
  }

  .chip {
    padding: 6px 14px;
    background: rgba(0,0,0,0.4);
    color: #e0e0e0;
    border: 1px solid rgba(255,255,255,0.2);
    border-radius: 9999px;
    font-size: 11px;
    cursor: pointer;
    transition: all 0.2s ease;
    white-space: nowrap;
  }

  .chip:hover {
    background: rgba(0,0,0,0.6);
    border-color: rgba(255,255,255,0.3);
  }

  .chip.selected {
    background: #6366f1;
    border-color: #6366f1;
    color: #ffffff;
    font-weight: 600;
    box-shadow: 0 2px 8px rgba(99,102,241,0.3);
  }

  @media (max-width: 768px) {
    .chart-wrapper { 
      padding: 12px; 
    }
    
    .filters-row {
      justify-content: center;
      gap: 15px;
    }
    
    .chart-container { 
      height: 350px;
    }
    
    .control-group select {
      min-width: 80px;
      font-size: 10px;
    }
    
    .legend-items {
      justify-content: center;
    }

    .chips {
      justify-content: center;
    }
  }
</style> 