<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import { base } from '$app/paths';
  import '../../app.css'; 
  import './trajetoria.css'; 
  import BillionaireMigrationMap from '$lib/components/BillionaireMigrationMap.svelte';
  import { fade } from 'svelte/transition';

  let allData = [];
  let currentView = 'wealth';
  let animatedTotal = 0;
  let finalTotal = 0;
  let chartData = [];
  let isLoading = true;
  let hoveredItem = null;
  let selectedCountrySelfMade = 'Todos';
  let countryOptionsSelfMade = [];
  let selfMadeDataFiltered = [];
  let selfMadeInsight = '';
  let selectedCountryAge = 'Todos';
  let countryOptionsAge = [];
  let ageDataFiltered = [];
  let ageInsight = '';

  const views = [
    { key: 'wealth', label: 'Patrimônio por País', type: 'bar', color: '#ffd700' },
    { key: 'selfmade', label: 'Self-Made por País', type: 'bar', color: '#ffd700' },
    { key: 'gender', label: 'Porcentagem de Mulheres Bilionárias por País', type: 'bar', color: '#ffd700' },
    { key: 'age', label: 'Idade Média por País', type: 'bar', color: '#ffd700' },
    { key: 'industries', label: 'Nº de Bilionários na Principal Indústria por País', type: 'bar', color: '#ffd700' }
  ];

  let currentViewIndex = 0;

  const xAxisTicks = [0, 1000, 2000, 3000, 4000, 5000];

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

  async function navigateTo(event, targetPath) {
    event.preventDefault();
    const pageContainer = document.getElementById('trajetoria-page-container');
    if (pageContainer) {
      pageContainer.classList.add('fading-out');
      await new Promise(resolve => setTimeout(resolve, 700));
    }
    window.location.href = targetPath;
  }

  function updateSelfMadeInsight() {
    if (!selfMadeDataFiltered.length) {
      selfMadeInsight = '';
      return;
    }
    const total = selfMadeDataFiltered.reduce((sum, d) => sum + d.count, 0);
    const selfMade = selfMadeDataFiltered.find(d => d.label === 'Self-Made');
    const notSelfMade = selfMadeDataFiltered.find(d => d.label === 'Não Self-Made');
    const percentSelf = selfMade ? ((selfMade.count / total) * 100).toFixed(1) : '0.0';
    const percentNot = notSelfMade ? ((notSelfMade.count / total) * 100).toFixed(1) : '0.0';
    if (selectedCountrySelfMade === 'Todos') {
      selfMadeInsight = `Globalmente, ${percentSelf}% dos bilionários são self-made e ${percentNot}% não são self-made.`;
    } else {
      selfMadeInsight = `No país selecionado, ${percentSelf}% dos bilionários são self-made e ${percentNot}% não são self-made.`;
    }
  }

  function updateAgeInsight() {
    if (!ageDataFiltered.length) {
      ageInsight = '';
      return;
    }
    const total = ageDataFiltered.reduce((sum, d) => sum + d.count, 0);
    const weightedSum = ageDataFiltered.reduce((sum, d) => {
      const [start] = d.ageGroup.split('-');
      return sum + (parseInt(start) + 5) * d.count;
    }, 0);
    const avgAge = total > 0 ? (weightedSum / total).toFixed(1) : '0.0';
    if (selectedCountryAge === 'Todos') {
      ageInsight = `Globalmente, a idade média dos bilionários é ${avgAge} anos.`;
    } else {
      ageInsight = `No país selecionado, a idade média dos bilionários é ${avgAge} anos.`;
    }
  }

  $: if (allData.length > 0) {
    countryOptionsSelfMade = Array.from(new Set(allData.map(d => d.country).filter(Boolean))).sort();
    countryOptionsSelfMade.unshift('Todos');
    countryOptionsAge = Array.from(new Set(allData.map(d => d.country).filter(Boolean))).sort();
    countryOptionsAge.unshift('Todos');
  }

  $: if (selectedCountrySelfMade && allData.length > 0 && currentView === 'selfmade') {
    if (selectedCountrySelfMade === 'Todos') {
      const total = allData.length;
      const selfMade = allData.filter(d => d.selfMade === true).length;
      const notSelfMade = total - selfMade;
      selfMadeDataFiltered = [
        { label: 'Self-Made', count: selfMade },
        { label: 'Não Self-Made', count: notSelfMade }
      ];
    } else {
      const countryData = allData.filter(d => d.country === selectedCountrySelfMade);
      const total = countryData.length;
      const selfMade = countryData.filter(d => d.selfMade === true).length;
      const notSelfMade = total - selfMade;
      selfMadeDataFiltered = [
        { label: 'Self-Made', count: selfMade },
        { label: 'Não Self-Made', count: notSelfMade }
      ];
    }
    updateSelfMadeInsight();
  }

  $: if (selectedCountryAge && allData.length > 0 && currentView === 'age') {
    let filtered = allData;
    if (selectedCountryAge !== 'Todos') {
      filtered = allData.filter(d => d.country === selectedCountryAge);
    }
    ageDataFiltered = d3.rollups(filtered.filter(d => d.age && d.age > 0), v => v.length, d => Math.floor(d.age / 10) * 10)
      .map(([key, value]) => ({ ageGroup: `${key}-${key+9}`, count: value }))
      .sort((a,b) => parseInt(a.ageGroup) - parseInt(b.ageGroup));
    updateAgeInsight();
  }

  $: if (allData.length > 0) updateChart();
  $: currentViewData = views[currentViewIndex];
  $: maxValue = Math.max(...chartData.map(d => d.value));
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
            controlando quase dois terços da riqueza gerada desde 2020, enquanto o 
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
          
          {#if currentView === 'selfmade'}
            <!-- filtro removido -->
          {/if}
          {#if currentView === 'age'}
            <div class="gender-filter-container-side" style="margin-bottom: 10px; justify-content: flex-end;">
              <label for="country-select-age">Filtrar por país:</label>
              <select id="country-select-age" bind:value={selectedCountryAge}>
                {#each countryOptionsAge as country}
                  <option value={country}>{country}</option>
                {/each}
              </select>
            </div>
          {/if}
          <div class="visualization-container">
            {#if currentView === 'age'}
              {#key currentView}
                {#each chartData as item, i}
                  <div class="bar-item" on:mouseenter={() => hoveredItem = item} on:mouseleave={() => hoveredItem = null} style="animation-delay: {i * 0.1}s">
                    <div class="bar-label">{item.label}</div>
                    <div class="bar-container">
                      <div class="bar" style="width: {(item.value / maxValue) * 100}%; background: {currentViewData.color};"></div>
                      <div class="bar-value">{item.displayValue}</div>
                    </div>
                  </div>
                {/each}
                <div class="gender-insight" style="margin-top: 18px;">{ageInsight}</div>
              {/key}
            {:else if !isLoading && chartData.length > 0}
              {#key currentView}
                {#each chartData as item, i}
                  <div class="bar-item" on:mouseenter={() => hoveredItem = item} on:mouseleave={() => hoveredItem = null} style="animation-delay: {i * 0.1}s">
                    <div class="bar-label">
                      {item.label}
                      {#if currentView === 'industries'}
                        {#if item.displayValue && item.displayValue !== 'N/A'}
                          {' (' + item.displayValue.split(' (')[0] + ')'}
                        {/if}
                      {/if}
                    </div>
                    <div class="bar-container">
                      {#if currentView === 'wealth'}
                        <div class="bar" style="width: {(item.value / 5000000) * 100}%; background: {currentViewData.color};"></div>
                        <div class="bar-value">{item.displayValue}</div>
                      {:else if currentView === 'selfmade' || currentView === 'gender'}
                        <div class="bar" style="width: {item.value}%; background: {currentViewData.color};"></div>
                        <div class="bar-value">{item.displayValue}</div>
                      {:else if currentView === 'age'}
                        <!-- nunca cai aqui, pois já tratamos acima -->
                      {:else if currentView === 'industries'}
                        <div class="bar" style="width: {(item.value / 200) * 100}%; background: {currentViewData.color};"></div>
                        <div class="bar-value">{item.value}</div>
                      {/if}
                    </div>
                  </div>
                {/each}
              {/key}
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
    </div>
  </section>

  <!-- Seção do Mapa de Migração de Bilionários -->
  <section class="story-section full-width map-section">
    <div class="map-text-content">
      <h1>Fluxos Migratórios da Elite Global</h1>
      <p>
        A riqueza transcende fronteiras. Este mapa revela os padrões migratórios dos bilionários, 
        mostrando como a elite financeira se move pelo mundo. Cada círculo representa a concentração 
        de bilionários por país, enquanto as setas douradas indicam os fluxos migratórios mais significativos 
        entre nações de nascimento e residência atual.
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
