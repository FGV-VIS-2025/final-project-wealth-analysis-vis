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
      .call(d3.axisLeft(y));

    const x = d3.scaleLinear()
      .domain([0, d3.max(data, d => d.count) || 10])
      .range([0, innerWidth]);

    svg.append('g')
      .attr('transform', `translate(0,${innerHeight})`)
      .call(d3.axisBottom(x))
      .selectAll('text')
      .attr('transform', 'translate(-10,0)rotate(-45)')
      .style('text-anchor', 'end');

    svg.selectAll('rect')
      .data(data)
      .join('rect')
      .attr('y', d => y(d.country))
      .attr('x', 0) 
      .attr('height', y.bandwidth())
      .attr('width', d => x(d.count))
      .attr('fill', '#4CAF50');

    svg.append('text')
        .attr('text-anchor', 'middle')
        .attr('x', innerWidth / 2)
        .attr('y', innerHeight + margin.bottom - 30)
        .text('Number of Billionaires');

    svg.append('text')
        .attr('text-anchor', 'middle')
        .attr('transform', 'rotate(-90)')
        .attr('y', -margin.left + 50) 
        .attr('x', -innerHeight / 2)
        .text('Country');
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

<div class="chart-container" bind:clientWidth={width} bind:clientHeight={height}>
    <svg bind:this={svgElement}></svg>
</div>

<style>
  .chart-container {
    width: 100%;
    height: 450px;
    display: flex;
    justify-content: center;
    align-items: center;
    background: #f7fafc;
    border-radius: 16px;
    box-shadow: 0 2px 12px rgba(44,62,80,0.07);
    transition: box-shadow 0.2s;
  }
  .chart-container:hover {
    box-shadow: 0 6px 24px rgba(44,62,80,0.13);
  }
  svg {
    max-width: 100%;
    max-height: 100%;
    background: transparent;
    border-radius: 12px;
  }
  @media (max-width: 600px) {
    .chart-container {
      height: 220px;
      padding: 0;
    }
  }
</style> 