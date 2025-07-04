<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';

  export let data = [];
  
  let svgElement;
  let selectedCountries = new Set();
  let availableCountries = [];
  
  const width = 380;
  const height = 380;
  const margin = 60;
  const radius = Math.min(width, height) / 2 - margin;
  
  const indicators = [
    { key: 'totalWealth', label: 'Riqueza Total', unit: 'trilhões USD', inverse: false },
    { key: 'gdp', label: 'PIB', unit: 'trilhões USD', inverse: false },
    { key: 'lifeExpectancy', label: 'Expectativa de Vida', unit: 'anos', inverse: false },
    { key: 'taxRate', label: 'Taxa de Impostos', unit: '%', inverse: true },
    { key: 'corruptionIndex', label: 'CPI (Inflação)', unit: 'índice', inverse: true }
  ];
  
  let processedData = [];
  let scales = {};
  
  const colors = ['#ffd700', '#4ecdc4', '#a259ec', '#10b981', '#f39c12', '#9b59b6', '#e74c3c', '#2ecc71'];
  
  $: if (data.length > 0) {
    processData();
  }
  
  onMount(() => {
    drawChart();
  });
  
  function processData() {
    const countryData = d3.rollups(data, v => {
      const totalWealth = d3.sum(v, d => d.finalWorth || 0) / 1000;
      const gdpSample = v.find(d => d.gdp_country);
      const gdp = gdpSample ? parseFloat(gdpSample.gdp_country.replace(/[\$,]/g, '')) / 1e12 : 0;
      const lifeExpectancy = gdpSample ? parseFloat(gdpSample.life_expectancy_country) || 0 : 0;
      const taxRate = gdpSample ? parseFloat(gdpSample.total_tax_rate_country) || 0 : 0;
      const corruptionIndex = gdpSample ? parseFloat(gdpSample.cpi_country) || 0 : 0;
      
      return {
        country: v[0].country,
        totalWealth,
        gdp,
        lifeExpectancy,
        taxRate,
        corruptionIndex,
        billionaireCount: v.length
      };
    }, d => d.country)
    .map(([country, stats]) => ({ ...stats, country }))
    .filter(d => d.country && d.country !== 'Unknown' && d.gdp > 0)
    .sort((a, b) => b.billionaireCount - a.billionaireCount)
    .slice(0, 8);
    
    availableCountries = countryData.map(d => d.country);
    
    indicators.forEach(indicator => {
      const values = countryData.map(d => d[indicator.key]).filter(v => v > 0);
      if (values.length > 0) {
        const extent = d3.extent(values);
        scales[indicator.key] = d3.scaleLinear()
          .domain(indicator.inverse ? extent.reverse() : extent)
          .range([0, 1]);
      }
    });
    
    processedData = countryData.map(d => ({
      ...d,
      normalized: indicators.map(indicator => ({
        indicator: indicator.key,
        label: indicator.label,
        value: scales[indicator.key] ? scales[indicator.key](d[indicator.key]) : 0,
        rawValue: d[indicator.key],
        unit: indicator.unit
      }))
    }));
    
    selectedCountries = new Set(availableCountries.slice(0, 3));
  }
  
  function drawChart() {
    if (!svgElement || processedData.length === 0) return;
    
    const svg = d3.select(svgElement);
    svg.selectAll("*").remove();
    
    const center = [width / 2, height / 2];
    const g = svg.append('g').attr('transform', `translate(${center[0]}, ${center[1]})`);
    
    const circles = [0.2, 0.4, 0.6, 0.8, 1.0];
    circles.forEach(r => {
      g.append('circle')
        .attr('cx', 0).attr('cy', 0).attr('r', r * radius)
        .attr('fill', 'none').attr('stroke', '#444').attr('stroke-width', 1).attr('opacity', 0.4);
    });
    
    const angleSlice = (Math.PI * 2) / indicators.length;
    
    indicators.forEach((indicator, i) => {
      const angle = angleSlice * i - Math.PI / 2;
      const x = Math.cos(angle) * radius;
      const y = Math.sin(angle) * radius;
      
      g.append('line')
        .attr('x1', 0).attr('y1', 0).attr('x2', x).attr('y2', y)
        .attr('stroke', '#666').attr('stroke-width', 2);
      
      const labelRadius = radius + 25;
      const labelX = Math.cos(angle) * labelRadius;
      const labelY = Math.sin(angle) * labelRadius;
      

      let textAnchor = 'middle';
      if (labelX > 10) textAnchor = 'start';
      else if (labelX < -10) textAnchor = 'end';
      
      const text = g.append('text')
        .attr('x', labelX).attr('y', labelY)
        .attr('text-anchor', textAnchor).attr('dominant-baseline', 'middle')
        .attr('fill', '#e0e0e0').attr('font-size', '10px').attr('font-weight', 'bold');
      
      
      const words = indicator.label.split(' ');
      if (words.length > 1 && indicator.label.length > 12) {
        text.append('tspan').attr('x', labelX).attr('dy', '-0.5em').text(words[0]);
        text.append('tspan').attr('x', labelX).attr('dy', '1em').text(words.slice(1).join(' '));
      } else {
        text.text(indicator.label);
      }
    });
    
    const selectedData = processedData.filter(d => selectedCountries.has(d.country));
    
    selectedData.forEach((countryData, countryIndex) => {
      const color = colors[countryIndex % colors.length];
      
      const pathCoords = countryData.normalized.map((d, i) => {
        const angle = angleSlice * i - Math.PI / 2;
        const r = d.value * radius;
        return [Math.cos(angle) * r, Math.sin(angle) * r];
      });
      
      pathCoords.push(pathCoords[0]);
      
      g.append('path')
        .datum(pathCoords).attr('d', d3.line())
        .attr('fill', color).attr('fill-opacity', 0.15)
        .attr('stroke', color).attr('stroke-width', 2.5).attr('stroke-opacity', 0.9);
      
      countryData.normalized.forEach((d, i) => {
        const angle = angleSlice * i - Math.PI / 2;
        const r = d.value * radius;
        const x = Math.cos(angle) * r;
        const y = Math.sin(angle) * r;
        
        g.append('circle')
          .attr('cx', x).attr('cy', y).attr('r', 3)
          .attr('fill', color).attr('stroke', '#fff').attr('stroke-width', 1.5)
          .style('cursor', 'pointer')
          .on('mouseover', function(event) {
            showTooltip(event, countryData.country, d);
          })
          .on('mouseout', hideTooltip);
      });
    });
    
    circles.forEach((r, i) => {
      if (i > 0) {
        g.append('text')
          .attr('x', 8).attr('y', -r * radius + 4)
          .attr('fill', '#888').attr('font-size', '9px').attr('font-weight', '500')
          .text(`${(r * 100).toFixed(0)}%`);
      }
    });
    
    g.append('text')
      .attr('x', 0).attr('y', 5)
      .attr('text-anchor', 'middle').attr('fill', '#666')
      .attr('font-size', '10px').attr('font-weight', 'bold')
      .text('0%');
  }
  
  function toggleCountry(country) {
    if (selectedCountries.has(country)) {
      selectedCountries.delete(country);
    } else {
      if (selectedCountries.size >= 4) {
        const firstCountry = Array.from(selectedCountries)[0];
        selectedCountries.delete(firstCountry);
      }
      selectedCountries.add(country);
    }
    selectedCountries = new Set(selectedCountries);
    drawChart();
  }
  
  function showTooltip(event, country, indicator) {
    const tooltip = d3.select('body').append('div')
      .attr('class', 'radar-tooltip')
      .style('position', 'absolute').style('background', 'rgba(0, 0, 0, 0.9)')
      .style('color', '#fff').style('padding', '12px').style('border-radius', '4px')
      .style('font-size', '12px').style('pointer-events', 'none')
      .style('border', '1px solid #ffd700').style('box-shadow', '0 4px 8px rgba(0,0,0,0.3)')
      .style('opacity', 0);
    
    const countryData = processedData.find(d => d.country === country);
    const percentValue = (indicator.value * 100).toFixed(0);
    
    let contextInfo = '';
    if (indicator.indicator === 'totalWealth') {
      contextInfo = `<br><small style="color: #ccc">Total de ${countryData.billionaireCount} bilionários</small>`;
    }
    
    tooltip.html(`
      <strong style="color: #ffd700">${country}</strong><br>
      <strong>${indicator.label}</strong><br>
      Valor: ${indicator.rawValue.toFixed(2)} ${indicator.unit}<br>
      Posição relativa: ${percentValue}%${contextInfo}
    `);
    
    // Posicionamento inteligente do tooltip
    const tooltipRect = tooltip.node().getBoundingClientRect();
    const windowWidth = window.innerWidth;
    const windowHeight = window.innerHeight;
    
    let left = event.pageX + 15;
    let top = event.pageY - 10;
    
    // Ajustar se o tooltip sair da tela à direita
    if (left + tooltipRect.width > windowWidth) {
      left = event.pageX - tooltipRect.width - 15;
    }
    
    // Ajustar se o tooltip sair da tela abaixo
    if (top + tooltipRect.height > windowHeight) {
      top = event.pageY - tooltipRect.height - 10;
    }
    
    tooltip.style('left', left + 'px')
           .style('top', top + 'px')
           .transition().duration(200).style('opacity', 1);
    
    setTimeout(() => {
      tooltip.transition().duration(200).style('opacity', 0).remove();
    }, 4000);
  }
  
  function hideTooltip() {
    d3.selectAll('.radar-tooltip').remove();
  }
</script>

<div class="radar-chart-container">
  <h3 class="radar-title">Comparativo de Indicadores por País</h3>
  <div class="country-selector">
    <h4>Selecionar Países (máximo 4):</h4>
    <div class="country-buttons">
      {#each availableCountries as country, i}
        <button
          class="country-button {selectedCountries.has(country) ? 'selected' : ''}"
          style="border-color: {selectedCountries.has(country) ? colors[Array.from(selectedCountries).indexOf(country) % colors.length] : '#666'}; {selectedCountries.has(country) ? `background-color: ${colors[Array.from(selectedCountries).indexOf(country) % colors.length]}20` : ''}"
          on:click={() => toggleCountry(country)}
        >
          {country}
        </button>
      {/each}
    </div>
  </div>
  
  <div class="chart-and-legend-wrapper">
    <div class="chart-wrapper">
      <svg bind:this={svgElement} {width} {height}></svg>
      
      {#if selectedCountries.size > 0}
        <div class="selected-countries">
          {#each Array.from(selectedCountries) as country, i}
            <span class="country-item" style="color: {colors[i % colors.length]}">
              ● {country}
            </span>
          {/each}
        </div>
      {/if}
    </div>
    
    <div class="legend">
      <h4>Indicadores:</h4>
      <p class="normalization-note">
        <strong>Atenção:</strong> Os valores são normalizados de 0-100% em relação ao país com maior valor em cada indicador.
      </p>
      <div class="legend-items">
        {#each indicators as indicator}
          <div class="legend-item">
            <span class="legend-label">{indicator.label}:</span>
            <span class="legend-description">
              {#if indicator.key === 'totalWealth'}
                Soma total da riqueza dos bilionários do país. MAIOR = MELHOR (mais riqueza).
              {:else if indicator.key === 'gdp'}
                PIB do país em trilhões USD. MAIOR = MELHOR (economia forte).
              {:else if indicator.key === 'lifeExpectancy'}
                Expectativa de vida em anos. MAIOR = MELHOR (viver mais).
              {:else if indicator.key === 'taxRate'}
                Taxa de impostos total. MENOR = MELHOR (menos impostos).
              {:else if indicator.key === 'corruptionIndex'}
                Índice de Preços ao Consumidor (inflação). MENOR = MELHOR (baixa inflação).
              {/if}
            </span>
          </div>
        {/each}
      </div>
    </div>
  </div>
</div>

<style>
  .radar-chart-container {
    display: flex;
    flex-direction: column;
    gap: 10px; 
    padding: 12px;
    background: transparent;
    border-radius: 6px;
    max-width: 800px; 
    margin: 0 auto;
  }
  
  .radar-title {
    color: #ffd700;
    text-align: center;
    font-size: 1.2em;
    font-weight: 700;
    margin-bottom: 6px;
    margin-top: 0;
    letter-spacing: 0.01em;
  }
  
  .country-selector {
    text-align: center;
  }
  
  .country-selector h4 {
    color: #fff;
    margin: 0 0 8px 0; 
    font-size: 13px;
    font-weight: 600;
  }
  
  .country-buttons {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(100px, 1fr));
    gap: 5px;
    max-width: 550px; 
    margin: 0 auto;
  }
  
  .country-button {
    padding: 4px 6px;
    background: transparent;
    border: 2px solid #666;
    border-radius: 3px;
    color: #e0e0e0;
    cursor: pointer;
    font-size: 9px;
    font-weight: 500;
    transition: all 0.2s ease;
  }
  
  .country-button:hover {
    background: rgba(255, 255, 255, 0.1);
  }
  
  .country-button.selected {
    color: #ffd700;
    border-width: 2px;
    font-weight: 600;
  }
  
  .chart-and-legend-wrapper {
    display: flex;
    flex-direction: row;
    align-items: flex-start;
    gap: 15px; 
    justify-content: center;
  }
  
  .chart-wrapper {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 8px;
    flex-shrink: 0;
  }
  
    .selected-countries {
    display: flex;
    flex-wrap: wrap;
    justify-content: center;
    gap: 8px; 
    margin-top: 4px; 
  }

  .country-item {
    font-size: 10px; 
    font-weight: 600;
  }
  
    .legend {
    flex: 1;
    max-width: 250px; 
    min-width: 220px; 
  }

  .legend h4 {
    color: #ffd700;
    margin: 0 0 6px 0; 
    font-size: 13px; 
    text-align: left;
  }
  
  .normalization-note {
    background: rgba(255, 215, 0, 0.1);
    border: 1px solid #ffd700;
    border-radius: 3px;
    padding: 5px; 
    margin-bottom: 8px; 
    font-size: 9px; 
    color: #ffd700;
    line-height: 1.2;
  }
  
  .normalization-note strong {
    color: #ffd700;
  }
  
    .legend-items {
    display: flex;
    flex-direction: column;
    gap: 4px; 
  }

  .legend-item {
    padding: 5px; 
    background: rgba(255, 255, 255, 0.05);
    border-radius: 3px;
    border: 1px solid #444;
  }
  
    .legend-label {
    color: #ffd700;
    font-weight: bold;
    font-size: 10px; 
    display: block;
    margin-bottom: 1px; 
  }

  .legend-description {
    color: #c0c0c0;
    font-size: 8px;
    line-height: 1.2;
  }
  
  @media (max-width: 920px) { 
    .radar-chart-container {
      padding: 10px; 
      gap: 8px;
    }
    
    .radar-title {
      font-size: 1em;
      margin-bottom: 6px;
    }
    
    .chart-and-legend-wrapper {
      flex-direction: column;
      align-items: center;
      gap: 10px; 
    }
    
    .legend {
      max-width: 100%;
      width: 100%;
    }
    
    .legend h4 {
      text-align: center;
    }
  }
  
  @media (max-width: 600px) {
    .radar-chart-container {
      padding: 8px; 
      gap: 6px; 
    }
    
    .country-buttons {
      grid-template-columns: repeat(auto-fit, minmax(85px, 1fr)); 
      gap: 4px; 
    }
    
    .country-button {
      font-size: 8px; 
      padding: 3px 5px; 
    }
    
    .chart-and-legend-wrapper {
      gap: 8px;
    }
    
    .legend-item {
      padding: 4px; 
    }
  }
</style> 