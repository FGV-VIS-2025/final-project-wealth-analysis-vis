<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import { base } from '$app/paths';
  import '../../app.css'; 
  import './trajetoria.css'; 

  let allData = [];
  let currentView = 'wealth';
  let animatedTotal = 0;
  let finalTotal = 0;
  let chartData = [];
  let isLoading = true;
  let hoveredItem = null;

  const views = [
    { key: 'wealth', label: 'Patrimônio por País', type: 'bar', color: '#ffd700' },
    { key: 'selfmade', label: 'Self-Made por País', type: 'bar', color: '#ffd700' },
    { key: 'gender', label: 'Porcentagem de Mulheres Bilionárias por País', type: 'bar', color: '#ffd700' },
    { key: 'age', label: 'Idade Média por País', type: 'bar', color: '#ffd700' },
    { key: 'industries', label: 'Nº de Bilionários na Principal Indústria por País', type: 'bar', color: '#ffd700' }
  ];

  let currentViewIndex = 0;

  onMount(async () => {
    await loadData();
    animateTotal();
    updateChart();

    const sections = document.querySelectorAll('#trajetoria-page-container .story-section');
    const options = {
      root: document.querySelector('#trajetoria-page-container'), 
      rootMargin: '0px',
      threshold: 0.7
    };

    let activeSection = null; 

    const observer = new IntersectionObserver((entries, observer) => {
      entries.forEach(entry => {
        if (entry.isIntersecting && entry.intersectionRatio >= 0.7) {
          if (activeSection && activeSection !== entry.target) {
            activeSection.classList.remove('active');
          }
          entry.target.classList.add('active');
          activeSection = entry.target;
        } else {
          if (entry.target.classList.contains('active') && entry.target !== activeSection) {
          }
        }
      });
    }, options);

    sections.forEach(section => {
      observer.observe(section);
    });

    if (sections.length > 0) {
        setTimeout(() => {
            if (!activeSection) { 
                sections[0].classList.add('active');
                activeSection = sections[0];
            }
        }, 100); 
    }
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
        value: percentage,
        percentage: percentage.toFixed(1),
        displayValue: `${percentage.toFixed(1)}%`,
        metadata: `${femaleCount} mulheres de ${total} bilionários`
      };
    }).sort((a, b) => b.value - a.value);
    
    console.log('Gender data for bar chart:', result);
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
      const topIndustryRollup = d3.rollups(industries, v => v.length, d => d)
        .sort((a, b) => b[1] - a[1]);
      
      const topIndustry = topIndustryRollup.length > 0 ? topIndustryRollup[0] : null;
      
      return {
        label: country,
        value: topIndustry ? topIndustry[1] : 0,
        percentage: countryData.length.toString(),
        displayValue: topIndustry ? `${topIndustry[0]} (${topIndustry[1]})` : 'N/A',
        metadata: topIndustry ? `${topIndustry[1]} bilionários nesta indústria de um total de ${countryData.length} no país` : `Sem dados de indústria para ${countryData.length} bilionários`
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

  async function navigateToHome(event) {
    event.preventDefault();
    const targetHref = event.currentTarget.href;
    const pageContainer = document.getElementById('trajetoria-page-container');
    if (pageContainer) {
      pageContainer.classList.add('fading-out');
      await new Promise(resolve => setTimeout(resolve, 700));
    }
    window.location.href = targetHref;
  }

  $: if (allData.length > 0) updateChart();
  $: currentViewData = views[currentViewIndex];
</script>

<svelte:head>
  <title>Visão Geral dos Bilionários por País</title>
</svelte:head>

<div class="page-container story-container" id="trajetoria-page-container">
  <section class="story-section side-by-side">
    <div class="content-half left total-wealth-display">
      <div class="total-section">
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
      </div>
    </div>
    <div class="content-half right text-content">
        <h2>Desigualdade Global da Riqueza</h2>
        <p>
            A riqueza total do mundo, em 2024, foi estimada em US$ 454,38 trilhões. 
            No entanto, a riqueza global está muito desequilibrada, com o 1% mais rico 
            controlando quase dois terços da riqueza gerada since 2020, enquanto o 
            restante da população, cerca de 7 bilhões de pessoas, acumulou muito menos.
            É fato que podemos ver que os bilionários têm uma significância nessa desigualdade.
        </p>
    </div>
  </section>

  <section class="story-section side-by-side">
    <div class="content-half left text-content">
        <h2>Entendendo os Bilionários</h2>
        <p>
            Visando entender melhor quem são essas pessoas e como elas ficaram ricas, 
            nós separamos os países mais relevantes para um análise macro da situação.
            Conseguimos identificar alguns insights interessantes, como a alta concentração de bilionários
            em países como Estados Unidos e China, além de um padrão claro na idade média dos bilionários.
        </p>
    </div>
    <div class="content-half right chart-area-wrapper">
      <div class="chart-section">
        <div class="chart-container">
          <div class="chart-header">
            <button class="nav-arrow prev-arrow" on:click={prevView} disabled={isLoading}>←</button>
            <div class="chart-title-container">
              <h3 class="chart-title">{currentViewData.label}</h3>
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
      </div>
    </div>
  </section>

  <section class="story-section navigation-section-wrapper">
    <div class="navigation-section">
        <a href="{base}/" class="nav-button-container back-button" on:click={navigateToHome}>
            <div class="nav-arrow-button back-arrow-icon">
            <span>&larr;</span> 
            </div>
            <div class="nav-button-text">
            Voltar para página inicial
            </div>
        </a>
    </div>
  </section>
</div>
