<script>
  export let data = []; 
  import { onMount, afterUpdate } from 'svelte';
  import * as d3 from 'd3';

  let svgElement, tooltip;
  
  const margin = { top: 60, right: 40, bottom: 80, left: 80 };
  let width = 600, height = 380;
  let innerWidth = width - margin.left - margin.right;
  let innerHeight = height - margin.top - margin.bottom;

  const ageGroupOrder = ['10-19', '20-29', '30-39', '40-49', '50-59', '60-69', '70-79', '80-89', '90-99', '100-109'];
  const uniformColor = '#6366f1';

  $: processedData = data
    .filter(d => d.ageGroup && d.count > 0)
    .sort((a, b) => ageGroupOrder.indexOf(a.ageGroup) - ageGroupOrder.indexOf(b.ageGroup))
    .map(d => ({ ...d, color: uniformColor }));

  $: totalBillionaires = d3.sum(processedData, d => d.count);
  $: dataWithStats = processedData.map(d => ({
    ...d,
    percentage: ((d.count / totalBillionaires) * 100).toFixed(1)
  }));

  function drawChart() {
    if (!svgElement || dataWithStats.length === 0) return;

    d3.select(svgElement).selectAll('*').remove();

    const svg = d3.select(svgElement).attr('width', width).attr('height', height);

    svg.append('text')
      .attr('x', width / 2).attr('y', 30).attr('text-anchor', 'middle')
      .style('font-size', '18px').style('font-weight', 'bold').style('fill', '#e0e0e0')
      .text('Distribuição de Idades dos Bilionários');

    const chartGroup = svg.append('g').attr('transform', `translate(${margin.left},${margin.top})`);

    const x = d3.scaleBand().range([0, innerWidth]).domain(dataWithStats.map(d => d.ageGroup)).padding(0.2);
    const y = d3.scaleLinear().domain([0, d3.max(dataWithStats, d => d.count) || 10]).range([innerHeight, 0]);

    chartGroup.append('g').attr('transform', `translate(0,${innerHeight})`).call(d3.axisBottom(x))
      .selectAll('text').style('fill', '#e0e0e0').style('font-size', '12px');

    chartGroup.append('g').call(d3.axisLeft(y).tickFormat(d3.format('d')))
      .selectAll('text').style('fill', '#e0e0e0');

    const bars = chartGroup.selectAll('.age-bar').data(dataWithStats).join('rect')
      .attr('class', 'age-bar')
      .attr('x', d => x(d.ageGroup)).attr('y', innerHeight).attr('width', x.bandwidth()).attr('height', 0)
      .attr('fill', d => d.color).attr('stroke', 'rgba(255,255,255,0.3)').attr('stroke-width', 1)
      .style('cursor', 'pointer');

    bars.transition().duration(800).delay((d, i) => i * 80)
      .attr('y', d => y(d.count)).attr('height', d => innerHeight - y(d.count));

    const labels = chartGroup.selectAll('.value-label').data(dataWithStats).join('text')
      .attr('class', 'value-label')
      .attr('x', d => x(d.ageGroup) + x.bandwidth() / 2).attr('y', innerHeight).attr('text-anchor', 'middle')
      .style('font-weight', 'bold').style('font-size', '12px').style('opacity', 0);

    labels.transition().duration(800).delay((d, i) => i * 80 + 400).style('opacity', 1)
      .attr('y', d => {
        const barHeight = innerHeight - y(d.count);
        return barHeight < 25 ? y(d.count) - 8 : y(d.count) + 16;
      })
      .style('fill', d => {
        const barHeight = innerHeight - y(d.count);
        return barHeight < 25 ? '#e0e0e0' : '#ffffff';
      })
      .text(d => d.count.toLocaleString());

    const percentLabels = chartGroup.selectAll('.percent-label')
      .data(dataWithStats.filter(d => (innerHeight - y(d.count)) >= 40))
      .join('text').attr('class', 'percent-label')
      .attr('x', d => x(d.ageGroup) + x.bandwidth() / 2).attr('y', d => y(d.count) + 32).attr('text-anchor', 'middle')
      .style('fill', 'rgba(255,255,255,0.8)').style('font-weight', 'bold').style('font-size', '10px').style('opacity', 0);

    percentLabels.transition().duration(800).delay((d, i) => i * 80 + 600).style('opacity', 1).text(d => `${d.percentage}%`);

    bars.on('mouseover', function(event, d) {
        d3.select(this).transition().duration(200).attr('opacity', 0.8).attr('stroke-width', 3);
        showTooltip(event, d);
      })
      .on('mouseout', function() {
        d3.select(this).transition().duration(200).attr('opacity', 1).attr('stroke-width', 1);
        hideTooltip();
      })
      .on('mousemove', function(event) { updateTooltipPosition(event); });

    chartGroup.append('text').attr('text-anchor', 'middle').attr('x', innerWidth / 2).attr('y', innerHeight + 40)
      .text('Faixa Etária').style('fill', '#e0e0e0').style('font-size', '14px');
    chartGroup.append('text').attr('text-anchor', 'middle').attr('transform', 'rotate(-90)')
      .attr('y', -50).attr('x', -innerHeight / 2).text('Número de Bilionários').style('fill', '#e0e0e0').style('font-size', '14px');
  }

  function showTooltip(event, d) {
    if (!tooltip) return;
    const content = `<div style="background: rgba(0,0,0,0.9); padding: 12px; border-radius: 8px; color: white; font-size: 13px; box-shadow: 0 4px 12px rgba(0,0,0,0.3); border: 1px solid ${d.color};">
      <div style="font-weight: bold; margin-bottom: 8px; color: ${d.color};">Faixa: ${d.ageGroup} anos</div>
      <div><strong>Bilionários:</strong> ${d.count.toLocaleString()}</div>
      <div><strong>Porcentagem:</strong> ${d.percentage}%</div></div>`;
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

  afterUpdate(() => { drawChart(); });
</script>

<div class="chart-wrapper">
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
  }
</style> 