<script>
  import AgeHistogram from '$lib/components/AgeHistogram.svelte';
  import CountryBarChart from '$lib/components/CountryBarChart.svelte';
  import GenderChart from '$lib/components/GenderChart.svelte';
  import SunburstChart from '$lib/components/SunburstChart.svelte';
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
      
    } catch (error) {
      console.error("Error processing data:", error);
      loadingError = error.message;
    } finally {
      isLoading = false;
    }
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
          <div class="section-icon small">🌎</div>
          <h3>Concentração por País</h3>
        </div>
        <div class="card-content reverse">
          <div class="chart-area">
            {#if countryData.length > 0}
              <CountryBarChart data={countryData} />
            {:else}
              <p class="no-data">Dados indisponíveis</p>
            {/if}
          </div>
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

<style>
  :global(body) {
    margin: 0;
    padding: 0;
    background: linear-gradient(135deg, #f8f9fa 0%, #e9ecef 100%);
    color: #343a40;
    font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
    line-height: 1.6;
  }

  .page-container {
    max-width: 1200px;
    margin: 0 auto;
    padding: 0 16px 32px;
  }
  
  .main-nav {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 24px 0;
    margin-bottom: 0;
    border-bottom: 1px solid rgba(0,0,0,0.1);
  }
  
  .site-title {
    font-size: 1.6em;
    font-weight: 700;
    color: #1a1a1a;
    margin: 0;
    background: linear-gradient(90deg, #2b6cb0 0%, #4299e1 100%);
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    background-clip: text;
  }
  
  .nav-links {
    display: flex;
    gap: 16px;
  }
  
  .nav-link {
    text-decoration: none;
    color: #4a5568;
    font-weight: 500;
    padding: 8px 16px;
    border-radius: 8px;
    transition: all 0.2s ease;
    font-size: 0.95rem;
    letter-spacing: 0.3px;
  }
  
  .nav-link:hover {
    background-color: #edf2f7;
    color: #2b6cb0;
    transform: translateY(-1px);
  }
  
  .nav-link.active {
    background-color: #ebf8ff;
    color: #2b6cb0;
    box-shadow: 0 1px 3px rgba(0,0,0,0.1);
  }

  .hero {
    display: flex;
    justify-content: center;
    align-items: center;
    text-align: center;
    padding: 60px 20px;
    background: linear-gradient(135deg, #ebf8ff 0%, #bee3f8 100%);
    border-radius: 0 0 24px 24px;
    margin-bottom: 40px;
    box-shadow: 0 4px 12px rgba(0,0,0,0.08);
  }

  .hero-content {
    max-width: 800px;
  }

  .hero h1 {
    font-size: 3rem;
    font-weight: 800;
    margin: 0 0 16px;
    color: #2d3748;
    letter-spacing: -0.5px;
    line-height: 1.2;
    background: linear-gradient(90deg, #2b6cb0 0%, #4299e1 100%);
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    background-clip: text;
  }

  .hero p {
    font-size: 1.2rem;
    color: #4a5568;
    margin: 0;
    max-width: 700px;
    margin: 0 auto;
  }

  .loading-container, .error-container {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    padding: 60px 20px;
    text-align: center;
    background: white;
    border-radius: 16px;
    box-shadow: 0 4px 12px rgba(0,0,0,0.08);
    margin-bottom: 40px;
  }

  .loading-spinner {
    width: 40px;
    height: 40px;
    border: 4px solid rgba(66, 153, 225, 0.2);
    border-radius: 50%;
    border-top-color: #4299e1;
    animation: spin 1s linear infinite;
    margin-bottom: 20px;
  }

  @keyframes spin {
    to { transform: rotate(360deg); }
  }

  .error-icon {
    font-size: 3rem;
    margin-bottom: 16px;
    color: #e53e3e;
  }

  .action-button {
    display: inline-flex;
    align-items: center;
    background: #4299e1;
    color: white;
    font-weight: 600;
    padding: 10px 20px;
    border-radius: 8px;
    border: none;
    cursor: pointer;
    margin-top: 16px;
    transition: all 0.2s ease;
    box-shadow: 0 2px 6px rgba(66, 153, 225, 0.3);
  }

  .action-button:hover {
    background: #3182ce;
    transform: translateY(-2px);
    box-shadow: 0 4px 10px rgba(66, 153, 225, 0.4);
  }

  .button-icon {
    margin-right: 8px;
  }

  .content-section {
    display: flex;
    flex-direction: column;
    gap: 40px;
    margin-bottom: 40px;
  }

  .section-intro {
    text-align: center;
    max-width: 800px;
    margin: 0 auto 30px;
  }

  .section-intro h2 {
    font-size: 2em;
    color: #2d3748;
    margin: 14px 0 12px;
    font-weight: 700;
  }

  .section-intro p {
    font-size: 1.1em;
    color: #4a5568;
    margin: 0;
  }

  .section-icon {
    font-size: 2rem;
    background: #ebf8ff;
    color: #2b6cb0;
    width: 60px;
    height: 60px;
    display: flex;
    align-items: center;
    justify-content: center;
    border-radius: 12px;
    margin: 0 auto;
    box-shadow: 0 2px 8px rgba(66, 153, 225, 0.2);
  }

  .section-icon.small {
    width: 40px;
    height: 40px;
    font-size: 1.5rem;
    margin: 0;
  }

  .visualization-card {
    background: white;
    border-radius: 16px;
    overflow: hidden;
    box-shadow: 0 4px 20px rgba(0,0,0,0.08);
    border: 1px solid rgba(0,0,0,0.1);
    transition: transform 0.3s ease, box-shadow 0.3s ease;
  }

  .visualization-card:hover {
    transform: translateY(-5px);
    box-shadow: 0 8px 30px rgba(0,0,0,0.12);
  }

  .card-header {
    padding: 20px 24px;
    border-bottom: 1px solid rgba(0,0,0,0.1);
    display: flex;
    align-items: center;
    gap: 16px;
  }

  .card-header h3 {
    font-size: 1.5em;
    color: #2d3748;
    margin: 0;
    font-weight: 600;
  }

  .card-content {
    display: flex;
    min-height: 400px;
  }

  .card-content.reverse {
    flex-direction: row-reverse;
  }

  .text-area, .chart-area {
    flex: 1;
    padding: 30px;
    display: flex;
    flex-direction: column;
    justify-content: center;
  }

  .text-area {
    background: #fff;
  }

  .chart-area {
    background: linear-gradient(135deg, #f8fafc 0%, #f1f5f9 100%);
    border-left: 1px solid rgba(0,0,0,0.05);
    min-height: 320px;
  }

  .card-content.reverse .chart-area {
    border-left: none;
    border-right: 1px solid rgba(0,0,0,0.05);
  }

  .text-area p {
    font-size: 1.05em;
    margin-bottom: 16px;
    color: #4a5568;
    line-height: 1.7;
  }

  .insight-box {
    margin-top: 20px;
    padding: 20px;
    background: linear-gradient(135deg, #f8f9fa 0%, rgba(235, 248, 255, 0.5) 100%);
    border-radius: 12px;
    display: flex;
    gap: 15px;
    border: 1px solid rgba(66, 153, 225, 0.2);
    box-shadow: 0 2px 8px rgba(0,0,0,0.05);
  }

  .insight-icon {
    font-size: 1.8rem;
    margin-top: -2px;
  }

  .insight-content p {
    margin: 0;
    font-size: 0.95rem;
    color: #4a5568;
  }

  .explore-more {
    background: white;
    padding: 40px;
    border-radius: 16px;
    margin-bottom: 40px;
    text-align: center;
    box-shadow: 0 4px 20px rgba(0,0,0,0.08);
    border: 1px solid rgba(0,0,0,0.1);
  }

  .explore-more h2 {
    font-size: 1.8em;
    color: #2d3748;
    margin: 0 0 16px;
    font-weight: 700;
  }

  .explore-more p {
    font-size: 1.1em;
    color: #4a5568;
    margin-bottom: 24px;
  }

  .explore-buttons {
    display: flex;
    flex-wrap: wrap;
    justify-content: center;
    gap: 16px;
  }
  
  .explore-button {
    display: inline-flex;
    align-items: center;
    background: linear-gradient(135deg, #3182ce 0%, #2b6cb0 100%);
    color: white;
    font-weight: 600;
    padding: 14px 28px;
    border-radius: 8px;
    text-decoration: none;
    transition: all 0.3s ease;
    box-shadow: 0 4px 14px rgba(43, 108, 176, 0.25);
  }
  
  .explore-button:hover {
    transform: translateY(-2px);
    box-shadow: 0 6px 20px rgba(43, 108, 176, 0.35);
  }
  
  .arrow {
    margin-left: 8px;
    font-size: 1.2em;
    transition: transform 0.2s ease;
  }
  
  .explore-button:hover .arrow {
    transform: translateX(4px);
  }

  .site-footer {
    margin-top: 60px;
    text-align: center;
    padding: 24px 0;
    border-top: 1px solid rgba(0,0,0,0.1);
  }

  .footer-content p {
    margin: 0;
    font-size: 0.9rem;
    color: #718096;
  }

  .no-data {
    display: flex;
    justify-content: center;
    align-items: center;
    height: 100%;
    color: #a0aec0;
    font-style: italic;
  }

  @media (max-width: 900px) {
    .main-nav {
      flex-direction: column;
      gap: 16px;
      text-align: center;
      padding: 20px 0;
    }
    
    .hero {
      padding: 40px 16px;
    }
    
    .hero h1 {
      font-size: 2.2rem;
    }
    
    .hero p {
      font-size: 1rem;
    }
    
    .card-content, .card-content.reverse {
      flex-direction: column;
    }
    
    .text-area, .chart-area {
      padding: 20px;
    }
    
    .chart-area {
      border-left: none;
      border-top: 1px solid rgba(0,0,0,0.05);
    }
    
    .card-content.reverse .chart-area {
      border-right: none;
      border-top: 1px solid rgba(0,0,0,0.05);
    }
    
    .explore-more {
      padding: 30px 20px;
    }
    
    .insight-box {
      flex-direction: column;
      gap: 10px;
      padding: 16px;
    }
  }

  @media (max-width: 600px) {
    .page-container {
      padding: 0 12px 20px;
    }
    
    .hero h1 {
      font-size: 1.8rem;
    }
    
    .section-intro h2, .card-header h3, .explore-more h2 {
      font-size: 1.4em;
    }
    
    .text-area p, .explore-more p {
      font-size: 1em;
    }
    
    .site-title {
      font-size: 1.3em;
    }
    
    .card-header {
      padding: 16px;
    }
    
    .explore-buttons {
      flex-direction: column;
    }
    
    .explore-button {
      width: 100%;
      justify-content: center;
    }
    
    .section-icon {
      width: 50px;
      height: 50px;
      font-size: 1.6rem;
    }
    
    .section-icon.small {
      width: 36px;
      height: 36px;
      font-size: 1.3rem;
    }
    
    .visualization-card {
      margin-bottom: 20px;
    }
  }
</style>
