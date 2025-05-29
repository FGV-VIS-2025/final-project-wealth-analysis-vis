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

  function drawChart() {
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

    svg.selectAll('rect')
      .data(data)
      .join('rect')
      .attr('x', d => x(d.gender))
      .attr('y', d => y(d.count))
      .attr('width', x.bandwidth())
      .attr('height', d => innerHeight - y(d.count))
      .attr('fill', d => {
        if (d.gender === 'M') return '#2563eb'; // Blue for Male
        if (d.gender === 'F') return '#dc2626'; // Red for Female
        return '#6b7280'; // Default color for 'Unknown' or other
      });

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

<div class="chart-container" bind:clientWidth={width} bind:clientHeight={height}>
    <svg bind:this={svgElement}></svg>
</div>

<style>
  .chart-container {
    width: 100%;
    height: 400px;
    display: flex;
    justify-content: center;
    align-items: center;
  }
  svg {
    max-width: 100%;
    max-height: 100%;
  }
</style> 