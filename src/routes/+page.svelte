<script>
  import AgeHistogram from '$lib/components/AgeHistogram.svelte';
  import CountryBarChart from '$lib/components/CountryBarChart.svelte';
  import GenderChart from '$lib/components/GenderChart.svelte';
  import SelfMadePieChart from '$lib/components/SelfMadePieChart.svelte';
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import { base } from '$app/paths';
  import '../app.css';

  let allData = [];
  let ageData = [];
  let countryData = [];
  let genderData = [];
  let selfMadeData = [];
  let selectedCountry = 'Todos';
  let countryOptions = [];
  let genderDataFiltered = [];
  let genderDataGlobal = [];
  let genderInsight = '';
  let selectedContinent = 'Todos';
  let continentOptions = ['Todos', 'América do Norte', 'América do Sul', 'Europa', 'Ásia', 'Oceania', 'África'];
  let countryDataAll = [];
  let countryInsight = '';
  let selectedCountrySelfMade = 'Todos';
  let countryOptionsSelfMade = [];
  let selfMadeDataFiltered = [];
  let selfMadeInsight = '';
  let selectedCountryAge = 'Todos';
  let countryOptionsAge = [];
  let ageDataFiltered = [];
  let ageInsight = '';
  const countryToContinent = {
    'United States': 'América do Norte',
    'Canada': 'América do Norte',
    'Mexico': 'América do Norte',
    'Brazil': 'América do Sul',
    'Chile': 'América do Sul',
    'Argentina': 'América do Sul',
    'Germany': 'Europa',
    'United Kingdom': 'Europa',
    'France': 'Europa',
    'Italy': 'Europa',
    'Russia': 'Europa',
    'Switzerland': 'Europa',
    'Sweden': 'Europa',
    'Spain': 'Europa',
    'Netherlands': 'Europa',
    'Czech Republic': 'Europa',
    'China': 'Ásia',
    'India': 'Ásia',
    'Japan': 'Ásia',
    'Hong Kong': 'Ásia',
    'Singapore': 'Ásia',
    'South Korea': 'Ásia',
    'Israel': 'Ásia',
    'Indonesia': 'Ásia',
    'Thailand': 'Ásia',
    'Turkey': 'Ásia',
    'Saudi Arabia': 'Ásia',
    'Taiwan': 'Ásia',
    'Australia': 'Oceania',
    'South Africa': 'África',
  };

  // Função executada quando o componente é montado
  // Carrega os dados do CSV e inicializa as visualizações
  onMount(async () => {
    const csvPath = `${base}/Billionaires Statistics Dataset.csv`;
    const rawData = await d3.csv(csvPath);
    
    // Converte os dados brutos para o formato adequado
    allData = rawData.map(d => ({
      ...d,
      age: +d.age,
      finalWorth: +d.finalWorth
    }));

    // Processa os dados de idade agrupando em faixas etárias de 10 anos
    ageData = d3.rollups(allData.filter(d => d.age && d.age > 0), v => v.length, d => Math.floor(d.age / 10) * 10)
                .map(([key, value]) => ({ ageGroup: `${key}-${key+9}`, count: value }))
                .sort((a,b) => parseInt(a.ageGroup) - parseInt(b.ageGroup));

    // Processa os dados por país, pegando todos os países
    countryDataAll = d3.rollups(allData.filter(d => d.country), v => v.length, d => d.country)
      .map(([key, value]) => ({ country: key, count: value }))
      .sort((a,b) => b.count - a.count);

    // Processa os dados por gênero
    genderData = d3.rollups(allData.filter(d => d.gender), v => v.length, d => d.gender)
                   .map(([key, value]) => ({ gender: key || 'Unknown', count: value }));

    // Processa os dados self-made
    const total = allData.length;
    const selfMade = allData.filter(d => d.selfMade === 'TRUE' || d.selfMade === 'True' || d.selfMade === 'true' || d.selfMade === true).length;
    const notSelfMade = total - selfMade;
    selfMadeData = [
      { label: 'Self-Made', value: selfMade },
      { label: 'Não Self-Made', value: notSelfMade }
    ];

    // Gera lista de países para o filtro
    countryOptions = Array.from(new Set(allData.map(d => d.country).filter(Boolean))).sort();
    countryOptions.unshift('Todos');

    // Dados globais
    genderDataGlobal = d3.rollups(allData.filter(d => d.gender), v => v.length, d => d.gender)
      .map(([key, value]) => ({ gender: key || 'Unknown', count: value }));

    // Inicializa filtrado como global
    genderDataFiltered = [...genderDataGlobal];
    updateGenderInsight();

    // Configuração do sistema de observação para o scroll-telling
    const sections = document.querySelectorAll('#scroll-telling-page .story-section');
    const options = {
      root: document.querySelector('#scroll-telling-page'), 
      rootMargin: '0px',
      threshold: 0.7
    };

    let activeSection = null; 

    // Observer que controla a animação das seções durante o scroll
    const observer = new IntersectionObserver((entries, observer) => {
      entries.forEach(entry => {
        if (entry.isIntersecting && entry.intersectionRatio >= 0.7) {
          if (activeSection && activeSection !== entry.target) {
            activeSection.classList.remove('active');
          }
          entry.target.classList.add('active');
          activeSection = entry.target;
        } else if (!entry.isIntersecting && entry.target === activeSection) {
        }
        if (!entry.isIntersecting || entry.intersectionRatio < 0.7) {
            if (entry.target.classList.contains('active') && entry.target !== activeSection) {
                entry.target.classList.remove('active');
            }
        }
      });
    }, options);

    // Observa todas as seções para ativar as animações
    sections.forEach(section => {
      observer.observe(section);
    });

    // Ativa a primeira seção após um pequeno delay
    if (sections.length > 0) {
        setTimeout(() => {
            if (!activeSection) {
                sections[0].classList.add('active');
                activeSection = sections[0];
            }
        }, 100); 
    }

    // Após carregar allData:
    countryOptionsAge = Array.from(new Set(allData.map(d => d.country).filter(Boolean))).sort();
    countryOptionsAge.unshift('Todos');

  });

  // Função que gerencia a navegação para a página de trajetória
  // Adiciona uma animação de fade-out antes da transição
  async function navigateToTrajectory(event) {
    event.preventDefault();
    const targetHref = event.currentTarget.href; 

    const pageContainer = document.getElementById('scroll-telling-page');
    if (pageContainer) {
      pageContainer.classList.add('fading-out');
      await new Promise(resolve => setTimeout(resolve, 700)); 
    }
    window.location.href = targetHref;
  }

  // Atualiza o insight do gênero com base no país selecionado
  function updateGenderInsight() {
    if (!genderDataFiltered.length) {
      genderInsight = '';
      return;
    }
    const total = genderDataFiltered.reduce((sum, d) => sum + d.count, 0);
    const female = genderDataFiltered.find(d => d.gender === 'F');
    const male = genderDataFiltered.find(d => d.gender === 'M');
    const percentF = female ? ((female.count / total) * 100).toFixed(1) : '0.0';
    const percentM = male ? ((male.count / total) * 100).toFixed(1) : '0.0';
    if (selectedCountry === 'Todos') {
      genderInsight = `Globalmente, ${percentF}% dos bilionários são mulheres e ${percentM}% são homens.`;
    } else {
      // Compara com global
      const totalGlobal = genderDataGlobal.reduce((sum, d) => sum + d.count, 0);
      const femaleGlobal = genderDataGlobal.find(d => d.gender === 'F');
      const percentFGlobal = femaleGlobal ? ((femaleGlobal.count / totalGlobal) * 100).toFixed(1) : '0.0';
      genderInsight = `No país selecionado, ${percentF}% dos bilionários são mulheres (global: ${percentFGlobal}%).`;
    }
  }

  // Atualiza os dados de gênero com base no país selecionado
  $: if (selectedCountry && allData.length > 0) {
    if (selectedCountry === 'Todos') {
      genderDataFiltered = [...genderDataGlobal];
    } else {
      genderDataFiltered = d3.rollups(
        allData.filter(d => d.gender && d.country === selectedCountry),
        v => v.length,
        d => d.gender
      ).map(([key, value]) => ({ gender: key || 'Unknown', count: value }));
    }
    updateGenderInsight();
  }

  $: filteredCountryData = countryDataAll.filter(d => selectedContinent === 'Todos' || countryToContinent[d.country] === selectedContinent);
  $: countryData = filteredCountryData.slice(0, 10);

  $: if (countryData.length > 0 && selectedContinent !== 'Todos') {
    const total = countryData.reduce((sum, d) => sum + d.count, 0);
    const top = countryData[0];
    const percent = ((top.count / total) * 100).toFixed(1);
    countryInsight = `${top.country} concentra ${percent}% dos bilionários do continente selecionado.`;
  } else if (countryData.length > 0 && selectedContinent === 'Todos') {
    const total = countryData.reduce((sum, d) => sum + d.count, 0);
    const top = countryData[0];
    const percent = ((top.count / total) * 100).toFixed(1);
    countryInsight = `${top.country} concentra ${percent}% dos bilionários do top 10 global.`;
  } else {
    countryInsight = '';
  }

  function updateSelfMadeInsight() {
    if (!selfMadeDataFiltered.length) {
      selfMadeInsight = '';
      return;
    }
    const total = selfMadeDataFiltered.reduce((sum, d) => sum + d.value, 0);
    const selfMade = selfMadeDataFiltered.find(d => d.label === 'Self-Made');
    const notSelfMade = selfMadeDataFiltered.find(d => d.label === 'Não Self-Made');
    const percentSelf = selfMade ? ((selfMade.value / total) * 100).toFixed(1) : '0.0';
    const percentNot = notSelfMade ? ((notSelfMade.value / total) * 100).toFixed(1) : '0.0';
    if (selectedCountrySelfMade === 'Todos') {
      selfMadeInsight = `Globalmente, ${percentSelf}% dos bilionários são self-made e ${percentNot}% não são self-made.`;
    } else {
      selfMadeInsight = `No país selecionado, ${percentSelf}% dos bilionários são self-made e ${percentNot}% não são self-made.`;
    }
  }

  $: if (allData.length > 0) {
    countryOptionsSelfMade = Array.from(new Set(allData.map(d => d.country).filter(Boolean))).sort();
    countryOptionsSelfMade.unshift('Todos');
  }

  $: if (selectedCountrySelfMade && allData.length > 0) {
    if (selectedCountrySelfMade === 'Todos') {
      const total = allData.length;
      const selfMade = allData.filter(d => d.selfMade === 'TRUE' || d.selfMade === 'True' || d.selfMade === 'true' || d.selfMade === true).length;
      const notSelfMade = total - selfMade;
      selfMadeDataFiltered = [
        { label: 'Self-Made', value: selfMade },
        { label: 'Não Self-Made', value: notSelfMade }
      ];
    } else {
      const countryData = allData.filter(d => d.country === selectedCountrySelfMade);
      const total = countryData.length;
      const selfMade = countryData.filter(d => d.selfMade === 'TRUE' || d.selfMade === 'True' || d.selfMade === 'true' || d.selfMade === true).length;
      const notSelfMade = total - selfMade;
      selfMadeDataFiltered = [
        { label: 'Self-Made', value: selfMade },
        { label: 'Não Self-Made', value: notSelfMade }
      ];
    }
    updateSelfMadeInsight();
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

  $: if (selectedCountryAge && allData.length > 0) {
    let filtered = allData;
    if (selectedCountryAge !== 'Todos') {
      filtered = allData.filter(d => d.country === selectedCountryAge);
    }
    ageDataFiltered = d3.rollups(filtered.filter(d => d.age && d.age > 0), v => v.length, d => Math.floor(d.age / 10) * 10)
      .map(([key, value]) => ({ ageGroup: `${key}-${key+9}`, count: value }))
      .sort((a,b) => parseInt(a.ageGroup) - parseInt(b.ageGroup));
    updateAgeInsight();
  }

</script>

<svelte:head>
  <title>Desvendando o Perfil Bilionário</title>
</svelte:head>

<div id="scroll-telling-page" class="story-container">
  <section class="story-section full-page-section">
    <div class="text-content centered-content">
      <h1>Análise Global de Bilionários</h1>
      <p>
        Nesta página, tentamos desvendar o que define o perfil de um bilionário.
        Mergulhamos nos dados para expor não apenas números, mas para estimular um olhar crítico sobre
        a concentração de riqueza e os padrões que emergem, como as disparidades de gênero e a distribuição geográfica do capital.
      </p>
    </div>
  </section>

  <section class="story-section side-by-side">
    <div class="text-content left">
      <h2>Gênero e a Elite Financeira: Um Clube Exclusivo?</h2>
      <p>
        Como o gênero se reflete no topo da pirâmide financeira? Esta seção deixa claro a 
        representação de diferentes gêneros na população bilionária. O gráfico não mente:
        a disparidade é evidente.
      </p>
      <p>
        Estes dados são um convite à reflexão: quais são as barreiras sistêmicas e culturais 
        que perpetuam essa desigualdade no acesso à riqueza e ao poder econômico?
      </p>
    </div>
    <div class="chart-content right">
      <div class="gender-chart-flex-col">
        <div class="gender-filter-container-side">
          <label for="country-select">Filtrar por país:</label>
          <select id="country-select" bind:value={selectedCountry}>
            {#each countryOptions as country}
              <option value={country}>{country}</option>
            {/each}
          </select>
        </div>
        <div class="gender-chart-area">
          {#if genderDataFiltered.length > 0}
            <GenderChart data={genderDataFiltered} animate={true} />
            <div class="gender-insight">{genderInsight}</div>
          {:else}
            <p class="loading-text">Carregando dados de gênero...</p>
          {/if}
        </div>
      </div>
    </div>
  </section>

  <section class="story-section side-by-side">
    <div class="text-content left">
      <div class="country-chart-text">
        <h2>A Geografia da Riqueza Extrema: Onde o Capital se Concentra?</h2>
        <p>
          Onde, no mundo, os bilionários fincam suas raízes – e suas fortunas? Este gráfico 
          destaca os países com o maior número deles, oferecendo uma perspectiva geográfica da 
          gritante concentração de riqueza e, por consequência, de poder.
        </p>
        <p>
          Note a dominância de certas nações. Quais fatores econômicos, políticos e históricos 
          você acredita que alimentam essa distribuição tão desigual de capital global?
        </p>
      </div>
    </div>
    <div class="chart-content right">
      <div class="country-chart-area">
        <div class="continent-filter-container-top">
          <label for="continent-select">Filtrar por continente:</label>
          <select id="continent-select" bind:value={selectedContinent}>
            {#each continentOptions as continent}
              <option value={continent}>{continent}</option>
            {/each}
          </select>
        </div>
        {#if countryData.length > 0}
          <CountryBarChart data={countryData} />
          <div class="country-insight">{countryInsight}</div>
        {:else}
          <p class="loading-text">Nenhum país encontrado para o continente selecionado.</p>
        {/if}
      </div>
    </div>
  </section>

  <section class="story-section side-by-side">
    <div class="text-content left">
      <h2>A Idade da Riqueza: Quando o Capital Atinge o Ápice?</h2>
      <p>
        A distribuição de bilionários por faixa etária revela mais do que simples números; 
        ela pode indicar os mecanismos e o tempo necessário para a acumulação extrema de capital.
        Este histograma expõe a contagem de bilionários em diferentes fases da vida.
      </p>
      <p>
        Observamos certas concentrações: seriam estes os "anos dourados" do empreendedorismo agressivo, 
        o auge da especulação financeira ou o reflexo de fortunas herdadas se consolidando?
      </p>
    </div>
    <div class="chart-content right">
      <div class="gender-chart-flex-col">
        <div class="gender-filter-container-side" style="justify-content: flex-end;">
          <label for="country-select-age">Filtrar por país:</label>
          <select id="country-select-age" bind:value={selectedCountryAge}>
            {#each countryOptionsAge as country}
              <option value={country}>{country}</option>
            {/each}
          </select>
        </div>
        <div class="gender-chart-area">
          {#if ageDataFiltered.length > 0}
            <AgeHistogram data={ageDataFiltered} />
            <div class="gender-insight">{ageInsight}</div>
          {:else}
            <p class="loading-text">Carregando dados de idade...</p>
          {/if}
        </div>
      </div>
    </div>
  </section>

  <section class="story-section side-by-side">
    <div class="text-content left">
      <h2>Origem da Fortuna: Self-Made ou Herança?</h2>
      <p>
        O gráfico ao lado mostra a proporção de bilionários que construíram sua fortuna do zero (self-made) em comparação com aqueles que herdaram ou receberam grande parte de sua riqueza.
      </p>
      <p>
        Essa divisão revela o quanto o topo da pirâmide financeira é resultado de empreendedorismo, inovação e oportunidades, ou de dinastias familiares e transferências de patrimônio.
      </p>
    </div>
    <div class="chart-content right">
      <div class="gender-chart-flex-col">
        <div class="gender-filter-container-side">
          <label for="country-select-selfmade">Filtrar por país:</label>
          <select id="country-select-selfmade" bind:value={selectedCountrySelfMade}>
            {#each countryOptionsSelfMade as country}
              <option value={country}>{country}</option>
            {/each}
          </select>
        </div>
        <div class="gender-chart-area">
          {#if selfMadeDataFiltered.length > 0}
            <SelfMadePieChart data={selfMadeDataFiltered} />
            <div class="gender-insight">{selfMadeInsight}</div>
          {:else}
            <p class="loading-text">Carregando dados de self-made...</p>
          {/if}
        </div>
      </div>
    </div>
  </section>

  <section class="story-section">
    <div class="text-content centered-content">
        <a href="{base}/trajetoria" class="nav-button-container" on:click={navigateToTrajectory}>
          <div class="nav-button-text">
            Para você mesmo responder os questionamentos levantados, preparamos algumas visualizações cativantes.
          </div>
          <div class="nav-arrow-button">
            <span>&rarr;</span>
          </div>
        </a>
    </div>
  </section>
</div>

<style>
.gender-chart-flex-col {
  display: flex;
  flex-direction: column;
  align-items: stretch;
  width: 100%;
}
.gender-filter-container-side {
  display: flex;
  align-items: center;
  gap: 10px;
  justify-content: flex-end;
  margin-bottom: 10px;
  width: 100%;
}
.gender-chart-area {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
}
.gender-insight {
  margin-top: 14px;
  font-size: 1.1em;
  color: #ffd700;
  background: #232323;
  border-radius: 6px;
  padding: 8px 14px;
  display: inline-block;
}
.country-chart-flex-col {
  display: flex;
  flex-direction: column;
  align-items: stretch;
  width: 100%;
}
.country-chart-text {
  max-width: 900px;
  margin: 0 auto 18px auto;
  text-align: center;
}
.country-chart-area {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
}
.country-insight {
  margin-top: 14px;
  font-size: 1.1em;
  color: #ffd700;
  background: #232323;
  border-radius: 6px;
  padding: 8px 14px;
  display: inline-block;
}
</style>
