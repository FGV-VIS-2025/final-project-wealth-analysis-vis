<script>
  import '../global.css';
  import AgeHistogram from '$lib/components/AgeHistogram.svelte';
  import CountryBarChart from '$lib/components/CountryBarChart.svelte';
  import GenderChart from '$lib/components/GenderChart.svelte';
  import SunburstChart from '$lib/components/SunburstChart.svelte';
  import BillionaireSearch from '$lib/components/BillionaireSearch.svelte';
  import SelfMadePieChart from '$lib/components/SelfMadePieChart.svelte';
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import { base } from '$app/paths';
  import { fly, fade } from 'svelte/transition';

  let allData = [];
  let ageData = [];
  let countryData = [];
  let genderData = [];
  let sunburstData = [];
  let isLoading = true;
  let loadingError = null;
  
  let continent_map = {
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

  async function loadData() {
    try {
      isLoading = true;
      loadingError = null;
      
      const csvPath = `${base}/Billionaires Statistics Dataset.csv`;
      console.log("Loading CSV from:", csvPath);
      
      const response = await fetch(csvPath);
      if (!response.ok) {
        throw new Error(`Failed to load CSV: ${response.status} ${response.statusText}`);
      }
      
      const rawData = await d3.csv(csvPath);
      console.log("Raw data loaded, count:", rawData.length);
      
      if (!rawData || rawData.length === 0) {
        throw new Error("Failed to parse CSV data");
      }
      
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
      
      sunburstData = allData.map(d => ({
        Region: continent_map[d.country] || 'Other',
        Country: d.country,
        Industry: d.industries || d.source || 'Unknown',
        NetWorth: d.finalWorth
      }));
      
    } catch (error) {
      console.error("Error processing data:", error);
      loadingError = error.message;
    } finally {
      isLoading = false;
    }
  }

  function processSelfMadeData(data) {
    return data.filter(d => d.selfMade !== undefined);
  }

  onMount(() => {
    loadData();
  });
</script>

<svelte:head>
  <title>Análise Global de Bilionários | Visão Geral</title>
  <meta name="description" content="Visualizações interativas sobre a distribuição de riqueza entre bilionários ao redor do mundo" />
</svelte:head>

<div class="page-container">
  <nav class="main-nav">
    <h1 class="site-title">Análise Global de Bilionários</h1>
    <div class="nav-links">
      <a href="{base}/" class="nav-link active">Visão Geral</a>
      <a href="{base}/gender-gap" class="nav-link">Gap de Gênero</a>
      <a href="{base}/map" class="nav-link">Mapa de Migração</a>
      <a href="{base}/network" class="nav-link">Rede de Indústrias</a>
    </div>
  </nav>

  <section class="hero">
    <div class="hero-content">
      <h1 in:fly="{{ y: 20, duration: 800, delay: 200 }}">Panorama Global de Bilionários</h1>
      <p in:fly="{{ y: 20, duration: 800, delay: 400 }}">Uma análise visual sobre a distribuição da riqueza extrema ao redor do mundo</p>
    </div>
  </section>

  {#if isLoading}
    <div class="loading-container" in:fade="{{ duration: 400 }}">
      <div class="loading-spinner"></div>
      <p>Carregando visualizações...</p>
    </div>
  {:else if loadingError}
    <div class="error-container" in:fade="{{ duration: 400 }}">
      <div class="error-icon">⚠️</div>
      <p>Erro ao carregar os dados: {loadingError}</p>
      <button class="action-button" on:click={() => loadData()}>
        <span class="button-icon">🔄</span> Tentar novamente
      </button>
    </div>
  {:else}
    <section class="content-section" in:fade="{{ duration: 800, delay: 600 }}">
      <div class="section-intro">
        <div class="section-icon">📊</div>
        <h2>Visão Multidimensional da Riqueza Global</h2>
        <p>Explore os diferentes aspectos da distribuição de bilionários ao redor do mundo através de visualizações interativas.</p>
      </div>

      <div class="visualization-card">
        <div class="card-header">
          <div class="section-icon small">💰</div>
          <h3>Distribuição da Riqueza por Setor e Região</h3>
        </div>
        <div class="card-content reverse">
          <div class="chart-area">
            {#if sunburstData.length > 0}
              <SunburstChart data={sunburstData} />
            {:else}
              <p class="no-data">Dados indisponíveis</p>
            {/if}
          </div>
          <div class="text-area">
            <p>
              O gráfico ao lado mostra como o patrimônio líquido dos bilionários está distribuído hierarquicamente por região, país e setor de atuação. 
              Setores como tecnologia, manufatura e finanças concentram a maior parte da riqueza, especialmente em países como Estados Unidos e China.
            </p>
            <div class="insight-box">
              <div class="insight-icon">💡</div>
              <div class="insight-content">
                <p>A concentração em certos setores, como tecnologia, reflete o potencial de escalabilidade e margens de lucro desses negócios, bem como a valorização diferenciada que os mercados atribuem a empresas inovadoras com alto potencial de crescimento.</p>
              </div>
            </div>
          </div>
        </div>
      </div>

      <div class="visualization-card">
        <div class="card-header">
          <div class="section-icon small">👨‍💼</div>
          <h3>Origem da Fortuna: Self-Made vs Herdeiros</h3>
        </div>
        <div class="card-content">
          <div class="text-area">
            <p>
              Este gráfico mostra a distribuição entre bilionários que construíram suas próprias fortunas (self-made) 
              e aqueles que herdaram suas riquezas. A análise revela padrões interessantes sobre a mobilidade 
              econômica e a criação de riqueza em diferentes contextos.
            </p>
            <div class="insight-box">
              <div class="insight-icon">💡</div>
              <div class="insight-content">
                <p>A proporção entre self-made e herdeiros pode indicar o nível de oportunidades econômicas e a facilidade de criar novas fortunas em diferentes períodos e regiões.</p>
              </div>
            </div>
          </div>
          <div class="chart-area">
            {#if allData.length > 0}
              <SelfMadePieChart data={allData} />
            {:else}
              <p class="no-data">Dados indisponíveis</p>
            {/if}
          </div>
        </div>
      </div>

      <div class="visualization-card">
        <div class="card-header">
          <div class="section-icon small">🌎</div>
          <h3>Concentração por País</h3>
        </div>
        <div class="card-content">
          <div class="text-area">
            <p>
              Onde no mundo residem os bilionários? Este gráfico apresenta os principais países
              com o maior número de bilionários, oferecendo uma perspectiva geográfica sobre a concentração da riqueza.
            </p>
            <div class="insight-box">
              <div class="insight-icon">💡</div>
              <div class="insight-content">
                <p>A dominância de certas nações reflete seus sistemas econômicos, políticas fiscais, ambientes de negócios e histórico de desenvolvimento industrial. Economias com forte proteção à propriedade privada e mercados de capitais desenvolvidos tendem a produzir mais bilionários.</p>
              </div>
            </div>
          </div>
          <div class="chart-area">
            {#if countryData.length > 0}
              <CountryBarChart data={countryData} />
            {:else}
              <p class="no-data">Dados indisponíveis</p>
            {/if}
          </div>
        </div>
      </div>

      <div class="visualization-card">
        <div class="card-header">
          <div class="section-icon small">👵</div>
          <h3>Distribuição por Idade</h3>
        </div>
        <div class="card-content">
          <div class="text-area">
            <p>
              A distribuição de bilionários em diferentes faixas etárias revela insights interessantes
              sobre quando a acumulação de riqueza atinge seu pico. Este histograma ilustra o número de bilionários
              em várias faixas etárias.
            </p>
            <div class="insight-box">
              <div class="insight-icon">💡</div>
              <div class="insight-content">
                <p>A concentração de bilionários em certas faixas etárias pode indicar os anos mais produtivos para o sucesso empresarial, o tempo necessário para acumular riqueza significativa, ou padrões de herança intergeracional.</p>
              </div>
            </div>
          </div>
          <div class="chart-area">
            {#if ageData.length > 0}
              <AgeHistogram data={ageData} />
            {:else}
              <p class="no-data">Dados indisponíveis</p>
            {/if}
          </div>
        </div>
      </div>

      <div class="visualization-card">
        <div class="card-header">
          <div class="section-icon small">⚧️</div>
          <h3>Distribuição por Gênero</h3>
        </div>
        <div class="card-content">
          <div class="text-area">
            <p>
              Esta seção explora a representação de diferentes gêneros na população bilionária.
              O gráfico destaca o número de bilionários identificados com várias categorias de gênero.
            </p>
            <div class="insight-box">
              <div class="insight-icon">💡</div>
              <div class="insight-content">
                <p>A disparidade de gênero na riqueza extrema reflete barreiras históricas e contínuas enfrentadas pelas mulheres no acesso a oportunidades empresariais, financiamento e redes de poder. Para uma análise mais profunda, visite a página de Gap de Gênero.</p>
              </div>
            </div>
          </div>
          <div class="chart-area">
            {#if genderData.length > 0}
              <GenderChart data={genderData} />
            {:else}
              <p class="no-data">Dados indisponíveis</p>
            {/if}
          </div>
        </div>
      </div>
    </section>

    <section class="billionaire-search-section" in:fade="{{ duration: 800, delay: 800 }}">
      {#if !isLoading && !loadingError}
        <BillionaireSearch billionaires={allData} continentMap={continent_map}/>
      {/if}
    </section>

    <section class="explore-more" in:fade="{{ duration: 800, delay: 800 }}">
      <div class="text-content centered">
        <h2>Explore Análises Específicas</h2>
        <p>Aprofunde-se em aspectos particulares da distribuição de riqueza com nossas visualizações especializadas</p>
        <div class="explore-buttons">
          <a href="{base}/gender-gap" class="explore-button">
            Disparidade de Riqueza entre Gêneros
            <span class="arrow">→</span>
          </a>
          <a href="{base}/map" class="explore-button">
            Mapa de Migração de Bilionários
            <span class="arrow">→</span>
          </a>
          <a href="{base}/network" class="explore-button">
            Rede de Bilionários por Indústria
            <span class="arrow">→</span>
          </a>
        </div>
      </div>
    </section>
  {/if}

  <footer class="site-footer">
    <div class="footer-content">
      <p>© 2023 Análise Global de Bilionários</p>
      <p>Dados baseados em estatísticas públicas de patrimônio de bilionários</p>
    </div>
  </footer>
</div>
