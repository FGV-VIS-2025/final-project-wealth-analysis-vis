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
  let breakdownType = 'country'; // 'country' ou 'selfmade'
  
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

  const selfMadeConfig = {
    true: {
      color: '#10b981',
      label: 'Self-Made',
      lightColor: '#34d399'
    },
    false: {
      color: '#8b5cf6',
      label: 'Não Self-Made',
      lightColor: '#a78bfa'
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
    
    const countryStats = {};
    
    // Primeiro, contar todos os bilionários por país
    allData.forEach(person => {
      const country = person.country;
      if (country && country !== 'Unknown' && country.trim() !== '' && country.toLowerCase() !== 'unknown') {
        if (!countryStats[country]) {
          countryStats[country] = { 
            total: 0, 
            byGender: { M: 0, F: 0 }
          };
        }
        countryStats[country].total++;
        
        if (person.gender === 'M') {
          countryStats[country].byGender.M++;
        } else if (person.gender === 'F') {
          countryStats[country].byGender.F++;
        }
      }
    });

    // Usar EXATAMENTE os mesmos critérios para ambos os gêneros
    const results = Object.entries(countryStats)
      .filter(([country, stats]) => {
        // MESMA REGRA: pelo menos 10 bilionários E ambos os gêneros representados
        return stats.total >= 10 && stats.byGender.M > 0 && stats.byGender.F > 0;
      })
      .map(([country, stats]) => {
        const genderCount = stats.byGender[gender] || 0;
        const percentage = stats.total > 0 ? (genderCount / stats.total) * 100 : 0;
        return { 
          country, 
          count: genderCount, 
          total: stats.total,
          percentage: percentage.toFixed(1),
          gender 
        };
      })
      .sort((a, b) => parseFloat(b.percentage) - parseFloat(a.percentage)) // MAIOR para MENOR - países com MAIS do gênero específico
      .slice(0, limit);

    return results;
  }

  function getSelfMadeByGender(gender) {
    if (!allData || allData.length === 0) return [];
    
    const selfMadeCounts = {};
    
    allData.forEach(person => {
      if (person.gender === gender) {
        const isSelfMade = person.selfMade === 'TRUE' || person.selfMade === 'True' || person.selfMade === 'true' || person.selfMade === true;
        const key = isSelfMade ? 'true' : 'false';
        selfMadeCounts[key] = (selfMadeCounts[key] || 0) + 1;
      }
    });

    return Object.entries(selfMadeCounts)
      .map(([isSelfMade, count]) => ({ 
        isSelfMade: isSelfMade === 'true',
        count, 
        gender,
        label: isSelfMade === 'true' ? 'Self-Made' : 'Não Self-Made'
      }))
      .sort((a, b) => b.count - a.count);
  }

  function drawChart() {
    if (!svgElement) return;
    
    // Se não temos dados principais mas temos breakdown, usar breakdown
    if (!dataWithPercentage || dataWithPercentage.length === 0) {
      if (!showBreakdown || !breakdownData || breakdownData.length === 0) {
        return;
      }
    }

    // Limpar o SVG completamente antes de desenhar
    d3.select(svgElement).selectAll('*').remove();

    // Garantir que o SVG tenha as dimensões corretas
    const svg = d3.select(svgElement)
      .attr('width', width)
      .attr('height', height);

    let titleText = 'Distribuição por Gênero';
    if (showBreakdown) {
      if (breakdownType === 'country') {
        titleText = `Porcentagem de Bilionários ${selectedGender === 'M' ? 'Masculinos' : 'Femininos'} por País`;
      } else {
        titleText = `${selectedGender === 'M' ? 'Bilionários' : 'Bilionárias'} - Self-Made vs Não Self-Made`;
      }
    }

    svg.append('text')
      .attr('x', width / 2)
      .attr('y', 25)
      .attr('text-anchor', 'middle')
      .style('font-size', '18px')
      .style('font-weight', 'bold')
      .style('fill', '#e0e0e0')
      .text(titleText);

    const chartGroup = svg.append('g')
      .attr('transform', `translate(${margin.left},${margin.top})`);

    const currentData = showBreakdown ? breakdownData : dataWithPercentage;

    if (showBreakdown) {
      if (breakdownType === 'country') {
        drawBreakdownChart(chartGroup, currentData);
      } else {
        drawSelfMadeChart(chartGroup, currentData);
      }
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

    const maxValue = d3.max(data, d => d.count) || 10;
    const y = d3.scaleLinear()
      .domain([0, maxValue])
      .range([innerHeight, 0]);

    // Gerar ticks mais proporcionais
    const tickCount = Math.min(5, Math.floor(maxValue / 500) + 1);
    const tickValues = y.ticks(tickCount);
    
    // Sempre incluir o valor máximo se não estiver muito próximo do último tick
    if (!tickValues.includes(maxValue) && (maxValue - tickValues[tickValues.length - 1]) > maxValue * 0.1) {
      tickValues.push(maxValue);
    }

    const yAxis = svg.append('g')
      .call(d3.axisLeft(y)
        .tickFormat(d3.format('d'))
        .tickValues(tickValues));
    
    yAxis.selectAll('text')
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
      .style('cursor', 'pointer')
      .style('pointer-events', 'none');

    bars.transition()
      .duration(800)
      .delay((d, i) => i * 150)
      .attr('y', d => y(d.count))
      .attr('height', d => innerHeight - y(d.count))
      .on('end', function() {
        d3.select(this).style('pointer-events', 'auto');
      });

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
        // Verificar se há transição em andamento
        const bar = d3.select(this);
        if (!bar.transition().active()) {
          bar.transition()
            .duration(200)
            .attr('opacity', 0.8)
            .attr('stroke-width', 3);
          
          showTooltip(event, d, false);
        }
      })
      .on('mouseout', function() {
        // Verificar se há transição em andamento
        const bar = d3.select(this);
        if (!bar.transition().active()) {
          bar.transition()
            .duration(200)
            .attr('opacity', 1)
            .attr('stroke-width', 2);
          
          hideTooltip();
        }
      })
      .on('mousemove', function(event) {
        updateTooltipPosition(event);
      })
      .on('click', function(event, d) {
        if (allData && allData.length > 0) {
          showCountryBreakdown(d.gender);
        }
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

    // Usar porcentagem como valor principal
    const maxValue = Math.max(100, d3.max(data, d => parseFloat(d.percentage)) || 50);
    const x = d3.scaleLinear()
      .domain([0, maxValue])
      .range([0, innerWidth]);

    const xAxis = svg.append('g')
      .attr('transform', `translate(0,${innerHeight})`)
      .call(d3.axisBottom(x)
        .tickFormat(d => `${d}%`)
        .ticks(6)
        .tickValues([0, 20, 40, 60, 80, 100]));
    
    xAxis.selectAll('text')
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
      .style('cursor', 'pointer')
      .style('pointer-events', 'none');

    bars.transition()
      .duration(800)
      .delay((d, i) => i * 40)
      .attr('width', d => x(parseFloat(d.percentage)))
      .on('end', function() {
        d3.select(this).style('pointer-events', 'auto');
      });

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
      .attr('x', d => x(parseFloat(d.percentage)) + 8)
      .text(d => `${d.percentage}%`);

    bars
      .on('mouseover', function(event, d) {
        // Verificar se há transição em andamento
        const bar = d3.select(this);
        if (!bar.transition().active()) {
          bar.transition()
            .duration(200)
            .attr('opacity', 0.8);
          
          showTooltip(event, d, true);
        }
      })
      .on('mouseout', function() {
        // Verificar se há transição em andamento
        const bar = d3.select(this);
        if (!bar.transition().active()) {
          bar.transition()
            .duration(200)
            .attr('opacity', 1);
          
          hideTooltip();
        }
      })
      .on('mousemove', function(event) {
        updateTooltipPosition(event);
      });

    svg.append('text')
      .attr('text-anchor', 'middle')
      .attr('transform', 'rotate(-90)')
      .attr('y', -60)
      .attr('x', -innerHeight / 2)
      .text('País')
      .style('fill', '#e0e0e0')
      .style('font-size', '14px');

    svg.append('text')
      .attr('text-anchor', 'middle')
      .attr('x', innerWidth / 2)
      .attr('y', innerHeight + 40)
      .text('Porcentagem (%)')
      .style('fill', '#e0e0e0')
      .style('font-size', '14px');
  }

  function drawSelfMadeChart(svg, data) {
    const x = d3.scaleBand()
      .range([0, innerWidth])
      .domain(data.map(d => d.label))
      .padding(0.3);

    svg.append('g')
      .attr('transform', `translate(0,${innerHeight})`)
      .call(d3.axisBottom(x))
      .selectAll('text')
        .style('fill', '#e0e0e0')
        .style('font-size', '12px')
        .style('text-anchor', 'middle');

    const maxValue = d3.max(data, d => d.count) || 10;
    const y = d3.scaleLinear()
      .domain([0, maxValue])
      .range([innerHeight, 0]);

    const yAxis = svg.append('g')
      .call(d3.axisLeft(y)
        .tickFormat(d3.format('d'))
        .ticks(Math.min(6, maxValue))
        .tickValues(y.ticks(Math.min(6, maxValue)).concat(maxValue)));
    
    yAxis.selectAll('text')
      .style('fill', '#e0e0e0');

    const bars = svg.selectAll('.selfmade-bar')
      .data(data)
      .join('rect')
      .attr('class', 'selfmade-bar')
      .attr('x', d => x(d.label))
      .attr('y', innerHeight)
      .attr('width', x.bandwidth())
      .attr('height', 0)
      .attr('fill', d => selfMadeConfig[d.isSelfMade]?.color || '#6b7280')
      .attr('stroke', 'rgba(255,255,255,0.3)')
      .attr('stroke-width', 2)
      .attr('rx', 6).attr('ry', 6)
      .style('cursor', 'pointer')
      .style('pointer-events', 'none');

    bars.transition()
      .duration(800)
      .delay((d, i) => i * 200)
      .attr('y', d => y(d.count))
      .attr('height', d => innerHeight - y(d.count))
      .on('end', function() {
        d3.select(this).style('pointer-events', 'auto');
      });

    const labels = svg.selectAll('.selfmade-label')
      .data(data)
      .join('text')
      .attr('class', 'selfmade-label')
      .attr('x', d => x(d.label) + x.bandwidth() / 2)
      .attr('y', innerHeight)
      .attr('text-anchor', 'middle')
      .style('fill', '#fff')
      .style('font-weight', 'bold')
      .style('font-size', '14px')
      .style('opacity', 0);

    labels.transition()
      .duration(800)
      .delay((d, i) => i * 200 + 400)
      .attr('y', d => y(d.count) - 10)
      .style('opacity', 1)
      .text(d => d.count.toLocaleString());

    const total = d3.sum(data, d => d.count);
    const percentLabels = svg.selectAll('.selfmade-percent-label')
      .data(data)
      .join('text')
      .attr('class', 'selfmade-percent-label')
      .attr('x', d => x(d.label) + x.bandwidth() / 2)
      .attr('y', d => y(d.count) + 20)
      .attr('text-anchor', 'middle')
      .style('fill', '#000')
      .style('font-weight', 'bold')
      .style('font-size', '12px')
      .style('opacity', 0);

    percentLabels.transition()
      .duration(800)
      .delay((d, i) => i * 200 + 600)
      .style('opacity', 1)
      .text(d => `${((d.count / total) * 100).toFixed(1)}%`);

    bars
      .on('mouseover', function(event, d) {
        // Verificar se há transição em andamento
        const bar = d3.select(this);
        if (!bar.transition().active()) {
          bar.transition()
            .duration(200)
            .attr('opacity', 0.8)
            .attr('stroke-width', 3);
          
          showTooltip(event, d, false, true);
        }
      })
      .on('mouseout', function() {
        const bar = d3.select(this);
        if (!bar.transition().active()) {
          bar.transition()
            .duration(200)
            .attr('opacity', 1)
            .attr('stroke-width', 2);
          
          hideTooltip();
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

  function showTooltip(event, d, isBreakdown, isSelfMade = false) {
    if (!tooltip) return;
    
    let content;
    if (isSelfMade) {
      const total = d3.sum(breakdownData, item => item.count);
      content = `
        <div style="background: rgba(0,0,0,0.9); padding: 12px; border-radius: 8px; color: white; font-size: 13px; box-shadow: 0 4px 12px rgba(0,0,0,0.3); border: 1px solid ${selfMadeConfig[d.isSelfMade]?.color};">
          <div style="font-weight: bold; margin-bottom: 8px; color: ${selfMadeConfig[d.isSelfMade]?.color};">${genderConfig[selectedGender]?.icon} ${d.label}</div>
          <div><strong>Quantidade:</strong> ${d.count.toLocaleString()}</div>
          <div><strong>Porcentagem:</strong> ${((d.count / total) * 100).toFixed(1)}%</div>
        </div>
      `;
    } else if (isBreakdown) {
      content = `
        <div style="background: rgba(0,0,0,0.9); padding: 12px; border-radius: 8px; color: white; font-size: 13px; box-shadow: 0 4px 12px rgba(0,0,0,0.3); border: 1px solid ${genderConfig[selectedGender]?.color};">
          <div style="font-weight: bold; margin-bottom: 8px; color: ${genderConfig[selectedGender]?.color};">${d.country}</div>
          <div><strong>Bilionários ${genderConfig[selectedGender]?.label}s:</strong> ${d.count} de ${d.total}</div>
          <div><strong>Porcentagem:</strong> ${d.percentage}%</div>
          <div style="margin-top: 4px; font-size: 11px; color: #ccc;">
            ${parseFloat(d.percentage) > 50 ? 'Representação acima da média' : 
              parseFloat(d.percentage) > 20 ? 'Representação moderada' : 'Representação baixa'}
          </div>
        </div>
      `;
    } else {
      content = `
        <div style="background: rgba(0,0,0,0.9); padding: 12px; border-radius: 8px; color: white; font-size: 13px; box-shadow: 0 4px 12px rgba(0,0,0,0.3); border: 1px solid ${d.config.color};">
          <div style="font-weight: bold; margin-bottom: 8px; color: ${d.config.color};">${d.config.icon} ${d.config.label}</div>
          <div><strong>Quantidade:</strong> ${d.count.toLocaleString()}</div>
          <div><strong>Porcentagem:</strong> ${d.percentage}%</div>
          ${allData && allData.length > 0 ? '<div style="margin-top: 8px; font-size: 11px; color: #ccc;">Clique para ver detalhes</div>' : ''}
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
    breakdownType = 'country';
    
    // Limpar completamente o SVG e forçar recálculo total
    if (svgElement) {
      d3.select(svgElement).selectAll('*').remove();
    }
    
    // Aguardar um pouco para o DOM se estabilizar e recalcular tudo
    setTimeout(() => {
      if (svgElement && svgElement.parentElement) {
        // Forçar recálculo completo das dimensões
        const parentWidth = svgElement.parentElement.clientWidth;
        const parentHeight = svgElement.parentElement.clientHeight;
        
        width = parentWidth > 0 ? parentWidth : 550;
        height = parentHeight > 0 ? parentHeight : 380;
        
        if (width < 300) width = 300; 
        if (height < 250) height = 250;
        
        innerWidth = width - margin.left - margin.right;
        innerHeight = height - margin.top - margin.bottom;
        
        // Redefinir atributos do SVG
        d3.select(svgElement)
          .attr('width', width)
          .attr('height', height);
        
        // Redesenhar completamente
        drawChart();
      }
    }, 150);
  }

  function showCountryBreakdown(gender) {
    if (allData && allData.length > 0) {
      selectedGender = gender;
      breakdownData = getTopCountriesByGender(gender);
      breakdownType = 'country';
      showBreakdown = true;
      setTimeout(() => {
        updateDimensions();
      }, 100);
    }
  }

  function showSelfMadeBreakdown(gender) {
    if (allData && allData.length > 0) {
      selectedGender = gender;
      breakdownData = getSelfMadeByGender(gender);
      breakdownType = 'selfmade';
      showBreakdown = true;
      setTimeout(() => {
        updateDimensions();
      }, 100);
    }
  }
  
  function updateDimensions() {
    if (svgElement && svgElement.parentElement) {
      const parentWidth = svgElement.parentElement.clientWidth;
      const parentHeight = svgElement.parentElement.clientHeight;
      
      width = parentWidth > 0 ? parentWidth : 550;
      height = parentHeight > 0 ? parentHeight : 380;
      
      if (width < 300) width = 300; 
      if (height < 250) height = 250;
      
      innerWidth = width - margin.left - margin.right;
      innerHeight = height - margin.top - margin.bottom;
      
      // Garantir que o SVG tenha as dimensões corretas
      if (svgElement) {
        d3.select(svgElement)
          .attr('width', width)
          .attr('height', height);
      }
      
      drawChart();
    }
  }

  onMount(() => {
    tooltip = d3.select('body').append('div')
      .style('position', 'absolute')
      .style('opacity', 0)
      .style('pointer-events', 'none')
      .style('z-index', 1000);
    
    // Aguardar o elemento estar pronto
    setTimeout(() => {
      updateDimensions();
    }, 100);
    
    window.addEventListener('resize', updateDimensions);
    return () => {
      window.removeEventListener('resize', updateDimensions);
      if (tooltip) tooltip.remove();
    };
  });

  afterUpdate(() => {
    // Aguardar a atualização do DOM e verificar se precisamos redesenhar
    setTimeout(() => {
      if (svgElement) {
        const currentWidth = svgElement.clientWidth || 0;
        const currentHeight = svgElement.clientHeight || 0;
        
        // Se as dimensões mudaram ou se o SVG está vazio, redesenhar
        if (currentWidth !== width || currentHeight !== height || !svgElement.hasChildNodes()) {
          updateDimensions();
        }
      }
    }, 50);
  });

  // Reactive statement otimizado para dados principais
  $: if (dataWithPercentage && dataWithPercentage.length > 0 && svgElement && !showBreakdown) {
    setTimeout(() => {
      updateDimensions();
    }, 20);
  }

  // Reactive statement otimizado para breakdown
  $: if (showBreakdown && breakdownData && breakdownData.length > 0 && svgElement) {
    setTimeout(() => {
      updateDimensions();
    }, 20);
  }
</script>

<div class="chart-wrapper">
  {#if showBreakdown}
    <div class="back-controls">
      <button class="back-button" on:click={goBack}>
        ← Voltar para Dist. Gênero
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

    {#if allData && allData.length > 0}
      <div class="breakdown-controls">
        <h4>Explorar por gênero:</h4>
        <div class="control-buttons">
          {#each dataWithPercentage as item}
            <div class="gender-controls">
              <span class="gender-label" style="color: {item.config.color}">
                {item.config.icon} {item.config.label}
              </span>
              <div class="button-group">
                <button 
                  class="breakdown-button country-button" 
                  on:click={() => showCountryBreakdown(item.gender)}
                >
                  Por País
                </button>
                <button 
                  class="breakdown-button selfmade-button" 
                  on:click={() => showSelfMadeBreakdown(item.gender)}
                >
                  Self-Made
                </button>
              </div>
            </div>
          {/each}
        </div>
        <p class="note"><b>Observação:</b> Consideramos "Não Self-Made" aqueles indivíduos cuja maior parte do patrimônio é proveniente de herança.</p>
      </div>
    {/if}
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

  .breakdown-controls {
    margin-top: 20px;
    padding-top: 20px;
    border-top: 1px solid rgba(255, 255, 255, 0.1);
  }

  .breakdown-controls h4 {
    color: #e0e0e0;
    margin-bottom: 15px;
    text-align: center;
    font-size: 16px;
  }

  .control-buttons {
    display: flex;
    gap: 20px;
    justify-content: center;
    flex-wrap: wrap;
  }

  .gender-controls {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 10px;
  }

  .gender-label {
    font-size: 16px;
    font-weight: bold;
    margin-bottom: 5px;
  }

  .button-group {
    display: flex;
    gap: 8px;
  }

  .breakdown-button {
    background: rgba(78, 205, 196, 0.2);
    border: 1px solid #4ecdc4;
    color: #4ecdc4;
    padding: 8px 16px;
    border-radius: 6px;
    cursor: pointer;
    font-size: 13px;
    transition: all 0.3s ease;
    white-space: nowrap;
  }

  .breakdown-button:hover {
    background: rgba(78, 205, 196, 0.3);
    transform: translateY(-2px);
  }

  .selfmade-button {
    background: rgba(16, 185, 129, 0.2);
    border: 1px solid #10b981;
    color: #10b981;
  }

  .selfmade-button:hover {
    background: rgba(16, 185, 129, 0.3);
  }

  .note {
    font-size: 12px;
    color: #b0b0b0;
    text-align: center;
    margin-top: 20px;
    max-width: 500px;
    margin-left: auto;
    margin-right: auto;
    font-style: italic;
    line-height: 1.4;
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

    .control-buttons {
      flex-direction: column;
      gap: 15px;
    }

    .gender-controls {
      width: 100%;
    }

    .button-group {
      justify-content: center;
    }

    .breakdown-button {
      flex: 1;
      max-width: 120px;
    }
  }
</style> 