<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import { base } from '$app/paths';

  let allData = [];
  let currentView = 'wealth';
  let animatedTotal = 0;
  let finalTotal = 0;
  let chartData = [];
  let isLoading = true;
  let hoveredItem = null;

  const views = [
    { key: 'wealth', label: 'Patrimônio por País', type: 'bar', color: '#3B82F6' },
    { key: 'selfmade', label: 'Self-Made por País', type: 'bar', color: '#10B981' },
    { key: 'gender', label: 'Gênero por País', type: 'bar', color: '#F59E0B' },
    { key: 'age', label: 'Idade Média por País', type: 'bar', color: '#EF4444' },
    { key: 'industries', label: 'Top Indústrias por País', type: 'bar', color: '#8B5CF6' }
  ];

  let currentViewIndex = 0;

  onMount(async () => {
    await loadData();
    animateTotal();
    updateChart();
  });

  async function loadData() {
    try {
      const csvPath = `${base}/Billionaires Statistics Dataset.csv`;
      const rawData = await d3.csv(csvPath);
      
      allData = rawData.map(d => ({
        ...d,
        age: +d.age || 0,
        finalWorth: +d.finalWorth || 0,
        selfMade: d.selfMade === 'TRUE' || d.selfMade === 'True' || d.selfMade === 'true' || d.selfMade === true,
        birthYear: +d.birthYear || 0
      })).filter(d => d.finalWorth > 0);

      finalTotal = allData.reduce((sum, d) => sum + d.finalWorth, 0);
      isLoading = false;
    } catch (error) {
      console.error('Erro ao carregar dados:', error);
      isLoading = false;
    }
  }

  function animateTotal() {
    const duration = 2000;
    const steps = 60;
    const increment = finalTotal / steps;
    let current = 0;
    
    const interval = setInterval(() => {
      current += increment;
      if (current >= finalTotal) {
        current = finalTotal;
        clearInterval(interval);
      }
      animatedTotal = Math.floor(current / 1000);
    }, duration / steps);
  }

  function updateChart() {
    if (!allData.length) return;

    switch (currentView) {
      case 'wealth':
        chartData = getWealthByCountry();
        break;
      case 'selfmade':
        chartData = getSelfMadeByCountry();
        break;
      case 'gender':
        chartData = getGenderByCountry();
        break;
      case 'age':
        chartData = getAgeByCountry();
        break;
      case 'industries':
        chartData = getIndustriesByCountry();
        break;
    }
  }

  function getWealthByCountry() {
    const grouped = d3.rollups(allData, v => d3.sum(v, d => d.finalWorth), d => d.country)
      .filter(([country]) => country && country !== 'Unknown')
      .sort((a, b) => b[1] - a[1])
      .slice(0, 5);

    const total = d3.sum(grouped, d => d[1]);
    return grouped.map(([country, wealth]) => ({
      label: country,
      value: wealth,
      percentage: ((wealth / total) * 100).toFixed(1),
      displayValue: `$${(wealth / 1000).toFixed(1)}T`,
      metadata: `${d3.rollup(allData.filter(d => d.country === country), v => v.length)} bilionários`
    }));
  }

  function getSelfMadeByCountry() {
    const topCountries = d3.rollups(allData, v => v.length, d => d.country)
      .filter(([country]) => country && country !== 'Unknown')
      .sort((a, b) => b[1] - a[1])
      .slice(0, 5)
      .map(d => d[0]);
    
    const result = topCountries.map(country => {
      const countryData = allData.filter(d => d.country === country);
      const selfMadeCount = countryData.filter(d => d.selfMade === true).length;
      const total = countryData.length;
      const percentage = total > 0 ? (selfMadeCount / total) * 100 : 0;
      
      return {
        label: country,
        value: percentage,
        percentage: percentage.toFixed(1),
        displayValue: `${percentage.toFixed(1)}%`,
        metadata: `${selfMadeCount} de ${total} bilionários`
      };
    }).sort((a, b) => b.value - a.value);
    
    console.log('Gender data for donut chart:', result);
    return result;
  }

  function getGenderByCountry() {
    const topCountries = d3.rollups(allData, v => v.length, d => d.country)
      .filter(([country]) => country && country !== 'Unknown')
      .sort((a, b) => b[1] - a[1])
      .slice(0, 5)
      .map(d => d[0]);
    
    const result = topCountries.map(country => {
      const countryData = allData.filter(d => d.country === country);
      const femaleCount = countryData.filter(d => d.gender === 'F').length;
      const total = countryData.length;
      const percentage = total > 0 ? (femaleCount / total) * 100 : 0;
      
      return {
        label: country,
        value: femaleCount,
        percentage: percentage.toFixed(1),
        displayValue: `${percentage.toFixed(1)}%`,
        metadata: `${femaleCount} mulheres de ${total} bilionários`
      };
    }).filter(d => d.value > 0).sort((a, b) => b.value - a.value);
    
    console.log('Gender data for donut chart:', result);
    return result;
  }

  function getAgeByCountry() {
    const topCountries = d3.rollups(allData, v => v.length, d => d.country)
      .filter(([country]) => country && country !== 'Unknown')
      .sort((a, b) => b[1] - a[1])
      .slice(0, 5)
      .map(d => d[0]);
    
    return topCountries.map(country => {
      const countryData = allData.filter(d => d.country === country && d.age > 0);
      const avgAge = countryData.length > 0 ? d3.mean(countryData, d => d.age) : 0;
      
      return {
        label: country,
        value: avgAge,
        percentage: avgAge.toFixed(1),
        displayValue: `${avgAge.toFixed(1)} anos`,
        metadata: `${countryData.length} bilionários com idade conhecida`
      };
    }).sort((a, b) => b.value - a.value);
  }

  function getIndustriesByCountry() {
    const topCountries = d3.rollups(allData, v => v.length, d => d.country)
      .filter(([country]) => country && country !== 'Unknown')
      .sort((a, b) => b[1] - a[1])
      .slice(0, 5)
      .map(d => d[0]);

    return topCountries.map(country => {
      const countryData = allData.filter(d => d.country === country);
      const industries = countryData.flatMap(d => 
        d.industries ? d.industries.split(',').map(i => i.trim()) : []
      );
      const topIndustry = d3.rollups(industries, v => v.length, d => d)
        .sort((a, b) => b[1] - a[1])[0];
      
      return {
        label: country,
        value: topIndustry ? topIndustry[1] : 0,
        percentage: countryData.length.toString(),
        displayValue: topIndustry ? `${topIndustry[0]}` : 'N/A',
        metadata: topIndustry ? `${topIndustry[1]} bilionários nesta indústria` : 'Sem dados'
      };
    }).sort((a, b) => b.value - a.value);
  }

  function nextView() {
    currentViewIndex = (currentViewIndex + 1) % views.length;
    currentView = views[currentViewIndex].key;
    updateChart();
  }

  function prevView() {
    currentViewIndex = (currentViewIndex - 1 + views.length) % views.length;
    currentView = views[currentViewIndex].key;
    updateChart();
  }

  function formatNumber(num) {
    if (num >= 1000000) return (num / 1000000).toFixed(1) + 'M';
    if (num >= 1000) return (num / 1000).toFixed(1) + 'K';
    return num.toString();
  }

  $: if (allData.length > 0) updateChart();
  $: currentViewData = views[currentViewIndex];
</script>

<svelte:head>
  <title>Trajetória dos Bilionários</title>
</svelte:head>

<div class="page-container">
  <nav class="main-nav">
    <h1 class="site-title">Análise Global de Bilionários</h1>
    <div class="nav-links">
      <a href="{base}/" class="nav-link">Visão Geral</a>
      <a href="{base}/trajetoria" class="nav-link active">Trajetória</a>
    </div>
  </nav>

  <main class="main-content">
    <h1>Trajetória dos Bilionários</h1>
    
    <section class="total-section">
      <h2>Patrimônio Total dos Bilionários</h2>
      {#if isLoading}
        <div class="loading">Carregando dados...</div>
      {:else}
        <div class="animated-total">
          <span class="currency">$</span>
          <span class="number">{formatNumber(animatedTotal)}</span>
          <span class="suffix">Bi</span>
        </div>
        <div class="total-info">{allData.length} bilionários no total</div>
      {/if}
    </section>

    <section class="chart-section">
      <div class="chart-container">
        <div class="chart-header">
          <button class="nav-arrow prev-arrow" on:click={prevView} disabled={isLoading}>←</button>
          
          <div class="chart-title-container">
            <h3 class="chart-title" style="color: {currentViewData.color}">{currentViewData.label}</h3>
            <div class="view-indicator">{currentViewIndex + 1} de {views.length}</div>
          </div>
          
          <button class="nav-arrow next-arrow" on:click={nextView} disabled={isLoading}>→</button>
        </div>
        
        {#if !isLoading && chartData.length > 0}
          <div class="visualization-container">
            {#if currentViewData.type === 'bar'}
              <div class="bar-chart">
                {#each chartData as item, i}
                  <div class="bar-item" on:mouseenter={() => hoveredItem = item} on:mouseleave={() => hoveredItem = null} style="animation-delay: {i * 0.1}s">
                    <div class="bar-label">{item.label}</div>
                    <div class="bar-container">
                      <div class="bar" style="width: {(item.value / Math.max(...chartData.map(d => d.value))) * 100}%; background: {currentViewData.color};"></div>
                      <div class="bar-value">{item.displayValue}</div>
                    </div>
                  </div>
                {/each}
              </div>
            {/if}
          </div>
        {:else if isLoading}
          <div class="loading-chart">
            <div class="loading-spinner"></div>
            <span>Carregando visualização...</span>
          </div>
        {:else}
          <div class="no-data">Nenhum dado disponível</div>
        {/if}
      </div>
    </section>
  </main>
</div>

<style>
  .page-container {
    max-width: 1200px;
    margin: 0 auto;
    padding: 0 20px;
    font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
    background: #fafafa;
    min-height: 100vh;
    color: #374151;
  }

  .main-nav {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 32px 0;
    border-bottom: 1px solid #e5e7eb;
    margin-bottom: 48px;
    background: transparent;
  }

  .site-title {
    font-size: 1.5em;
    font-weight: 600;
    color: #111827;
    margin: 0;
    letter-spacing: -0.025em;
  }

  .nav-links { display: flex; gap: 8px; }

  .nav-link {
    text-decoration: none;
    color: #6b7280;
    padding: 12px 20px;
    border-radius: 8px;
    transition: all 0.2s ease;
    font-weight: 500;
    font-size: 0.95em;
  }

  .nav-link:hover { background-color: #f3f4f6; color: #374151; }
  .nav-link.active { background-color: #374151; color: white; }

  .main-content { max-width: 900px; margin: 0 auto; }

  .main-content h1 {
    text-align: center;
    color: #111827;
    margin-bottom: 56px;
    font-weight: 700;
    font-size: 2.5em;
    letter-spacing: -0.025em;
    line-height: 1.2;
  }

  .total-section {
    text-align: center;
    margin-bottom: 72px;
    padding: 64px 40px;
    background: white;
    border: 1px solid #e5e7eb;
    border-radius: 12px;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05);
  }

  .total-section h2 {
    color: #6b7280;
    margin-bottom: 32px;
    font-weight: 500;
    font-size: 1em;
    text-transform: uppercase;
    letter-spacing: 0.1em;
  }

  .animated-total {
    font-size: 4em;
    font-weight: 800;
    color: #059669;
    margin: 32px 0;
    line-height: 1;
    letter-spacing: -0.02em;
  }

  .currency { font-size: 0.7em; vertical-align: top; color: #6b7280; }
  .number { font-size: 1em; }
  .suffix { font-size: 1em; color: #059669; margin-left: 8px; font-weight: 800; }

  .total-info { color: #6b7280; font-size: 1em; font-weight: 500; margin-top: 20px; }
  .loading { text-align: center; color: #6b7280; padding: 40px; font-weight: 500; }

  .chart-section { margin-bottom: 56px; }

  .chart-container {
    background: white;
    border: 1px solid #e5e7eb;
    border-radius: 12px;
    padding: 40px;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05);
  }

  .chart-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 48px;
    padding-bottom: 24px;
    border-bottom: 1px solid #f3f4f6;
  }

  .chart-title-container { text-align: center; flex: 1; }

  .chart-title {
    margin: 0;
    font-size: 1.3em;
    font-weight: 600;
    transition: color 0.2s ease;
    letter-spacing: -0.015em;
  }

  .view-indicator {
    color: #9ca3af;
    font-size: 0.8em;
    margin-top: 8px;
    font-weight: 500;
    text-transform: uppercase;
    letter-spacing: 0.05em;
  }

  .nav-arrow {
    background: #f9fafb;
    color: #4b5563;
    border: 1px solid #e5e7eb;
    border-radius: 8px;
    width: 48px;
    height: 48px;
    cursor: pointer;
    font-size: 1.2em;
    transition: all 0.2s ease;
    display: flex;
    align-items: center;
    justify-content: center;
    font-weight: 600;
  }

  .nav-arrow:hover:not(:disabled) { background: #374151; color: white; border-color: #374151; }
  .nav-arrow:disabled { background: #f9fafb; color: #d1d5db; cursor: not-allowed; border-color: #e5e7eb; }

  .visualization-container { position: relative; }
  .bar-chart { display: flex; flex-direction: column; gap: 24px; }

  .bar-item {
    position: relative;
    opacity: 0;
    animation: slideInUp 0.4s ease forwards;
    transition: transform 0.15s ease;
  }

  .bar-item:hover { transform: translateX(6px); }

  @keyframes slideInUp {
    from { opacity: 0; transform: translateY(20px); }
    to { opacity: 1; transform: translateY(0); }
  }

  .bar-label { font-weight: 600; color: #374151; margin-bottom: 10px; font-size: 1em; }

  .bar-container {
    display: flex;
    align-items: center;
    gap: 20px;
    background: #f9fafb;
    padding: 16px 20px;
    border: 1px solid #e5e7eb;
    border-radius: 8px;
  }

  .bar {
    height: 28px;
    border-radius: 4px;
    min-width: 4px;
    transition: all 0.6s cubic-bezier(0.4, 0, 0.2, 1);
    position: relative;
    overflow: hidden;
  }

  .bar-value { min-width: 90px; font-weight: 600; color: #374151; font-size: 0.95em; text-align: right; }

  .loading-chart {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 24px;
    padding: 80px;
    color: #6b7280;
  }

  .loading-spinner {
    width: 36px;
    height: 36px;
    border: 3px solid #f3f4f6;
    border-top: 3px solid #374151;
    border-radius: 50%;
    animation: spin 1s linear infinite;
  }

  @keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
  }

  .no-data {
    text-align: center;
    color: #9ca3af;
    padding: 80px;
    font-style: italic;
    font-size: 1em;
    font-weight: 500;
  }

  @media (max-width: 768px) {
    .main-nav { flex-direction: column; gap: 20px; }
    .animated-total { font-size: 3em; }
    .chart-header { flex-direction: column; gap: 28px; }
    .nav-arrow { width: 44px; height: 44px; font-size: 1.1em; }
    .total-section { padding: 40px 24px; }
    .chart-container { padding: 28px; }
  }
</style> 