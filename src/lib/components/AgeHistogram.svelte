<script>
  export let data = []; // Agora espera dados brutos dos bilionários
  import { onMount } from 'svelte';
  import * as d3 from 'd3';

  let svgElement, tooltip;
  let selectedIndustry = 'Todos';
  let industries = ['Todos'];
  let filteredData = []; // Dados filtrados por indústria
  
  const margin = { top: 60, right: 40, bottom: 80, left: 80 };
  let width = 600, height = 380;
  let innerWidth = width - margin.left - margin.right;
  let innerHeight = height - margin.top - margin.bottom;

  const ageGroupOrder = ['10-19', '20-29', '30-39', '40-49', '50-59', '60-69', '70-79', '80-89', '90-99', '100-109'];
  const uniformColor = '#6366f1';

  // Extrair indústrias dos dados (similar ao mapa de migração)
  function extractFilters() {
    const industriesSet = new Set();
    
    data.forEach(person => {
      if (person.industries) industriesSet.add(person.industries);
    });
    
    industries = ['Todos', ...Array.from(industriesSet).sort()];
  }

  // Filtrar dados por indústria (similar ao mapa de migração)
  function updateFilteredData() {
    if (!data || data.length === 0) {
      filteredData = [];
      return;
    }
    
    filteredData = data.filter(person => {
      let industryMatch = false;
      if (selectedIndustry === 'Todos') {
        industryMatch = true;
      } else if (person.industries) {
        industryMatch = person.industries.toString().trim() === selectedIndustry.toString().trim();
      }
      
      return industryMatch;
    });
  }

  // Agrupar dados filtrados por faixa etária
  function processAgeGroups() {
    const ageGroups = {};
    ageGroupOrder.forEach(group => {
      ageGroups[group] = 0;
    });
    
    filteredData.forEach(person => {
      if (person.age) {
        const age = parseInt(person.age);
        if (age >= 10 && age < 20) ageGroups['10-19']++;
        else if (age >= 20 && age < 30) ageGroups['20-29']++;
        else if (age >= 30 && age < 40) ageGroups['30-39']++;
        else if (age >= 40 && age < 50) ageGroups['40-49']++;
        else if (age >= 50 && age < 60) ageGroups['50-59']++;
        else if (age >= 60 && age < 70) ageGroups['60-69']++;
        else if (age >= 70 && age < 80) ageGroups['70-79']++;
        else if (age >= 80 && age < 90) ageGroups['80-89']++;
        else if (age >= 90 && age < 100) ageGroups['90-99']++;
        else if (age >= 100 && age < 110) ageGroups['100-109']++;
      }
    });
    
    return Object.entries(ageGroups)
      .map(([ageGroup, count]) => ({ ageGroup, count, color: uniformColor }))
      .filter(d => d.count > 0)
      .sort((a, b) => ageGroupOrder.indexOf(a.ageGroup) - ageGroupOrder.indexOf(b.ageGroup));
  }

  // Inicializar quando os dados chegarem
  $: if (data.length > 0) {
    extractFilters();
    updateFilteredData();
  }

  // Reagir a mudanças no filtro de indústria
  $: if (selectedIndustry && data.length > 0) {
    updateFilteredData();
  }

  // Processar dados para o gráfico
  $: processedData = filteredData.length > 0 ? processAgeGroups() : [];

  $: totalBillionaires = d3.sum(processedData, d => d.count);

  // Redesenhar quando processedData mudar
  $: if (processedData && svgElement) {
    drawChart();
  }

  function drawChart() {
    if (!svgElement || processedData.length === 0) return;

    d3.select(svgElement).selectAll('*').remove();

    const svg = d3.select(svgElement)
      .attr('width', width)
      .attr('height', height);

    svg.append('text')
      .attr('x', width / 2)
      .attr('y', 30)
      .attr('text-anchor', 'middle')
      .style('font-size', '18px')
      .style('font-weight', 'bold')
      .style('fill', '#e0e0e0')
      .text('Distribuição de Bilionários por Idade');

    const chartGroup = svg.append('g')
      .attr('transform', `translate(${margin.left},${margin.top})`);

    const x = d3.scaleBand()
      .range([0, innerWidth])
      .domain(ageGroupOrder.filter(age => processedData.some(d => d.ageGroup === age)))
      .padding(0.2);

    chartGroup.append('g')
      .attr('transform', `translate(0,${innerHeight})`)
      .call(d3.axisBottom(x))
      .selectAll('text')
        .style('fill', '#e0e0e0')
        .style('font-size', '12px');

    const maxValue = d3.max(processedData, d => d.count) || 10;
    const y = d3.scaleLinear()
      .domain([0, maxValue])
      .range([innerHeight, 0]);

    // Gerar ticks mais proporcionais incluindo o valor máximo
    const tickCount = Math.min(6, Math.floor(maxValue / 20) + 1);
    const tickValues = y.ticks(tickCount);
    
    // Sempre incluir o valor máximo se não estiver muito próximo do último tick
    if (!tickValues.includes(maxValue) && (maxValue - tickValues[tickValues.length - 1]) > maxValue * 0.1) {
      tickValues.push(maxValue);
    }

    chartGroup.append('g')
      .call(d3.axisLeft(y).tickFormat(d3.format('d')).tickValues(tickValues))
      .selectAll('text')
        .style('fill', '#e0e0e0');

    const bars = chartGroup.selectAll('.age-bar')
      .data(processedData)
      .join('rect')
      .attr('class', 'age-bar')
      .attr('x', d => x(d.ageGroup))
      .attr('y', innerHeight)
      .attr('width', x.bandwidth())
      .attr('height', 0)
      .attr('fill', uniformColor)
      .attr('stroke', 'rgba(255,255,255,0.2)')
      .attr('stroke-width', 1)
      .attr('rx', 4)
      .attr('ry', 4)
      .style('cursor', 'pointer');

    bars.transition()
      .duration(800)
      .delay((d, i) => i * 100)
      .attr('y', d => y(d.count))
      .attr('height', d => innerHeight - y(d.count));

    const labels = chartGroup.selectAll('.age-label')
      .data(processedData)
      .join('text')
      .attr('class', 'age-label')
      .attr('x', d => x(d.ageGroup) + x.bandwidth() / 2)
      .attr('y', innerHeight)
      .attr('text-anchor', 'middle')
      .style('fill', '#fff')
      .style('font-weight', 'bold')
      .style('font-size', '12px')
      .style('opacity', 0);

    labels.transition()
      .duration(800)
      .delay((d, i) => i * 100 + 400)
      .attr('y', d => y(d.count) - 8)
      .style('opacity', 1)
      .text(d => d.count.toLocaleString());

    bars
      .on('mouseover', function(event, d) {
        d3.select(this)
          .transition()
          .duration(200)
          .attr('opacity', 0.8)
          .attr('stroke-width', 2)
          .attr('stroke', '#ffffff');
        
        showTooltip(event, d);
      })
      .on('mouseout', function() {
        d3.select(this)
          .transition()
          .duration(200)
          .attr('opacity', 1)
          .attr('stroke-width', 1)
          .attr('stroke', 'rgba(255,255,255,0.2)');
        
        hideTooltip();
      })
      .on('mousemove', function(event) {
        updateTooltipPosition(event);
      });

    chartGroup.append('text')
      .attr('text-anchor', 'middle')
      .attr('x', innerWidth / 2)
      .attr('y', innerHeight + 50)
      .text('Faixa Etária')
      .style('fill', '#e0e0e0')
      .style('font-size', '14px');

    chartGroup.append('text')
      .attr('text-anchor', 'middle')
      .attr('transform', 'rotate(-90)')
      .attr('y', -50)
      .attr('x', -innerHeight / 2)
      .text('Número de Bilionários')
      .style('fill', '#e0e0e0')
      .style('font-size', '14px');
  }

  function showTooltip(event, d) {
    if (!tooltip) return;
    const percentage = totalBillionaires > 0 ? ((d.count / totalBillionaires) * 100).toFixed(1) : '0';
    const content = `<div style="background: rgba(0,0,0,0.9); padding: 12px; border-radius: 8px; color: white; font-size: 13px; box-shadow: 0 4px 12px rgba(0,0,0,0.3); border: 1px solid ${d.color};">
      <div style="font-weight: bold; margin-bottom: 8px; color: ${d.color};">Faixa: ${d.ageGroup} anos</div>
      <div><strong>Bilionários:</strong> ${d.count.toLocaleString()}</div>
      <div><strong>Porcentagem:</strong> ${percentage}%</div></div>`;
    tooltip.style('opacity', 1).html(content);
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
      width = svgElement.parentElement.clientWidth > 0 ? svgElement.parentElement.clientWidth : 600;
      height = svgElement.parentElement.clientHeight > 0 ? svgElement.parentElement.clientHeight : 380;
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
</script>

<div class="chart-wrapper">
  {#if industries.length > 0}
    <div class="controls-panel">
      <div class="industry-slicer">
        <h4 class="slicer-title">Filtrar por Indústria</h4>
        <div class="industry-grid">
          {#each industries.slice(0, 8) as industry}
            <button 
              class="industry-chip {selectedIndustry === industry ? 'active' : ''}"
              on:click={() => selectedIndustry = industry}
            >
              {industry}
              {#if selectedIndustry === industry}
                <span class="check-mark">✓</span>
              {/if}
            </button>
          {/each}
          {#if industries.length > 8}
            <details class="more-industries">
              <summary class="more-toggle">Mais +{industries.length - 8}</summary>
              <div class="more-industry-grid">
                {#each industries.slice(8) as industry}
                  <button 
                    class="industry-chip {selectedIndustry === industry ? 'active' : ''}"
                    on:click={() => selectedIndustry = industry}
                  >
                    {industry}
                    {#if selectedIndustry === industry}
                      <span class="check-mark">✓</span>
                    {/if}
                  </button>
                {/each}
              </div>
            </details>
          {/if}
        </div>
      </div>
      <div class="filter-info">
        <span class="total-count">Total: {totalBillionaires} bilionários</span>
        {#if selectedIndustry !== 'Todos'}
          <span class="filter-active">Filtro ativo: {selectedIndustry}</span>
        {/if}
      </div>
    </div>
  {/if}
  
  <div class="chart-container" bind:clientWidth={width} bind:clientHeight={height}>
    <svg bind:this={svgElement}></svg>
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

  .controls-panel {
    display: flex;
    justify-content: space-between;
    align-items: flex-start;
    gap: 20px;
    padding: 20px;
    background: #2a2a2a;
    border-radius: 12px;
    margin-bottom: 20px;
    border: 1px solid #444;
    flex-wrap: wrap;
  }

  .industry-slicer {
    flex: 1;
    min-width: 300px;
  }

  .slicer-title {
    color: #e0e0e0;
    font-size: 16px;
    font-weight: 600;
    margin: 0 0 15px 0;
    display: flex;
    align-items: center;
    gap: 8px;
  }

  .industry-grid {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
    margin-bottom: 10px;
  }

  .industry-chip {
    padding: 8px 14px;
    background: #1a1a1a;
    color: #e0e0e0;
    border: 1px solid #555;
    border-radius: 20px;
    cursor: pointer;
    font-size: 12px;
    font-weight: 500;
    transition: all 0.2s ease;
    display: flex;
    align-items: center;
    gap: 6px;
    white-space: nowrap;
  }

  .industry-chip:hover {
    background: #333;
    border-color: #6366f1;
    transform: translateY(-1px);
  }

  .industry-chip.active {
    background: linear-gradient(135deg, #6366f1, #4f46e5);
    color: white;
    border-color: #6366f1;
    box-shadow: 0 2px 8px rgba(99, 102, 241, 0.3);
  }

  .check-mark {
    font-size: 10px;
    font-weight: bold;
  }

  .more-industries {
    position: relative;
  }

  .more-toggle {
    padding: 8px 14px;
    background: #444;
    color: #e0e0e0;
    border: 1px solid #666;
    border-radius: 20px;
    cursor: pointer;
    font-size: 12px;
    font-weight: 500;
    list-style: none;
    transition: all 0.2s ease;
  }

  .more-toggle:hover {
    background: #555;
    border-color: #6366f1;
  }

  .more-industry-grid {
    position: absolute;
    top: 100%;
    left: 0;
    z-index: 10;
    background: #2a2a2a;
    border: 1px solid #444;
    border-radius: 8px;
    padding: 12px;
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
    min-width: 200px;
    box-shadow: 0 4px 15px rgba(0, 0, 0, 0.3);
    margin-top: 4px;
  }

  .filter-info {
    display: flex;
    flex-direction: column;
    gap: 5px;
    align-items: flex-end;
  }

  .total-count {
    color: #6366f1;
    font-weight: 600;
    font-size: 14px;
  }

  .filter-active {
    color: #ffd700;
    font-size: 12px;
    font-style: italic;
  }

  .chart-container {
    width: 100%;
    height: 380px;
    display: flex;
    justify-content: center;
    align-items: center;
  }

  svg { max-width: 100%; max-height: 100%; }

  @media (max-width: 768px) {
    .chart-wrapper { padding: 15px; }
    .chart-container { height: 320px; }
    
    .controls-panel {
      flex-direction: column;
      gap: 15px;
      align-items: stretch;
    }
    

    .filter-info {
      align-items: center;
      text-align: center;
    }
    
  }
</style> 