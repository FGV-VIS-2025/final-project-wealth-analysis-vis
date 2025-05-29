<script>
  import AgeHistogram from '$lib/components/AgeHistogram.svelte';
  import CountryBarChart from '$lib/components/CountryBarChart.svelte';
  import GenderChart from '$lib/components/GenderChart.svelte';
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import { base } from '$app/paths';

  let allData = [];
  let ageData = [];
  let countryData = [];
  let genderData = [];

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

</script>

<svelte:head>
  <title>Billionaires Data Story</title>
</svelte:head>

<!-- Navegação simples -->
<nav style="padding: 20px; border-bottom: 1px solid #eee; margin-bottom: 20px;">
  <div style="max-width: 1200px; margin: 0 auto; display: flex; justify-content: space-between; align-items: center;">
    <h1 style="margin: 0; color: #333;">Análise Global de Bilionários</h1>
    <div style="display: flex; gap: 20px;">
      <a href="{base}/" style="text-decoration: none; color: #007bff; font-weight: bold;">Visão Geral</a>
      <a href="{base}/trajetoria" style="text-decoration: none; color: #666;">Trajetória</a>
    </div>
  </div>
</nav>

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
</div>

<style>
  .story-container {
    font-family: 'Arial', sans-serif;
    color: #333;
    max-width: 1200px;
    margin: 0 auto;
    padding: 20px;
  }

  .story-section {
    display: flex;
    align-items: flex-start;
    margin-bottom: 80px;
    min-height: 400px;
    border-bottom: 1px solid #eee;
    padding-bottom: 40px;
  }

  .story-section:last-child {
    border-bottom: none;
    margin-bottom: 40px;
  }

  .text-content {
    flex: 1;
    padding: 20px;
    line-height: 1.6;
  }

  .text-content h2 {
    font-size: 2em;
    color: #1a1a1a;
    margin-bottom: 20px;
  }
  
  .text-content p {
    font-size: 1.1em;
    margin-bottom: 15px;
  }

  .chart-content {
    flex: 1;
    padding: 20px;
    display: flex;
    justify-content: center;
    align-items: center;
    min-height: 300px;
    background-color: #f9f9f9;
    border-radius: 8px;
    box-shadow: 0 2px 4px rgba(0,0,0,0.1);
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
</style>
