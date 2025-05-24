<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  
  export let data = [];
  
  let chartDiv;
  let Plotly;
  
  // Function to prepare data for the pie chart
  function preparePieData(data) {
    const selfMadeCount = data.filter(d => d.selfMade === 'TRUE').length;
    const inheritedCount = data.filter(d => d.selfMade === 'FALSE').length;
    
    return [
      {
        values: [selfMadeCount, inheritedCount],
        labels: ['Self-Made', 'Inherited'],
        type: 'pie',
        hole: 0.4,
        marker: {
          colors: ['#2ecc71', '#e74c3c']
        },
        textinfo: 'label+percent',
        textposition: 'outside',
        automargin: true
      }
    ];
  }
  
  onMount(async () => {
    Plotly = (await import('plotly.js-dist-min')).default;
    
    if (data.length > 0) {
      const pieData = preparePieData(data);
      
      Plotly.newPlot(chartDiv, pieData, {
        title: 'Distribution of Billionaires: Self-Made vs Inherited',
        showlegend: true,
        legend: {
          orientation: 'h',
          y: -0.1
        },
      }, {responsive: true});
    }
  });
</script>

<div class="chart-container" bind:this={chartDiv}></div>

<style>
  .chart-container {
    width: 100%;
    height: 500px;
    margin: 20px 0;
  }
</style> 