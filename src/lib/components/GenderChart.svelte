<script>
  export let data = [];
  export let allData = [];
  import { onMount, afterUpdate } from 'svelte';
  import * as d3 from 'd3';

  let svgElement;
  let tooltip;
  let showBreakdown = false;
  let selectedGender = null;
  let breakdownData = [];
  
  const margin = { top: 60, right: 30, bottom: 80, left: 80 };
  let width = 550;
  let height = 380;
  let innerWidth = width - margin.left - margin.right;
  let innerHeight = height - margin.top - margin.bottom;

  const genderConfig = {
    'M': { 
      color: '#3b82f6', 
      label: 'Masculino', 
      icon: '♂',
      lightColor: '#60a5fa'
    },
    'F': { 
      color: '#ef4444', 
      label: 'Feminino', 
      icon: '♀',
      lightColor: '#f87171'
    }
  };

  $: totalCount = d3.sum(data, d => d.count);
  $: dataWithPercentage = data.map(d => ({
    ...d,
    percentage: ((d.count / totalCount) * 100).toFixed(1),
    config: genderConfig[d.gender] || { color: '#6b7280', label: d.gender, icon: '?', lightColor: '#9ca3af' }
  }));

  function getTopCountriesByGender(gender, limit = 8) {
    if (!allData || allData.length === 0) return [];
    
    const countryGenderCounts = {};
    
    allData.forEach(person => {
      if (person.gender === gender) {
        const country = person.country || 'Unknown';
        countryGenderCounts[country] = (countryGenderCounts[country] || 0) + 1;
      }
    });

    return Object.entries(countryGenderCounts)
      .map(([country, count]) => ({ country, count, gender }))
      .sort((a, b) => b.count - a.count)
      .slice(0, limit);
  }

  function drawChart() {
    if (!svgElement || dataWithPercentage.length === 0) return;

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
      .text(showBreakdown ? `Bilionários ${selectedGender === 'M' ? 'Masculinos' : 'Femininos'} por País` : 'Distribuição por Gênero');

    const chartGroup = svg.append('g')
      .attr('transform', `translate(${margin.left},${margin.top})`);

    const currentData = showBreakdown ? breakdownData : dataWithPercentage;

    if (showBreakdown) {
      drawBreakdownChart(chartGroup, currentData);
    } else {
      drawMainChart(chartGroup, currentData);
    }
  }

  function drawMainChart(svg, data) {
    const x = d3.scaleBand()
      .range([0, innerWidth])
      .domain(data.map(d => d.gender))
      .padding(0.4);

    const xAxis = svg.append('g')
      .attr('transform', `translate(0,${innerHeight})`);

    xAxis.selectAll('.gender-label')
      .data(data)
      .join('g')
      .attr('class', 'gender-label')
      .attr('transform', d => `translate(${x(d.gender) + x.bandwidth()/2}, 0)`)
      .each(function(d) {
        const g = d3.select(this);
        
        g.append('text')
          .attr('y', 25)
          .attr('text-anchor', 'middle')
          .style('font-size', '24px')
          .style('fill', d.config.color)
          .text(d.config.icon);
        
        g.append('text')
          .attr('y', 45)
          .attr('text-anchor', 'middle')
          .style('fill', '#e0e0e0')
          .style('font-size', '13px')
          .text(d.config.label);
      });

    const y = d3.scaleLinear()
      .domain([0, d3.max(data, d => d.count) || 10])
      .range([innerHeight, 0]);

    svg.append('g')
      .call(d3.axisLeft(y).tickFormat(d3.format('d')))
      .selectAll('text')
        .style('fill', '#e0e0e0');

    const bars = svg.selectAll('.main-bar')
      .data(data)
      .join('rect')
      .attr('class', 'main-bar')
      .attr('x', d => x(d.gender))
      .attr('y', innerHeight)
      .attr('width', x.bandwidth())
      .attr('height', 0)
      .attr('fill', d => d.config.color)
      .attr('stroke', 'rgba(255,255,255,0.3)')
      .attr('stroke-width', 2)
      .style('cursor', 'pointer');

    bars.transition()
      .duration(800)
      .delay((d, i) => i * 150)
      .attr('y', d => y(d.count))
      .attr('height', d => innerHeight - y(d.count));

    const labels = svg.selectAll('.value-label')
      .data(data)
      .join('text')
      .attr('class', 'value-label')
      .attr('x', d => x(d.gender) + x.bandwidth() / 2)
      .attr('y', innerHeight)
      .attr('text-anchor', 'middle')
      .style('fill', '#fff')
      .style('font-weight', 'bold')
      .style('font-size', '14px')
      .style('opacity', 0);

    labels.transition()
      .duration(800)
      .delay((d, i) => i * 150 + 400)
      .attr('y', d => y(d.count) - 10)
      .style('opacity', 1)
      .text(d => d.count.toLocaleString());

    const percentLabels = svg.selectAll('.percent-label')
      .data(data)
      .join('text')
      .attr('class', 'percent-label')
      .attr('x', d => x(d.gender) + x.bandwidth() / 2)
      .attr('y', d => y(d.count) + 20)
      .attr('text-anchor', 'middle')
      .style('fill', '#000')
      .style('font-weight', 'bold')
      .style('font-size', '12px')
      .style('opacity', 0);

    percentLabels.transition()
      .duration(800)
      .delay((d, i) => i * 150 + 600)
      .style('opacity', 1)
      .text(d => `${d.percentage}%`);

    bars
      .on('mouseover', function(event, d) {
        d3.select(this)
          .transition()
          .duration(200)
          .attr('opacity', 0.8)
          .attr('stroke-width', 3);
        
        showTooltip(event, d, false);
      })
      .on('mouseout', function() {
        d3.select(this)
          .transition()
          .duration(200)
          .attr('opacity', 1)
          .attr('stroke-width', 2);
        
        hideTooltip();
      })
      .on('click', function(event, d) {
        if (allData && allData.length > 0) {
          selectedGender = d.gender;
          breakdownData = getTopCountriesByGender(d.gender);
          showBreakdown = true;
          drawChart();
        }
      })
      .on('mousemove', function(event) {
        updateTooltipPosition(event);
      });

    svg.append('text')
      .attr('text-anchor', 'middle')
      .attr('transform', 'rotate(-90)')
      .attr('y', -50)
      .attr('x', -innerHeight / 2)
      .text('Número de Bilionários')
      .style('fill', '#e0e0e0')
      .style('font-size', '14px');
  }

  function drawBreakdownChart(svg, data) {
    const y = d3.scaleBand()
      .range([0, innerHeight])
      .domain(data.map(d => d.country))
      .padding(0.1);

    svg.append('g')
      .call(d3.axisLeft(y))
      .selectAll('text')
        .style('fill', '#e0e0e0')
        .style('font-size', '12px');

    const x = d3.scaleLinear()
      .domain([0, d3.max(data, d => d.count) || 10])
      .range([0, innerWidth]);

    svg.append('g')
      .attr('transform', `translate(0,${innerHeight})`)
      .call(d3.axisBottom(x).tickFormat(d3.format('d')))
      .selectAll('text')
        .style('fill', '#e0e0e0');

    const genderColor = genderConfig[selectedGender]?.color || '#6b7280';

    const bars = svg.selectAll('.breakdown-bar')
      .data(data)
      .join('rect')
      .attr('class', 'breakdown-bar')
      .attr('y', d => y(d.country))
      .attr('x', 0)
      .attr('height', y.bandwidth())
      .attr('width', 0)
      .attr('fill', genderColor)
      .attr('stroke', 'rgba(255,255,255,0.3)')
      .attr('stroke-width', 1)
      .style('cursor', 'pointer');

    bars.transition()
      .duration(800)
      .delay((d, i) => i * 40)
      .attr('width', d => x(d.count));

    const labels = svg.selectAll('.breakdown-label')
      .data(data)
      .join('text')
      .attr('class', 'breakdown-label')
      .attr('x', 5)
      .attr('y', d => y(d.country) + y.bandwidth() / 2)
      .attr('dy', '0.35em')
      .style('fill', '#fff')
      .style('font-size', '11px')
      .style('font-weight', 'bold')
      .style('opacity', 0);

    labels.transition()
      .duration(800)
      .delay((d, i) => i * 40 + 400)
      .style('opacity', 1)
      .attr('x', d => x(d.count) + 8)
      .text(d => d.count.toLocaleString());

    bars
      .on('mouseover', function(event, d) {
        d3.select(this)
          .transition()
          .duration(200)
          .attr('opacity', 0.8);
        
        showTooltip(event, d, true);
      })
      .on('mouseout', function() {
        d3.select(this)
          .transition()
          .duration(200)
          .attr('opacity', 1);
        
        hideTooltip();
      })
      .on('mousemove', function(event) {
        updateTooltipPosition(event);
      });

    svg.append('text')
      .attr('text-anchor', 'middle')
      .attr('x', innerWidth / 2)
      .attr('y', innerHeight + 40)
      .text('Número de Bilionários')
      .style('fill', '#e0e0e0')
      .style('font-size', '14px');
  }

  function showTooltip(event, d, isBreakdown) {
    if (!tooltip) return;
    
    let content;
    if (isBreakdown) {
      content = `
        <div style="background: rgba(0,0,0,0.9); padding: 12px; border-radius: 8px; color: white; font-size: 13px; box-shadow: 0 4px 12px rgba(0,0,0,0.3); border: 1px solid ${genderConfig[selectedGender]?.color};">
          <div style="font-weight: bold; margin-bottom: 8px; color: ${genderConfig[selectedGender]?.color};">${d.country}</div>
          <div><strong>Bilionários ${genderConfig[selectedGender]?.label}s:</strong> ${d.count.toLocaleString()}</div>
        </div>
      `;
    } else {
      content = `
        <div style="background: rgba(0,0,0,0.9); padding: 12px; border-radius: 8px; color: white; font-size: 13px; box-shadow: 0 4px 12px rgba(0,0,0,0.3); border: 1px solid ${d.config.color};">
          <div style="font-weight: bold; margin-bottom: 8px; color: ${d.config.color};">${d.config.icon} ${d.config.label}</div>
          <div><strong>Quantidade:</strong> ${d.count.toLocaleString()}</div>
          <div><strong>Porcentagem:</strong> ${d.percentage}%</div>
          ${allData && allData.length > 0 ? '<div style="margin-top: 8px; font-size: 11px; color: #ccc;">Clique para ver por país</div>' : ''}
        </div>
      `;
    }
    
    tooltip.style('opacity', 1).html(content);
    updateTooltipPosition(event);
  }

  function updateTooltipPosition(event) {
    if (!tooltip) return;
    tooltip
      .style('left', (event.pageX + 10) + 'px')
      .style('top', (event.pageY - 10) + 'px');
  }

  function hideTooltip() {
    if (!tooltip) return;
    tooltip.style('opacity', 0);
  }

  function goBack() {
    showBreakdown = false;
    selectedGender = null;
    breakdownData = [];
    drawChart();
  }
  
  function updateDimensions() {
    if (svgElement && svgElement.parentElement) {
      width = svgElement.parentElement.clientWidth > 0 ? svgElement.parentElement.clientWidth : 550;
      height = svgElement.parentElement.clientHeight > 0 ? svgElement.parentElement.clientHeight : 380;
      if (width < 300) width = 300; 
      if (height < 250) height = 250;
      innerWidth = width - margin.left - margin.right;
      innerHeight = height - margin.top - margin.bottom;
      drawChart();
    }
  }

  onMount(() => {
    tooltip = d3.select('body').append('div')
      .style('position', 'absolute')
      .style('opacity', 0)
      .style('pointer-events', 'none')
      .style('z-index', 1000);
    
    updateDimensions();
    window.addEventListener('resize', updateDimensions);
    return () => {
      window.removeEventListener('resize', updateDimensions);
      if (tooltip) tooltip.remove();
    };
  });

  afterUpdate(() => {
    drawChart();
  });
</script>

<div class="chart-wrapper">
  {#if showBreakdown}
    <div class="back-controls">
      <button class="back-button" on:click={goBack}>
        ← Voltar para Visão Geral
      </button>
    </div>
  {/if}

  <div class="chart-container" bind:clientWidth={width} bind:clientHeight={height}>
    <svg bind:this={svgElement}></svg>
  </div>

  {#if !showBreakdown && dataWithPercentage.length > 0}
    <div class="summary">
      <div class="summary-item">
        <span class="summary-label">Total de Bilionários:</span>
        <span class="summary-value">{totalCount.toLocaleString()}</span>
      </div>
      {#each dataWithPercentage as item}
        <div class="summary-item">
          <span class="summary-icon" style="color: {item.config.color}">{item.config.icon}</span>
          <span class="summary-label">{item.config.label}:</span>
          <span class="summary-value">{item.count.toLocaleString()} ({item.percentage}%)</span>
        </div>
      {/each}
    </div>
  {/if}
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

  .back-controls {
    margin-bottom: 15px;
  }

  .back-button {
    background: rgba(78, 205, 196, 0.2);
    border: 1px solid #4ecdc4;
    color: #4ecdc4;
    padding: 8px 16px;
    border-radius: 6px;
    cursor: pointer;
    font-size: 13px;
    transition: all 0.3s ease;
  }

  .back-button:hover {
    background: rgba(78, 205, 196, 0.3);
    transform: translateX(-2px);
  }

  .chart-container {
    width: 100%;
    height: 380px;
    display: flex;
    justify-content: center;
    align-items: center;
  }

  svg {
    max-width: 100%;
    max-height: 100%;
  }

  .summary {
    margin-top: 15px;
    padding-top: 15px;
    border-top: 1px solid rgba(255, 255, 255, 0.1);
    display: flex;
    flex-wrap: wrap;
    gap: 15px;
    justify-content: center;
  }

  .summary-item {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 8px 12px;
    background: rgba(255, 255, 255, 0.05);
    border-radius: 8px;
    font-size: 13px;
  }

  .summary-icon {
    font-size: 16px;
    font-weight: bold;
  }

  .summary-label {
    color: #b0b0b0;
  }

  .summary-value {
    color: #e0e0e0;
    font-weight: 600;
  }

  @media (max-width: 768px) {
    .chart-wrapper {
      padding: 15px;
    }
    
    .chart-container {
      height: 320px;
    }
    
    .summary {
      flex-direction: column;
      gap: 10px;
    }
    
    .summary-item {
      justify-content: center;
    }
  }
</style> 