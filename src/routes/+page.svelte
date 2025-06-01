<script>
  import AgeHistogram from '$lib/components/AgeHistogram.svelte';
  import CountryBarChart from '$lib/components/CountryBarChart.svelte';
  import GenderChart from '$lib/components/GenderChart.svelte';
  import SelfMadeBarChart from '$lib/components/SelfMadeBarChart.svelte';
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import { base } from '$app/paths';
  import '../app.css';

  let allData = [];
  let ageData = [];
  let countryData = [];
  let genderData = [];
  let selfMadeData = [];

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

    // Processa os dados por país, pegando os 10 países com mais bilionários
    countryData = d3.rollups(allData.filter(d => d.country), v => v.length, d => d.country)
                    .map(([key, value]) => ({ country: key, count: value }))
                    .sort((a,b) => b.count - a.count)
                    .slice(0, 10);

    // Processa os dados por gênero
    genderData = d3.rollups(allData.filter(d => d.gender), v => v.length, d => d.gender)
                   .map(([key, value]) => ({ gender: key || 'Unknown', count: value }));

    // Processa os dados de self-made
    const selfMadeCounts = d3.rollups(allData.filter(d => d.selfMade !== undefined), v => v.length, d => d.selfMade);
    selfMadeData = selfMadeCounts.map(([isSelfMade, count]) => ({
      label: isSelfMade ? 'Self-Made' : 'Não Self-Made',
      value: count
    }));

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
      {#if genderData.length > 0}
        <GenderChart data={genderData} />
      {:else}
        <p class="loading-text">Carregando dados de gênero...</p>
      {/if}
    </div>
  </section>

  <section class="story-section side-by-side">
    <div class="chart-content left">
      {#if countryData.length > 0}
        <CountryBarChart data={countryData} />
      {:else}
        <p class="loading-text">Carregando dados de país...</p>
      {/if}
    </div>
    <div class="text-content right">
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
      {#if ageData.length > 0}
        <AgeHistogram data={ageData} />
      {:else}
        <p class="loading-text">Carregando dados de idade...</p>
      {/if}
    </div>
  </section>

  <section class="story-section side-by-side">
    <div class="chart-content left">
      {#if selfMadeData.length > 0}
        <SelfMadeBarChart data={selfMadeData} />
      {:else}
        <p class="loading-text">Carregando dados de origem da fortuna...</p>
      {/if}
    </div>
    <div class="text-content right">
      <h2>Fortuna Conquistada ou Herdada: A Origem do Poder Econômico</h2>
      <p>
        Esta análise desvenda uma questão fundamental: quantos bilionários construíram suas 
        fortunas do zero versus aqueles que nasceram em berços de ouro? A distinção entre 
        "self-made" e herdeiros revela dinâmicas sociais profundas.
      </p>
      <p>
        Os números questionam narrativas sobre meritocracia e oportunidades iguais. 
        Será que o "sonho do empreendedor" é realmente acessível a todos, ou existe 
        um padrão hereditário na concentração de riqueza extrema?
      </p>
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
