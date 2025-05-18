<script>
  import AgeHistogram from '$lib/components/AgeHistogram.svelte';
  import CountryBarChart from '$lib/components/CountryBarChart.svelte';
  import GenderChart from '$lib/components/GenderChart.svelte';
  import SunburstChart from '$lib/components/SunburstChart.svelte';
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import { base } from '$app/paths';

  let allData = [];
  let ageData = [];
  let countryData = [];
  let genderData = [];
  let sunburstData = [];

  onMount(async () => {
    const csvPath = `${base}/Billionaires Statistics Dataset.csv`;
    const rawData = await d3.csv(csvPath);
    
    allData = rawData.map(d => ({
      ...d,
      age: +d.age,
      finalWorth: +d.finalWorth
    }));

    ageData = d3.rollups(allData.filter(d => d.age && d.age > 0), v => v.length, d => Math.floor(d.age / 10) * 10)
                .map(([key, value]) => ({ ageGroup: `${key}-${key+9}`, count: value }))
                .sort((a,b) => parseInt(a.ageGroup) - parseInt(b.ageGroup));

    countryData = d3.rollups(allData.filter(d => d.country), v => v.length, d => d.country)
                    .map(([key, value]) => ({ country: key, count: value }))
                    .sort((a,b) => b.count - a.count)
                    .slice(0, 10);

    genderData = d3.rollups(allData.filter(d => d.gender), v => v.length, d => d.gender)
                   .map(([key, value]) => ({ gender: key || 'Unknown', count: value }));
  });

  $: if (allData.length > 0) {
    const continent_map = {
      'United States': 'North America',
      'China': 'Asia',
      'India': 'Asia',
      'Germany': 'Europe',
      'France': 'Europe',
      'United Kingdom': 'Europe',
      'Russia': 'Europe',
      'Brazil': 'South America',
      'Canada': 'North America',
      'Australia': 'Oceania'
    };

    sunburstData = allData.map(d => ({
      Region: continent_map[d.country] || 'Other',
      Country: d.country,
      Industry: d.industries || d.industry || 'Unknown',
      NetWorth: d.finalWorth
    }));
    console.log('sunburstData', sunburstData);
  }

</script>

<svelte:head>
  <title>Billionaires Data Story</title>
</svelte:head>

<div class="story-container">
  <section class="story-section">
    <div class="text-content left">
      <h2>Chapter 1: The Age of Wealth</h2>
      <p>
        The distribution of billionaires across different age groups reveals interesting insights
        into when wealth accumulation peaks. This histogram illustrates the number of billionaires
        within various age brackets.
      </p>
      <p>
        We can observe patterns such as the concentration of billionaires in certain age ranges,
        potentially indicating the prime years for entrepreneurial success or inheritance.
      </p>
    </div>
    <div class="chart-content right">
      {#if ageData.length > 0}
        <AgeHistogram data={ageData} />
      {:else}
        <p>Loading age data...</p>
      {/if}
    </div>
  </section>

  <section class="story-section">
    <div class="chart-content left">
      {#if countryData.length > 0}
        <CountryBarChart data={countryData} />
      {:else}
        <p>Loading country data...</p>
      {/if}
    </div>
    <div class="text-content right">
      <h2>Chapter 2: Global Distribution of Billionaires</h2>
      <p>
        Where in the world do billionaires reside? This chart showcases the top countries
        with the highest number of billionaires. It provides a geographical perspective on wealth concentration.
      </p>
      <p>
        Notice the dominance of certain nations and consider the economic and social factors
        that might contribute to these distributions.
      </p>
    </div>
  </section>

  <section class="story-section">
    <div class="text-content left">
      <h2>Chapter 3: Gender and Billionaires</h2>
      <p>
        This section explores the representation of different genders within the billionaire population.
        The chart highlights the number of billionaires identifying with various gender categories.
      </p>
      <p>
        This data can spark discussions about gender disparity in wealth and the factors
        influencing success for different genders in the business world.
      </p>
    </div>
    <div class="chart-content right">
      {#if genderData.length > 0}
        <GenderChart data={genderData} />
      {:else}
        <p>Loading gender data...</p>
      {/if}
    </div>
  </section>

  <!-- Nova seção: Sunburst -->
  <section class="story-section">
    <div class="text-content left">
      <h2>Distribuição Global do Patrimônio de Bilionários</h2>
      <p>
        O gráfico ao lado mostra como o patrimônio líquido dos bilionários está distribuído hierarquicamente por região, país e setor de atuação. 
        Setores como tecnologia, manufatura e finanças concentram a maior parte da riqueza, especialmente em países como Estados Unidos e China, que oferecem ambientes favoráveis à inovação, acesso a capital e grandes mercados consumidores.
      </p>
      <p>
        Essa visualização permite entender como fatores econômicos, geográficos e setoriais influenciam a concentração de riqueza global.
      </p>
    </div>
    <div class="chart-content right">
      {#if sunburstData.length > 0}
        <SunburstChart data={sunburstData} />
      {:else}
        <p>Carregando dados do sunburst...</p>
      {/if}
    </div>
  </section>
</div>

<style>
  :global(html) {
    font-family: 'Inter', Arial, sans-serif;
    background: #f4f6fa;
    color: #23272f;
  }

  .story-container {
    max-width: 1200px;
    margin: 0 auto;
    padding: 32px 16px;
  }

  .story-section {
    display: flex;
    align-items: stretch;
    margin-bottom: 56px;
    min-height: 400px;
    background: #fff;
    border-radius: 18px;
    box-shadow: 0 4px 24px rgba(44,62,80,0.08), 0 1.5px 4px rgba(44,62,80,0.04);
    overflow: hidden;
    transition: box-shadow 0.2s;
    border: 1px solid #e3e8ee;
  }

  .story-section:hover {
    box-shadow: 0 8px 32px rgba(44,62,80,0.13), 0 2px 8px rgba(44,62,80,0.07);
  }

  .story-section:last-child {
    border-bottom: none;
    margin-bottom: 32px;
  }

  .text-content {
    flex: 1;
    padding: 40px 32px;
    line-height: 1.7;
    display: flex;
    flex-direction: column;
    justify-content: center;
  }

  .text-content h2 {
    font-size: 2.2em;
    color: #1a1a1a;
    margin-bottom: 18px;
    font-weight: 700;
    letter-spacing: -1px;
  }
  
  .text-content p {
    font-size: 1.13em;
    margin-bottom: 14px;
    color: #3a3a3a;
  }

  .chart-content {
    flex: 1;
    padding: 32px 24px;
    display: flex;
    justify-content: center;
    align-items: center;
    min-height: 320px;
    background: linear-gradient(120deg, #f7fafc 60%, #e9f3ff 100%);
    border-radius: 0 18px 18px 0;
    box-shadow: none;
    transition: background 0.3s;
  }

  .story-section:nth-child(even) .chart-content {
    border-radius: 18px 0 0 18px;
  }

  .text-content.left {
    padding-right: 40px;
  }

  .text-content.right {
    padding-left: 40px;
  }

  :global(.chart-content > *) {
    width: 100%;
    height: 100%;
  }

  @media (max-width: 900px) {
    .story-section {
      flex-direction: column;
      min-height: unset;
    }
    .chart-content, .text-content {
      padding: 24px 12px;
    }
    .chart-content {
      border-radius: 0 0 18px 18px !important;
      min-height: 220px;
    }
    .text-content {
      border-radius: 18px 18px 0 0;
      padding-bottom: 0;
    }
    .text-content.left, .text-content.right {
      padding: 0 0 12px 0;
    }
  }

  @media (max-width: 600px) {
    .story-container {
      padding: 8px 2px;
    }
    .story-section {
      margin-bottom: 24px;
    }
    .text-content h2 {
      font-size: 1.3em;
    }
    .text-content p {
      font-size: 1em;
    }
  }
</style>
