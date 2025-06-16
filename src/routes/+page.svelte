<script>
  import AgeHistogram from '$lib/components/AgeHistogram.svelte';
  import CountryBarChart from '$lib/components/CountryBarChart.svelte';
  import GenderChart from '$lib/components/GenderChart.svelte';
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import { base } from '$app/paths';
  import '../app.css';

  let allData = [];
  let ageData = [];
  let countryData = [];
  let genderData = [];

  // Função executada quando o componente é montado
  // Carrega os dados do CSV e inicializa as visualizações
  onMount(async () => {
    const csvPath = `${base}/Billionaires Statistics Dataset.csv`;
    const rawData = await d3.csv(csvPath);
    
    // Converte os dados brutos para o formato adequado
    allData = rawData.map(d => ({
      ...d,
      age: +d.age,
      finalWorth: +d.finalWorth,
      selfMade: d.selfMade === 'TRUE' || d.selfMade === 'True' || d.selfMade === 'true' || d.selfMade === true
    }));

    // Processa os dados de idade agrupando em faixas etárias de 10 anos
    ageData = d3.rollups(allData.filter(d => d.age && d.age > 0), v => v.length, d => Math.floor(d.age / 10) * 10)
                .map(([key, value]) => ({ ageGroup: `${key}-${key+9}`, count: value }))
                .sort((a,b) => parseInt(a.ageGroup) - parseInt(b.ageGroup));

    // Processa os dados por país com melhor validação e limpeza
    const validCountryData = allData.filter(d => d.country && d.country.trim() !== '');
    
    // Normaliza nomes de países para consistência
    const normalizedData = validCountryData.map(d => ({
      ...d,
      country: d.country.trim() // Remove espaços extras
    }));
    
    countryData = d3.rollups(normalizedData, v => v.length, d => d.country)
                    .map(([key, value]) => ({ 
                      country: key, 
                      count: value,
                      // Adiciona informações extras para debug
                      totalWealth: d3.sum(normalizedData.filter(p => p.country === key), p => +p.finalWorth || 0)
                    }))
                    .sort((a,b) => b.count - a.count)
                    .slice(0, 15); // Aumentado para 15 para incluir mais países

    // Processa os dados por gênero
    genderData = d3.rollups(allData.filter(d => d.gender), v => v.length, d => d.gender)
                   .map(([key, value]) => ({ gender: key || 'Unknown', count: value }));

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

</script>

<svelte:head>
  <title>Desvendando o Perfil Bilionário</title>
</svelte:head>

<style>
  :global(h1, h2, h3) {
    color: #ffd700 !important;
  }

  .scroll-hint-box {
    margin: 32px auto 0 auto;
    max-width: 480px;
    background: #181818;
    border-left: 5px solid #ffd700;
    color: #ffd700;
    padding: 18px 22px;
    border-radius: 8px;
    font-size: 1.1em;
    display: flex;
    align-items: center;
    gap: 12px;
    box-shadow: 0 2px 8px rgba(0,0,0,0.15);
    text-align: left;
  }
</style>

<div id="scroll-telling-page" class="story-container">
  <section class="story-section full-page-section">
    <div class="text-content centered-content">
      <h1>Análise Global de Bilionários</h1>
      <p>
        Estamos vivendo um momento singular em nossa história: pela primeira vez, observamos uma fuga en masse de bilionários dos Estados Unidos para outros países, especialmente para seu principal rival geopolítico, a China. Por outro lado, desde a pandemia, a disparidade de riqueza entre as classes sociais se aprofundou a tal ponto que, mesmo entre os bilionários, já se pode identificar uma elite ainda mais exclusiva.
      </p>
      <p>
        Diante disso, torna-se evidente que a estrutura interna dessas camadas ultrarricas é complexa. Nosso objetivo, portanto, é traçar paralelos com o mundo que conhecemos para, por meio do contraste, compreendermos melhor a dinâmica desse segmento exclusivo da sociedade.
      </p>
      <div class="scroll-hint-box">
        <span><strong>Observação:</strong> Role para baixo para continuar explorando o site!</span>
      </div>
    </div>
  </section>

  <section class="story-section side-by-side">
    <div class="text-content left">
      <h2>Gênero e a Elite Financeira: Um Clube Exclusivo?</h2>
      <p>
        A primeira característica dessa classe é a disparidade de patrimônio entre os sexos. Em geral, a proporção de bilionários self-made é semelhante, o que invalida a conjectura de que muitas mulheres, diferentemente dos homens, herdaram suas fortunas. 
      </p>
      <p>
        Ademais, observamos que tanto os Estados Unidos quanto a China lideram os rankings para ambos os sexos, o que evidencia que há um tratamento relativamente equilibrado, especialmente na China, onde essa transformação teve início ainda no regime de Mao Tsé-Tung, com sua famosa frase: "As mulheres sustentam metade do céu."
      </p>
    </div>
    <div class="chart-content right">
      {#if genderData.length > 0}
        <GenderChart data={genderData} allData={allData} />
      {:else}
        <p class="loading-text">Carregando dados de gênero...</p>
      {/if}
    </div>
  </section>

  <section class="story-section side-by-side">
    <div class="chart-content left">
      {#if countryData.length > 0}
        <CountryBarChart data={countryData} allData={allData} />
      {:else}
        <p class="loading-text">Carregando dados de país...</p>
      {/if}
    </div>
    <div class="text-content right">
      <h2>A Geografia da Riqueza Extrema: Onde o Capital se Concentra?</h2>
      <p>
        Seguindo nossa narrativa, não é surpreendente que a maior concentração de bilionários resida nas duas maiores potências econômicas mundiais. Ainda assim, observamos que a Europa também possui um destaque significativo. No geral, percebe-se que grande parte do continente asiático, da África e da América Latina não está bem representada.
      </p>
      <p>
        Algumas indagações podem ser levantadas: será que isso está relacionado à fuga desses indivíduos para outros países? E quais são as características desses países de destino?
      </p>

    </div>
  </section>

  <section class="story-section side-by-side">
    <div class="text-content left">
      <h2>A Idade da Riqueza: Quando o Capital Atinge o Ápice?</h2>
      <p>
        Por fim, vale salientar que, em geral, a faixa etária de pico de riqueza ocorre por volta dos 65 anos. No entanto, esse padrão não se mantém uniforme em todos os setores. Em particular, o setor de tecnologia se destaca por sua grande assimetria: é conhecido por gerar magnatas praticamente da noite para o dia.
      </p>
    </div>
    <div class="chart-content right">
      {#if allData.length > 0}
        <AgeHistogram data={allData} />
      {:else}
        <p class="loading-text">Carregando dados de idade...</p>
      {/if}
    </div>
  </section>

  <section class="story-section">
    <div class="text-content centered-content">
        <div class="navigation-section">
          <h3>Continue a Jornada</h3>
          <p>Para você mesmo responder os questionamentos levantados, preparamos algumas visualizações cativantes.</p>
          <a href="{base}/trajetoria" class="nav-button-container enhanced-nav-button" on:click={navigateToTrajectory} data-sveltekit-preload-data="false" data-sveltekit-preload-code="false">
                         <div class="nav-content">
               <div class="nav-text-group">
                 <div class="nav-title">Explorar Fluxos Migratórios</div>
                 <div class="nav-subtitle">Descubra para onde os bilionários estão migrando</div>
               </div>
               <div class="nav-arrow-button">
                 <span>&rarr;</span>
               </div>
             </div>
          </a>
          <div class="progress-indicator">
            <div class="progress-step active">1</div>
            <div class="progress-line"></div>
            <div class="progress-step">2</div>
            <div class="progress-line"></div>
            <div class="progress-step">3</div>
          </div>
        </div>
    </div>
  </section>
</div>
