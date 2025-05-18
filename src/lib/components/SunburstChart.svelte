<script>
    import { onMount } from 'svelte';
  
    export let data = [];
  
    let chartDiv;
    let Plotly; // Não importa no topo!
  
    // Função para transformar os dados no formato do Plotly Sunburst
    function prepareSunburstData(data) {
      // data: [{Region, Country, Industry, NetWorth}]
      const labels = [];
      const parents = [];
      const values = [];
  
      // Adiciona regiões
      const regions = [...new Set(data.map(d => d.Region))];
      regions.forEach(region => {
        labels.push(region);
        parents.push('');
        values.push(
          data.filter(d => d.Region === region).reduce((sum, d) => sum + d.NetWorth, 0)
        );
      });
  
      // Adiciona países
      const countries = [...new Set(data.map(d => d.Country))];
      countries.forEach(country => {
        const region = data.find(d => d.Country === country).Region;
        labels.push(country);
        parents.push(region);
        values.push(
          data.filter(d => d.Country === country).reduce((sum, d) => sum + d.NetWorth, 0)
        );
      });
  
      // Adiciona indústrias
      data.forEach(d => {
        labels.push(d.Industry);
        parents.push(d.Country);
        values.push(d.NetWorth);
      });
  
      return { labels, parents, values };
    }
  
    onMount(async () => {
      Plotly = (await import('plotly.js-dist-min')).default;
  
      if (data.length > 0) {
        const { labels, parents, values } = prepareSunburstData(data);
  
        Plotly.newPlot(chartDiv, [{
          type: 'sunburst',
          labels,
          parents,
          values,
          branchvalues: 'total',
          maxdepth: 3
        }], {
          title: 'Distribuição Hierárquica do Patrimônio Líquido de Bilionários',
          margin: { l: 0, r: 0, b: 0, t: 40 }
        }, {responsive: true});
      }
    });
  
    $: if (data.length > 0 && chartDiv && Plotly) {
      const { labels, parents, values } = prepareSunburstData(data);
      Plotly.react(chartDiv, [{
        type: 'sunburst',
        labels,
        parents,
        values,
        branchvalues: 'total',
        maxdepth: 3
      }], {
        title: 'Distribuição Hierárquica do Patrimônio Líquido de Bilionários',
        margin: { l: 0, r: 0, b: 0, t: 40 }
      }, {responsive: true});
    }
  </script>
  
  <div bind:this={chartDiv} style="width:100%;height:500px;"></div>