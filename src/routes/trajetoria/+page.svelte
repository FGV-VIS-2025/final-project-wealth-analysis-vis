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
  <section class="story-section full-width map-section">
    <div class="map-text-content">
      <h2>Fluxos Migratórios da Elite Global</h2>
      <p>
        Retomando a indagação feita na seção de análise por país, esta visualização revela que países como Brasil, Canadá, Suécia, Noruega, Austrália, entre outros, registram a saída de muitos de seus bilionários.        É razoável supor que o motivo esteja relacionado à alta carga tributária. Diante disso, muitos magnatas optam por migrar para regiões com políticas tributárias mais flexíveis, em busca de um ambiente financeiro menos hostil.
      </p>
    </div>
    
    <div class="map-container-wrapper">
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
      <h2>Raio-X dos Países com Maior Concentração de Bilionários</h2>
      <p>
        Para validar nossa hipótese, analisemos as estatísticas dos países com maior concentração de indivíduos dessa classe. Em geral, observa-se que ou o país possui uma carga tributária reduzida, ou, quando há alta tributação, ela é acompanhada por uma moeda desvalorizada e índices de inflação elevados. Raramente se observa um padrão fora dessas duas categorias.
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
    <div class="navigation-buttons-container">
        <a href="{base}/" 
           class="nav-button-container back-button" 
           on:click={(e) => navigateTo(e, `${base}/`)}>
            <div class="nav-arrow-button back-arrow-icon">
              <span>&larr;</span> 
            </div>
            <div class="nav-button-text">
              Voltar para Página Inicial
            </div>
        </a>
        
        <a href="{base}/search" 
           class="nav-button-container forward-button" 
           on:click={(e) => navigateTo(e, `${base}/search`)}>
            <div class="nav-button-text">
              Explorar Ferramenta de Busca
            </div>
            <div class="nav-arrow-button next-arrow-icon">
              <span>&rarr;</span> 
            </div>
        </a>
    </div>
  </section>
</div>
