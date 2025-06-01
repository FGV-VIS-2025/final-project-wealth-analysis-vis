<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import { base } from '$app/paths';
  import BilSearch from '$lib/components/BilSearch.svelte';
  import '../../app.css';
  import './search.css';

  let allData = [];
  let isLoading = true;

  // Mapeamento de países para continentes para facilitar a filtragem regional
  const continentMap = {
    "United States": "América do Norte",
    "China": "Ásia",
    "India": "Ásia",
    "Germany": "Europa",
    "Russia": "Europa", 
    "Hong Kong": "Ásia",
    "Brazil": "América do Sul",
    "Canada": "América do Norte",
    "France": "Europa",
    "Italy": "Europa",
    "Australia": "Oceania",
    "Japan": "Ásia",
    "Switzerland": "Europa",
    "Sweden": "Europa",
    "United Kingdom": "Europa",
    "Singapore": "Ásia",
    "South Korea": "Ásia",
    "Israel": "Ásia",
    "Spain": "Europa",
    "Mexico": "América do Norte",
    "Indonesia": "Ásia",
    "Turkey": "Ásia", 
    "Thailand": "Ásia",
    "Saudi Arabia": "Ásia",
    "Netherlands": "Europa",
    "Taiwan": "Ásia",
    "Ireland": "Europa",
    "Chile": "América do Sul",
  };

  // Função executada quando o componente é montado
  // Carrega os dados dos bilionários do CSV e prepara para a busca
  onMount(async () => {
    try {
      const csvPath = `${base}/Billionaires Statistics Dataset.csv`;
      const rawData = await d3.csv(csvPath);
      
      // Processa os dados brutos, convertendo tipos e filtrando entradas inválidas
      allData = rawData.map(d => ({
        ...d,
        age: +d.age || 0,
        finalWorth: +d.finalWorth || 0,
        selfMade: d.selfMade === 'TRUE' || d.selfMade === 'True' || d.selfMade === 'true' || d.selfMade === true,
        birthYear: +d.birthYear || 0,
        rank: +d.rank || null
      })).filter(d => d.personName && d.finalWorth > 0);

      isLoading = false;
    } catch (error) {
      console.error('Erro ao carregar dados para a busca:', error);
      isLoading = false;
    }

    // Torna a página visível após o carregamento (para animação de entrada)
    const pageContainer = document.getElementById('search-page-container');
    if (pageContainer) {
        pageContainer.classList.add('visible');
    }
  });

  // Função para navegar para a página de trajetória
  async function navigateToTrajectory(event) {
    event.preventDefault();
    const targetHref = event.currentTarget.href;
    window.location.href = targetHref;
  }
</script>

<svelte:head>
  <title>Ferramenta de Busca de Bilionários</title>
</svelte:head>

<div class="page-container" id="search-page-container">
  <section class="search-content-section">
    <div class="content-wrapper-side-by-side">
      <div class="text-column">
        <h1>Explore os Dados dos Bilionários</h1>
        <p class="intro-text">
          Ao longo desta análise, buscamos desvendar as múltiplas facetas que definem o perfil dos bilionários. Mergulhamos nos dados para ir além dos números, estimulando um olhar crítico sobre a impressionante concentração de riqueza e os padrões complexos que emergem.
        </p>
        <p class="intro-text">
          As tendências e observações que apresentamos aqui servem como um ponto de partida, um panorama inicial que revela tanto a magnitude quanto as nuances deste estrato econômico. Compreendemos que, embora os agregados e as médias ofereçam insights valiosos, a verdadeira profundidade da compreensão muitas vezes reside nos detalhes e na capacidade de investigar questões específicas.
        </p>
        <p class="intro-text">
          Se as nossas conclusões despertaram sua curiosidade, se você deseja investigar um setor industrial específico, comparar a riqueza em diferentes regiões ou simplesmente filtrar os dados para satisfazer seu próprio questionamento, o poder agora está em suas mãos. Encorajamos você a utilizar os filtros, cruzar informações, testar hipóteses e, quem sabe, descobrir novos padrões que complementem ou até mesmo desafiem o que foi apresentado.
        </p>
      </div>
      <div class="search-tool-column">
        {#if isLoading}
          <div class="loading-data-message">
            <div class="loading-spinner"></div>
            <p>Carregando dados dos bilionários...</p>
          </div>
        {:else if allData.length > 0}
          <BilSearch billionaires={allData} {continentMap} />
        {:else}
          <p>Não foi possível carregar os dados dos bilionários.</p>
        {/if}
      </div>
    </div>
  </section>

  <section class="search-navigation-section">
    <a href="{base}/trajetoria" class="nav-button-container back-button" on:click={navigateToTrajectory}>
        <div class="nav-arrow-button back-arrow-icon">
        <span>&larr;</span> 
        </div>
        <div class="nav-button-text">
        Voltar para Trajetória
        </div>
    </a>
  </section>
</div> 