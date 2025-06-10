<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import { base } from '$app/paths';
  import '../../app.css'; 
  import './trajetoria.css'; 
  import BillionaireMigrationMap from '$lib/components/BillionaireMigrationMap.svelte';
  import CountryRadarChart from '$lib/components/CountryRadarChart.svelte';

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
    { key: 'gender', label: 'Porcentagem de Mulheres Bilionárias por País', type: 'bar', color: '#ffd700' }
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

  async function navigateTo(event, targetPath) {
    event.preventDefault();
    const pageContainer = document.getElementById('trajetoria-page-container');
    if (pageContainer) {
      pageContainer.classList.add('fading-out');
      await new Promise(resolve => setTimeout(resolve, 700));
    }
    window.location.href = targetPath;
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

  <!-- Seção do Mapa de Migração de Bilionários -->
  <section class="story-section side-by-side map-section-redesigned">
    <div class="content-third left map-text-content">
      <h2>Fluxos Migratórios da Elite Global</h2>
      <p>
        <strong>Conectando com as análises anteriores:</strong> Após observarmos as concentrações de bilionários por país e as disparidades de gênero, surge uma questão fundamental: <em>para onde estão indo esses bilionários?</em>
      </p>
      <p>
        Esta visualização interativa revela os padrões migratórios da elite financeira global. Países como Brasil, Canadá e Suécia registram significativa saída de bilionários, enquanto destinos como Estados Unidos, Singapura e Suíça atraem essa população. 
      </p>
      <p>
        <strong>Use a interatividade:</strong> Clique nos países do mapa para focar nos fluxos específicos e descobrir os padrões únicos de cada região.
      </p>
    </div>
    
    <div class="content-two-thirds right map-container-wrapper">
      {#if !isLoading && allData.length > 0}
        <BillionaireMigrationMap data={allData} />
      {:else if isLoading}
        <div class="loading-map">
          <div class="loading-spinner"></div>
          <span>Carregando mapa de migração...</span>
        </div>
      {:else}
        <div class="no-data-map">Nenhum dado disponível para o mapa</div>
      {/if}
    </div>
  </section>

  <!-- Nova Seção do Gráfico Radar -->
  <section class="story-section full-width radar-section">
    <div class="radar-text-content">
      <h2>Análise Comparativa: Por Que Alguns Países Atraem Bilionários?</h2>
      <p>
        Este radar interativo compara múltiplos indicadores econômicos e sociais dos principais países concentradores de riqueza. Selecione até 4 países para comparar suas características em riqueza total, PIB, expectativa de vida, taxa de impostos e inflação.
        Este radar interativo compara múltiplos indicadores econômicos e sociais dos principais países concentradores de riqueza. Selecione até 4 países para comparar suas características em riqueza total, PIB, expectativa de vida, taxa de impostos e inflação.

      </p>

    </div>
    
    <div class="radar-container-wrapper">
      {#if !isLoading && allData.length > 0}
        <CountryRadarChart data={allData} />
      {:else if isLoading}
        <div class="loading-radar">
          <div class="loading-spinner"></div>
          <span>Carregando análise...</span>
        </div>
      {:else}
        <div class="no-data-radar">Nenhum dado disponível</div>
      {/if}
    </div>
  </section>

  <section class="story-section navigation-section-wrapper" id="trajetoria-final-nav">
    <div class="enhanced-navigation-container">
      <div class="progress-indicator">
        <div class="progress-step completed">1</div>
        <div class="progress-line completed"></div>
        <div class="progress-step active">2</div>
        <div class="progress-line"></div>
        <div class="progress-step">3</div>
      </div>
      
      <h3 class="nav-section-title">Continue Explorando</h3>
      
      <div class="navigation-buttons-container">
        <a href="{base}/" 
           class="enhanced-nav-btn back-btn" 
           on:click={(e) => navigateTo(e, `${base}/`)}>
                         <div class="nav-btn-content">
               <div class="nav-btn-text">
                 <div class="nav-btn-title">Página Inicial</div>
                 <div class="nav-btn-subtitle">Voltar ao início da análise</div>
               </div>
               <div class="nav-btn-arrow">
                 <span>&larr;</span> 
               </div>
             </div>
        </a>
        
        <a href="{base}/search" 
           class="enhanced-nav-btn forward-btn" 
           on:click={(e) => navigateTo(e, `${base}/search`)}>
                         <div class="nav-btn-content">
               <div class="nav-btn-text">
                 <div class="nav-btn-title">Ferramenta de Busca</div>
                 <div class="nav-btn-subtitle">Explore dados específicos</div>
               </div>
               <div class="nav-btn-arrow">
                 <span>&rarr;</span> 
               </div>
             </div>
        </a>
      </div>
    </div>
  </section>
</div>

<style>
  .content-third {
    flex: 1;
    max-width: 30%;
    padding: 2rem;
  }

  .content-two-thirds {
    flex: 2;
    max-width: 70%;
    padding: 1rem;
  }

  .map-section-redesigned {
    align-items: flex-start;
    gap: 2rem;
    min-height: 800px;
  }

  .map-container-wrapper {
    display: flex;
    justify-content: center;
    align-items: center;
    width: 100%;
    overflow-x: auto;
  }

  .map-text-content {
    position: sticky;
    top: 2rem;
  }

  .map-text-content h2 {
    color: #ffd700;
    font-size: 24px;
    font-weight: 600;
    margin-bottom: 1.5rem;
  }

  .map-text-content p {
    color: #e2e8f0;
    line-height: 1.7;
    margin-bottom: 1.2rem;
    font-size: 16px;
  }

  .map-text-content strong {
    color: #ffd700;
  }

  .map-text-content em {
    color: #a0aec0;
    font-style: italic;
  }

  @media (max-width: 1024px) {
    .content-third,
    .content-two-thirds {
      max-width: 100%;
      flex: 1;
    }

    .map-section-redesigned {
      flex-direction: column;
      min-height: auto;
    }

    .map-text-content {
      position: static;
      text-align: center;
    }
  }
</style>
