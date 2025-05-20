<script>
  import DumbbellChart from '$lib/components/DumbbellChart.svelte';
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import { base } from '$app/paths';
  import { fly, fade } from 'svelte/transition';

  let allData = [];
  let genderGapData = [];
  let isLoading = true;
  let loadingError = null;

  // Map different gender codes to standardized format
  function mapGenderCode(code) {
    if (!code) return 'Unknown';
    
    // Convert to uppercase for consistent comparison
    const upperCode = String(code).toUpperCase();
    
    if (upperCode === 'M' || upperCode === 'MALE' || upperCode === 'MAN') return 'M';
    if (upperCode === 'F' || upperCode === 'FEMALE' || upperCode === 'WOMAN') return 'F';
    
    return 'Unknown';
  }
  
  async function loadData() {
    try {
      isLoading = true;
      loadingError = null;
      
      // Use encodeURI to handle spaces in the filename
      const csvPath = `${base}/Billionaires Statistics Dataset.csv`;
      console.log("Loading CSV from:", csvPath);
      
      const response = await fetch(csvPath);
      if (!response.ok) {
        throw new Error(`Failed to load CSV: ${response.status} ${response.statusText}`);
      }
      
      const csvText = await response.text();
      console.log("CSV loaded, length:", csvText.length);
      
      // Debug the first few lines of the CSV
      console.log("CSV first few lines:", csvText.split('\n').slice(0, 3).join('\n'));
      
      if (!csvText || csvText.length === 0) {
        throw new Error("CSV file is empty");
      }
      
      const rawData = d3.csvParse(csvText);
      console.log("Raw data parsed, count:", rawData.length);
      
      if (!rawData || rawData.length === 0) {
        throw new Error("Failed to parse CSV data");
      }
      
      // Log column names to debug
      console.log("CSV columns:", rawData.columns);
      
      // Simple processing - use columns directly
      allData = rawData.map(d => ({
        country: d.country || '',
        gender: mapGenderCode(d.gender),
        finalWorth: +d.finalWorth || 0
      })).filter(d => d.country && d.gender !== 'Unknown' && d.finalWorth > 0);
      
      console.log("Processed data count:", allData.length);
      console.log("Sample processed data:", allData.slice(0, 5));
      
      // Simple way - collect all data points by country and gender
      const malesByCountry = {};
      const femalesByCountry = {};
      
      // Group by country and gender
      allData.forEach(d => {
        const country = d.country;
        const worth = d.finalWorth;
        
        if (d.gender === 'M') {
          if (!malesByCountry[country]) malesByCountry[country] = [];
          malesByCountry[country].push(worth);
        } else if (d.gender === 'F') {
          if (!femalesByCountry[country]) femalesByCountry[country] = [];
          femalesByCountry[country].push(worth);
        }
      });
      
      // Calculate averages and create the dataset
      const tempData = [];
      
      // Process countries with data for both genders
      const allCountries = new Set([...Object.keys(malesByCountry), ...Object.keys(femalesByCountry)]);
      
      allCountries.forEach(country => {
        const maleData = malesByCountry[country] || [];
        const femaleData = femalesByCountry[country] || [];
        
        // Only include if we have at least some data
        if (maleData.length > 0 || femaleData.length > 0) {
          const maleAvg = maleData.length > 0 ? d3.mean(maleData) : 0;
          const femaleAvg = femaleData.length > 0 ? d3.mean(femaleData) : 0;
          
          tempData.push({
            country,
            maleAvg,
            femaleAvg,
            gap: maleAvg - femaleAvg,
            maleCount: maleData.length,
            femaleCount: femaleData.length
          });
        }
      });
      
      // Sort by gap and filter for countries with both genders or significant data
      genderGapData = tempData
        .filter(d => (d.maleCount > 0 && d.femaleCount > 0) || d.maleCount > 5 || d.femaleCount > 5)
        .sort((a, b) => Math.abs(b.gap) - Math.abs(a.gap));
      
      console.log("Final gender gap data:", genderGapData.length);
      console.log("Sample gender gap data:", genderGapData.slice(0, 3));
      
      if (genderGapData.length === 0) {
        throw new Error("No gender gap data after processing");
      }
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
  <title>Disparidade de Riqueza entre Gêneros | Análise de Bilionários</title>
  <meta name="description" content="Visualização da disparidade de riqueza entre bilionários homens e mulheres em diferentes países" />
</svelte:head>

<div class="page-container">
  <nav class="main-nav">
    <h1 class="site-title">Análise Global de Bilionários</h1>
    <div class="nav-links">
      <a href="{base}/" class="nav-link">Visão Geral</a>
      <a href="{base}/gender-gap" class="nav-link active">Gap de Gênero</a>
      <a href="{base}/map" class="nav-link">Mapa de Migração</a>
      <a href="{base}/network" class="nav-link">Rede de Indústrias</a>
    </div>
  </nav>

  <section class="hero">
    <div class="hero-content">
      <h1 in:fly="{{ y: 20, duration: 800, delay: 200 }}">Disparidade de Riqueza entre Gêneros</h1>
      <p in:fly="{{ y: 20, duration: 800, delay: 400 }}">Uma análise visual das diferenças de patrimônio entre bilionários homens e mulheres ao redor do mundo</p>
    </div>
  </section>

  <section class="content-section" in:fade="{{ duration: 800, delay: 600 }}">
    <div class="text-container">
      <div class="section-header">
        <div class="section-icon">📊</div>
        <h2>Análise Comparativa</h2>
      </div>
      <p>
        Esta visualização ilustra a diferença entre o patrimônio médio de bilionários 
        homens e mulheres em diferentes países. Cada linha representa um país, com 
        o ponto vermelho indicando a média de riqueza das mulheres bilionárias e o 
        ponto azul representando a média dos homens.
      </p>
      <p>
        Os países estão ordenados pelo maior gap entre gêneros, destacando onde 
        as disparidades são mais significativas. O gráfico mostra os 15 países 
        com as maiores disparidades, permitindo identificar padrões globais sem poluição visual.
      </p>
      
      <div class="insight-box">
        <div class="insight-icon">💡</div>
        <div class="insight-content">
          <h3>Insights Principais</h3>
          <p>As disparidades financeiras entre gêneros persistem mesmo no topo da pirâmide econômica, 
          refletindo barreiras históricas ao acesso feminino à acumulação de riqueza, incluindo limitações 
          no controle de ativos, herança e oportunidades empresariais.</p>
        </div>
      </div>
    </div>

    <div class="chart-container">
      {#if isLoading}
        <div class="loading">
          <div class="loading-spinner"></div>
          <p>Carregando dados...</p>
        </div>
      {:else if loadingError}
        <div class="error">
          <div class="error-icon">⚠️</div>
          <p>Erro ao carregar os dados: {loadingError}</p>
          <p class="debug-info">Verifique se o arquivo CSV está disponível e contém os dados necessários.</p>
          <div class="debug-buttons">
            <button class="debug-button" on:click={async () => {
              try {
                isLoading = true;
                loadingError = null;
                
                const csvPath = `${base}/Billionaires Statistics Dataset.csv`;
                const response = await fetch(csvPath);
                
                if (!response.ok) {
                  throw new Error(`HTTP error: ${response.status}`);
                }
                
                const text = await response.text();
                console.log("CSV content (first 500 chars):", text.substring(0, 500));
                console.log("CSV rows:", text.split("\n").length);
                
                alert(`CSV carregado com sucesso! ${text.split("\n").length} linhas encontradas.`);
              } catch (err) {
                console.error("Teste falhou:", err);
                alert(`Erro ao testar CSV: ${err.message}`);
                loadingError = `Erro de teste: ${err.message}`;
              } finally {
                isLoading = false;
              }
            }}>Testar acesso ao CSV</button>
            
            <button class="debug-button primary" on:click={() => loadData()}>
              <span class="button-icon">🔄</span> Tentar carregar novamente
            </button>
          </div>
        </div>
      {:else if !genderGapData || genderGapData.length === 0}
        <div class="no-data">
          <div class="no-data-icon">📈</div>
          <p>Não há dados suficientes para exibir esta visualização.</p>
          <p class="debug-info">Verifique se o arquivo CSV está disponível e contém os dados necessários.</p>
          <div class="debug-buttons">
            <button class="debug-button primary" on:click={() => loadData()}>
              <span class="button-icon">🔄</span> Tentar carregar novamente
            </button>
          </div>
        </div>
      {:else}
        <div class="chart-wrapper" in:fade="{{ duration: 1000, delay: 300 }}">
          <DumbbellChart data={genderGapData} />
        </div>
      {/if}
    </div>
    
    <div class="methodology-section">
      <div class="section-header">
        <div class="section-icon">🔍</div>
        <h2>Metodologia</h2>
      </div>
      <p>
        Os dados apresentados são baseados na análise do patrimônio médio de bilionários por país e gênero. 
        Para garantir a relevância estatística, são exibidos apenas países com presença significativa de bilionários 
        (pelo menos 5 de um gênero ou presença de ambos os gêneros). 
        O valor apresentado representa a média do patrimônio em bilhões de dólares americanos.
      </p>
    </div>
  </section>
  
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

  .content-section {
    display: flex;
    flex-direction: column;
    gap: 40px;
    background: #fff;
    border-radius: 16px;
    box-shadow: 0 4px 20px rgba(0,0,0,0.08);
    padding: 40px;
    border: 1px solid rgba(0,0,0,0.1);
  }

  .section-header {
    display: flex;
    align-items: center;
    margin-bottom: 20px;
    gap: 14px;
  }

  .section-icon {
    font-size: 1.8rem;
    background: #ebf8ff;
    color: #2b6cb0;
    width: 50px;
    height: 50px;
    display: flex;
    align-items: center;
    justify-content: center;
    border-radius: 12px;
  }

  .text-container {
    max-width: 800px;
    margin: 0 auto;
  }

  h2 {
    font-size: 1.8em;
    color: #2d3748;
    margin: 0 0 12px;
    font-weight: 700;
    letter-spacing: -0.5px;
  }
  
  p {
    font-size: 1.05em;
    margin-bottom: 16px;
    color: #4a5568;
    line-height: 1.7;
  }

  .insight-box {
    margin-top: 30px;
    padding: 24px;
    background: linear-gradient(135deg, #f8f9fa 0%, rgba(235, 248, 255, 0.5) 100%);
    border-radius: 12px;
    display: flex;
    gap: 20px;
    border: 1px solid rgba(66, 153, 225, 0.2);
    box-shadow: 0 2px 8px rgba(0,0,0,0.05);
  }

  .insight-icon {
    font-size: 2rem;
  }

  .insight-content h3 {
    font-size: 1.2rem;
    color: #2b6cb0;
    margin: 0 0 12px;
    font-weight: 600;
  }

  .insight-content p {
    margin: 0;
    font-size: 1rem;
  }

  .chart-container {
    height: 600px;
    width: 100%;
    margin: 0 auto;
    background: linear-gradient(135deg, #f8f9fa 0%, #f1f5f9 100%);
    border-radius: 12px;
    overflow: hidden;
    box-shadow: 0 2px 12px rgba(0,0,0,0.05);
    border: 1px solid rgba(0,0,0,0.1);
  }

  .chart-wrapper {
    width: 100%;
    height: 100%;
  }

  .loading, .no-data, .error {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    height: 100%;
    text-align: center;
    padding: 20px;
  }

  .loading {
    font-size: 1.2em;
    color: #4a5568;
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

  .error-icon, .no-data-icon {
    font-size: 3rem;
    margin-bottom: 16px;
  }

  .no-data p, .error p {
    font-size: 1.2em;
    color: #4a5568;
    margin-bottom: 10px;
  }

  .error p {
    color: #e53e3e;
  }

  .debug-info {
    font-size: 0.9em;
    color: #718096;
    max-width: 400px;
  }

  .debug-buttons {
    display: flex;
    flex-wrap: wrap;
    gap: 12px;
    margin-top: 24px;
    justify-content: center;
  }
  
  .debug-button {
    padding: 10px 18px;
    background-color: #e2e8f0;
    color: #4a5568;
    border: none;
    border-radius: 8px;
    cursor: pointer;
    font-size: 14px;
    font-weight: 500;
    transition: all 0.2s ease;
    display: flex;
    align-items: center;
    gap: 8px;
    box-shadow: 0 1px 3px rgba(0,0,0,0.1);
  }
  
  .debug-button:hover {
    background-color: #cbd5e0;
    transform: translateY(-1px);
  }

  .debug-button.primary {
    background-color: #4299e1;
    color: white;
  }

  .debug-button.primary:hover {
    background-color: #3182ce;
  }

  .button-icon {
    font-size: 16px;
  }

  .methodology-section {
    background: #f8fafc;
    padding: 30px;
    border-radius: 12px;
    border: 1px solid rgba(0,0,0,0.1);
  }

  .methodology-section p {
    margin-bottom: 0;
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
    
    .content-section {
      padding: 30px 20px;
    }
    
    .chart-container {
      height: 500px;
    }
    
    .insight-box {
      flex-direction: column;
      gap: 10px;
    }
    
    .methodology-section {
      padding: 20px;
    }
  }

  @media (max-width: 600px) {
    .page-container {
      padding: 0 12px 20px;
    }
    
    .hero h1 {
      font-size: 1.8rem;
    }
    
    h2 {
      font-size: 1.4em;
    }
    
    p {
      font-size: 1em;
    }
    
    .site-title {
      font-size: 1.3em;
    }
    
    .chart-container {
      height: 400px;
    }
    
    .section-icon {
      width: 40px;
      height: 40px;
      font-size: 1.4rem;
    }
  }
</style> 