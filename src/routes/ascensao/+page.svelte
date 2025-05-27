<script>
  import { base } from '$app/paths';
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import SankeyDiagram from '$lib/components/SankeyDiagram.svelte';

  let sankeyData = [];
  let isLoading = true;
  let loadingError = null;
  let sankeyNodes = [];
  let sankeyLinks = [];
  let sankeyLinkColors = [];

  // Filtros múltiplos
  let origemFiltro = [];
  const opcoesOrigem = ['Self-Made', 'Herdeiro'];
  let paisFiltro = [];
  let opcoesPaises = [];
  let setorFiltro = [];
  let opcoesSetores = [];

  // Dropdown customizado para filtro de país
  let paisDropdownAberto = false;
  let paisFiltroTemp = [];
  function togglePaisDropdown() { 
    paisDropdownAberto = !paisDropdownAberto; 
    if (paisDropdownAberto) paisFiltroTemp = [...paisFiltro];
  }
  function fecharPaisDropdown() { paisDropdownAberto = false; }

  function paisSelecionadoTemp(pais) {
    return paisFiltroTemp.includes(pais);
  }

  function handlePaisCheckboxTemp(pais) {
    if (pais === 'Todos') {
      if (paisFiltroTemp.includes('Todos')) {
        paisFiltroTemp = [];
      } else {
        paisFiltroTemp = ['Todos'];
      }
    } else {
      if (paisFiltroTemp.includes('Todos')) {
        paisFiltroTemp = [pais];
      } else {
        if (paisFiltroTemp.includes(pais)) {
          paisFiltroTemp = paisFiltroTemp.filter(p => p !== pais);
        } else {
          if (paisFiltroTemp.length < 15) {
            paisFiltroTemp = [...paisFiltroTemp, pais];
          }
        }
      }
    }
  }

  $: if (paisFiltroTemp.length > 15) {
    paisFiltroTemp = paisFiltroTemp.slice(0, 15);
  }

  function aplicarFiltroPais() {
    paisFiltro = [...paisFiltroTemp];
    fecharPaisDropdown();
  }

  // Lista de países para o dropdown: 'Todos', top 15, 'Outros', depois todos os outros em ordem alfabética
  $: listaPaisesDropdown = (() => {
    let todosPaises = Array.from(new Set(sankeyData.map(d => d.pais))).sort();
    let top15 = opcoesPaises;
    let outros = todosPaises.filter(p => !top15.includes(p));
    return ['Todos', ...top15, ...outros];
  })();

  // Função para categorizar patrimônio
  function getWealthBracket(finalWorth) {
    if (finalWorth < 2000) return '<2B';
    if (finalWorth < 5000) return '2B-5B';
    if (finalWorth < 10000) return '5B-10B';
    return '>10B';
  }

  // Atualizar opções de países e setores após carregar dados
  $: if (sankeyData) {
    // Países
    const paisesSet = new Set(sankeyData.map(d => d.pais));
    opcoesPaises = Array.from(paisesSet).sort().slice(0, 15);
    // Setores
    const setoresCount = {};
    sankeyData.forEach(d => {
      setoresCount[d.categoria] = (setoresCount[d.categoria] || 0) + 1;
    });
    opcoesSetores = Object.entries(setoresCount)
      .sort((a, b) => b[1] - a[1])
      .slice(0, 10)
      .map(([setor]) => setor);
  }

  function agruparPaisesPrincipais(data, topN = 15, paisSelecionado = '') {
    // Conta bilionários por país
    const counts = {};
    data.forEach(d => {
      counts[d.pais] = (counts[d.pais] || 0) + 1;
    });
    // Ordena países por quantidade
    let topPaises = Object.entries(counts)
      .sort((a, b) => b[1] - a[1])
      .slice(0, topN)
      .map(([pais]) => pais);
    // Garante que o país selecionado esteja presente
    if (paisSelecionado && !topPaises.includes(paisSelecionado)) {
      topPaises.push(paisSelecionado);
    }
    // Substitui países fora do topN+selecionado por 'Outros'
    return data.map(d => ({
      ...d,
      pais: topPaises.includes(d.pais) ? d.pais : 'Outros'
    }));
  }

  function buildSankeyNodesLinks(data) {
    // Cria os nós únicos para cada etapa
    const nodeLabels = [];
    const nodeIndex = {};
    let idx = 0;
    function addNode(label) {
      if (!(label in nodeIndex)) {
        nodeIndex[label] = idx;
        nodeLabels.push(label);
        idx++;
      }
      return nodeIndex[label];
    }

    // Nova lógica: propaga origem em cada fluxo
    // Mapeia: [origem, país, setor, faixa]
    const pathMap = new Map();
    data.forEach(d => {
      const key = [d.origem, d.pais, d.categoria, d.faixa].join('||');
      pathMap.set(key, (pathMap.get(key) || 0) + 1);
    });

    // Links detalhados com origem
    const linkList = [];
    pathMap.forEach((value, key) => {
      const [origem, pais, categoria, faixa] = key.split('||');
      const origemIdx = addNode(origem);
      const paisIdx = addNode(pais);
      const categoriaIdx = addNode(categoria);
      const faixaIdx = addNode(faixa);
      // origem -> país
      linkList.push({ source: origemIdx, target: paisIdx, value, origem });
      // país -> categoria
      linkList.push({ source: paisIdx, target: categoriaIdx, value, origem });
      // categoria -> faixa
      linkList.push({ source: categoriaIdx, target: faixaIdx, value, origem });
    });

    // Ordenar faixas de patrimônio no final dos nodes e atualizar índices dos links
    const faixaOrder = ['<2B', '2B-5B', '5B-10B', '>10B'];
    const otherLabels = nodeLabels.filter(l => !faixaOrder.includes(l));
    const orderedLabels = [...otherLabels, ...faixaOrder.filter(f => nodeLabels.includes(f))];
    const newIndex = {};
    orderedLabels.forEach((label, i) => { newIndex[label] = i; });
    // Atualizar os links para usar os novos índices
    const updatedLinks = linkList.map(l => ({
      source: newIndex[nodeLabels[l.source]],
      target: newIndex[nodeLabels[l.target]],
      value: l.value,
      origem: l.origem
    }));
    // Gerar array de cores
    const linkColors = updatedLinks.map(l =>
      l.origem === 'Self-Made' ? 'rgba(56, 142, 60, 0.18)' :
      l.origem === 'Herdeiro' ? 'rgba(255, 152, 0, 0.18)' :
      'rgba(160, 160, 160, 0.10)'
    );
    return { nodes: orderedLabels.map(name => ({ name })), links: updatedLinks, linkColors };
  }

  async function loadSankeyData() {
    try {
      isLoading = true;
      loadingError = null;
      sankeyData = [];
      sankeyNodes = [];
      sankeyLinks = [];
      sankeyLinkColors = [];

      const csvPath = `${base}/Billionaires Statistics Dataset.csv`;
      const response = await fetch(csvPath);
      if (!response.ok) throw new Error('Erro ao carregar CSV');
      const rawData = await d3.csv(csvPath);

      // Processar dados para Sankey: selfMade -> categoria -> país -> faixa de patrimônio
      sankeyData = rawData.map(d => ({
        origem: d.selfMade === 'TRUE' ? 'Self-Made' : 'Herdeiro',
        categoria: d.category || d.industries || 'Desconhecido',
        pais: d.country || 'Desconhecido',
        faixa: getWealthBracket(+d.finalWorth || 0)
      }));

      // Gerar nodes e links para Sankey
      const sankey = buildSankeyNodesLinks(sankeyData);
      sankeyNodes = sankey.nodes;
      sankeyLinks = sankey.links;
      sankeyLinkColors = sankey.linkColors;
    } catch (err) {
      loadingError = err.message;
    } finally {
      isLoading = false;
    }
  }

  // Atualizar Sankey sempre que origemFiltro, paisFiltro ou setorFiltro mudar
  $: {
    let filtrado = sankeyData;
    // Filtro de origem
    if (origemFiltro.length > 0 && !origemFiltro.includes('Todos')) {
      filtrado = filtrado.filter(d => origemFiltro.includes(d.origem));
    }
    // Filtro de país
    if (paisFiltro.length === 0 || paisFiltro.includes('Todos')) {
      filtrado = agruparPaisesPrincipais(filtrado, 15);
    } else {
      filtrado = filtrado.filter(d => paisFiltro.includes(d.pais));
    }
    // Filtro de setor
    if (setorFiltro.length > 0 && !setorFiltro.includes('Todos')) {
      filtrado = filtrado.filter(d => setorFiltro.includes(d.categoria));
    }
    const sankey = buildSankeyNodesLinks(filtrado);
    sankeyNodes = sankey.nodes;
    sankeyLinks = sankey.links;
    sankeyLinkColors = sankey.linkColors;
  }

  // Dropdown customizado para filtro de origem
  let origemDropdownAberto = false;
  let origemFiltroTemp = [];
  function toggleOrigemDropdown() {
    origemDropdownAberto = !origemDropdownAberto;
    if (origemDropdownAberto) origemFiltroTemp = [...origemFiltro];
  }
  function fecharOrigemDropdown() { origemDropdownAberto = false; }
  function origemSelecionadaTemp(origem) {
    return origemFiltroTemp.includes(origem);
  }
  function handleOrigemCheckboxTemp(origem) {
    if (origem === 'Todos') {
      if (origemFiltroTemp.includes('Todos')) {
        origemFiltroTemp = [];
      } else {
        origemFiltroTemp = ['Todos'];
      }
    } else {
      if (origemFiltroTemp.includes('Todos')) {
        origemFiltroTemp = [origem];
      } else {
        if (origemFiltroTemp.includes(origem)) {
          origemFiltroTemp = origemFiltroTemp.filter(o => o !== origem);
        } else {
          if (origemFiltroTemp.length < 2) {
            origemFiltroTemp = [...origemFiltroTemp, origem];
          }
        }
      }
    }
  }
  $: if (origemFiltroTemp.length > 2) {
    origemFiltroTemp = origemFiltroTemp.slice(0, 2);
  }
  function aplicarFiltroOrigem() {
    origemFiltro = [...origemFiltroTemp];
    fecharOrigemDropdown();
  }

  // Dropdown customizado para filtro de setor
  let setorDropdownAberto = false;
  let setorFiltroTemp = [];
  function toggleSetorDropdown() {
    setorDropdownAberto = !setorDropdownAberto;
    if (setorDropdownAberto) setorFiltroTemp = [...setorFiltro];
  }
  function fecharSetorDropdown() { setorDropdownAberto = false; }
  function setorSelecionadoTemp(setor) {
    return setorFiltroTemp.includes(setor);
  }
  function handleSetorCheckboxTemp(setor) {
    if (setor === 'Todos') {
      if (setorFiltroTemp.includes('Todos')) {
        setorFiltroTemp = [];
      } else {
        setorFiltroTemp = ['Todos'];
      }
    } else {
      if (setorFiltroTemp.includes('Todos')) {
        setorFiltroTemp = [setor];
      } else {
        if (setorFiltroTemp.includes(setor)) {
          setorFiltroTemp = setorFiltroTemp.filter(s => s !== setor);
        } else {
          if (setorFiltroTemp.length < 10) {
            setorFiltroTemp = [...setorFiltroTemp, setor];
          }
        }
      }
    }
  }
  $: if (setorFiltroTemp.length > 10) {
    setorFiltroTemp = setorFiltroTemp.slice(0, 10);
  }
  function aplicarFiltroSetor() {
    setorFiltro = [...setorFiltroTemp];
    fecharSetorDropdown();
  }

  // Lista de setores para o dropdown: 'Todos', top 10, depois todos os outros em ordem alfabética
  $: listaSetoresDropdown = (() => {
    let todosSetores = Array.from(new Set(sankeyData.map(d => d.categoria))).sort();
    let top10 = opcoesSetores;
    let outros = todosSetores.filter(s => !top10.includes(s));
    return ['Todos', ...top10, ...outros];
  })();

  onMount(() => {
    loadSankeyData();
  });
</script>

<svelte:head>
  <title>Trajetórias da Fortuna | Análise Global de Bilionários</title>
  <meta name="description" content="Visualização de trajetórias de bilionários self-made e herdeiros por setor, país e patrimônio." />
</svelte:head>

<div class="page-container">
  <nav class="main-nav">
    <h1 class="site-title">Análise Global de Bilionários</h1>
    <div class="nav-links">
      <a href="{base}/" class="nav-link">Visão Geral</a>
      <a href="{base}/gender-gap" class="nav-link">Gap de Gênero</a>
      <a href="{base}/map" class="nav-link">Mapa de Migração</a>
      <a href="{base}/network" class="nav-link">Rede de Indústrias</a>
      <a href="{base}/ascensao" class="nav-link active">Trajetórias da Fortuna</a>
    </div>
  </nav>

  <section class="hero" style="background: linear-gradient(135deg, #e3f0ff 0%, #d0e7ff 100%); border-radius: 0 0 24px 24px; margin-bottom: 30px; box-shadow: 0 4px 12px rgba(0,0,0,0.05); padding: 50px 20px; text-align: center;">
    <div class="hero-content" style="max-width: 800px; margin: 0 auto;">
      <h1 style="font-size: 2.8rem; font-weight: 800; margin: 0 0 16px; color: #2d3748; letter-spacing: -0.5px; line-height: 1.2; background: linear-gradient(90deg, #2b6cb0 0%, #4299e1 100%); -webkit-background-clip: text; -webkit-text-fill-color: transparent; background-clip: text;">Trajetórias da Fortuna</h1>
      <p style="font-size: 1.2rem; color: #4a5568; margin: 0 auto; max-width: 700px;">Descubra os caminhos percorridos por bilionários ao redor do mundo: de sua origem da fortuna, passando por países, setores de atuação e faixas de patrimônio. Visualize como diferentes fatores se conectam na formação das grandes fortunas globais.</p>
    </div>
  </section>

  <section class="content-section" style="display: flex; flex-direction: column; gap: 40px; background: #fff; border-radius: 16px; box-shadow: 0 4px 20px rgba(0,0,0,0.08); padding: 40px; border: 1px solid rgba(0,0,0,0.1); margin-bottom: 40px;">
    <div class="section-intro" style="max-width: 800px; margin: 0 auto; text-align: center;">
      <div class="section-icon" style="font-size: 1.8rem; background: #ebf8ff; color: #2b6cb0; width: 50px; height: 50px; display: flex; align-items: center; justify-content: center; border-radius: 12px; margin: 0 auto 16px;">🔀</div>
      <h2 style="font-size: 1.8em; color: #2d3748; margin: 0 0 12px; font-weight: 700; letter-spacing: -0.5px;">Como Bilionários Constroem Suas Fortunas</h2>
      <p style="font-size: 1.05em; margin-bottom: 16px; color: #4a5568; line-height: 1.7;">O diagrama Sankey abaixo mostra os fluxos de bilionários entre origem da fortuna, país de residência, setor de atuação e faixa de patrimônio líquido. Cada fluxo representa o número de pessoas que seguem aquele caminho, revelando padrões de mobilidade social, concentração setorial e distribuição geográfica da riqueza extrema.</p>
    </div>

    <!-- Sankey e filtros permanecem aqui -->
    <div class="visualization-card" style="background: #f8fafc; border-radius: 16px; box-shadow: 0 4px 15px rgba(0,0,0,0.06); border: 1px solid #e3e8ee; padding: 32px 0 32px 0; margin: 0 auto 32px auto; width: 100%;">
      <div style="display: flex; justify-content: center; align-items: flex-end; margin: 24px 32px 8px 32px; gap: 32px; flex-wrap: wrap;">
        <!-- Filtro Origem (dropdown customizado) -->
        <div style="flex: 1 1 180px; max-width: 260px; min-width: 140px; text-align: left; position: relative;">
          <label for="origemFiltro" style="font-weight: 500; color: #1976d2;">Origem da Fortuna:</label><br>
          <div tabindex="0" style="width: 100%; min-width: 100px; padding: 4px 8px; border-radius: 6px; border: 1px solid #bcd; background: #fff; cursor: pointer;" on:click={toggleOrigemDropdown}>
            {#if origemFiltro.length === 0}
              <span style="color: #888;">Selecione...</span>
            {:else if origemFiltro.includes('Todos')}
              <span>Todos</span>
            {:else}
              <span>{origemFiltro.join(', ')}</span>
            {/if}
          </div>
          {#if origemDropdownAberto}
            <div on:click={fecharOrigemDropdown} style="position: fixed; left: 0; top: 0; width: 100vw; height: 100vh; z-index: 9;"></div>
            <div style="position: absolute; left: 0; top: 48px; width: 100%; background: #fff; border: 1px solid #bcd; border-radius: 6px; z-index: 10; box-shadow: 0 2px 8px #0001;">
              <div style="max-height: 120px; overflow-y: auto;">
                {#each ['Todos', ...opcoesOrigem] as origem}
                  <div style="padding: 4px 8px; display: flex; align-items: center; gap: 8px; cursor: pointer; {origemFiltroTemp.includes('Todos') && origem !== 'Todos' ? 'color: #bbb;' : ''}" on:click|stopPropagation={() => handleOrigemCheckboxTemp(origem)}>
                    <input type="checkbox" checked={origemSelecionadaTemp(origem)} disabled={origemFiltroTemp.includes('Todos') && origem !== 'Todos'} />
                    <span>{origem}</span>
                  </div>
                {/each}
              </div>
              <div style="padding: 8px 0 4px 0; text-align: right; border-top: 1px solid #e3eaf6; background: #fff; position: sticky; bottom: 0; z-index: 2;">
                <button on:click|stopPropagation={aplicarFiltroOrigem} style="background: none; border: none; cursor: pointer; color: #1976d2; font-size: 1.2em; padding: 4px 12px; display: flex; align-items: center; gap: 4px;">
                  <svg xmlns='http://www.w3.org/2000/svg' width='20' height='20' fill='none' viewBox='0 0 24 24' stroke='currentColor'><path stroke-linecap='round' stroke-linejoin='round' stroke-width='2' d='M21 21l-4.35-4.35m0 0A7.5 7.5 0 104.5 4.5a7.5 7.5 0 0012.15 12.15z'/></svg>
                  <span style="font-size: 1em;">Aplicar</span>
                </button>
              </div>
            </div>
          {/if}
        </div>
        <!-- Filtro País (dropdown customizado) -->
        <div style="flex: 1 1 220px; max-width: 320px; min-width: 140px; text-align: center; position: relative;">
          <label for="paisFiltro" style="font-weight: 500; color: #1976d2;">País:</label><br>
          <div tabindex="0" style="width: 100%; min-width: 120px; padding: 4px 8px; border-radius: 6px; border: 1px solid #bcd; background: #fff; cursor: pointer;" on:click={togglePaisDropdown}>
            {#if paisFiltro.length === 0}
              <span style="color: #888;">Selecione...</span>
            {:else if paisFiltro.includes('Todos')}
              <span>Todos</span>
            {:else}
              <span>{paisFiltro.join(', ')}</span>
            {/if}
          </div>
          {#if paisDropdownAberto}
            <div on:click={fecharPaisDropdown} style="position: fixed; left: 0; top: 0; width: 100vw; height: 100vh; z-index: 9;"></div>
            <div style="position: absolute; left: 0; top: 48px; width: 100%; background: #fff; border: 1px solid #bcd; border-radius: 6px; z-index: 10; box-shadow: 0 2px 8px #0001;">
              <div style="max-height: 220px; overflow-y: auto;">
                {#each listaPaisesDropdown as pais}
                  <div style="padding: 4px 8px; display: flex; align-items: center; gap: 8px; cursor: pointer; {paisFiltroTemp.includes('Todos') && pais !== 'Todos' ? 'color: #bbb;' : ''}" on:click|stopPropagation={() => handlePaisCheckboxTemp(pais)}>
                    <input type="checkbox" checked={paisSelecionadoTemp(pais)} disabled={paisFiltroTemp.includes('Todos') && pais !== 'Todos'} />
                    <span>{pais}</span>
                  </div>
                {/each}
              </div>
              <div style="padding: 8px 0 4px 0; text-align: right; border-top: 1px solid #e3eaf6; background: #fff; position: sticky; bottom: 0; z-index: 2;">
                <button on:click|stopPropagation={aplicarFiltroPais} style="background: none; border: none; cursor: pointer; color: #1976d2; font-size: 1.2em; padding: 4px 12px; display: flex; align-items: center; gap: 4px;">
                  <svg xmlns='http://www.w3.org/2000/svg' width='20' height='20' fill='none' viewBox='0 0 24 24' stroke='currentColor'><path stroke-linecap='round' stroke-linejoin='round' stroke-width='2' d='M21 21l-4.35-4.35m0 0A7.5 7.5 0 104.5 4.5a7.5 7.5 0 0012.15 12.15z'/></svg>
                  <span style="font-size: 1em;">Aplicar</span>
                </button>
              </div>
            </div>
          {/if}
        </div>
        <!-- Filtro Setor (dropdown customizado) -->
        <div style="flex: 1 1 220px; max-width: 320px; min-width: 140px; text-align: center; position: relative;">
          <label for="setorFiltro" style="font-weight: 500; color: #1976d2;">Setor:</label><br>
          <div tabindex="0" style="width: 100%; min-width: 120px; padding: 4px 8px; border-radius: 6px; border: 1px solid #bcd; background: #fff; cursor: pointer;" on:click={toggleSetorDropdown}>
            {#if setorFiltro.length === 0}
              <span style="color: #888;">Selecione...</span>
            {:else if setorFiltro.includes('Todos')}
              <span>Todos</span>
            {:else}
              <span>{setorFiltro.join(', ')}</span>
            {/if}
          </div>
          {#if setorDropdownAberto}
            <div on:click={fecharSetorDropdown} style="position: fixed; left: 0; top: 0; width: 100vw; height: 100vh; z-index: 9;"></div>
            <div style="position: absolute; left: 0; top: 48px; width: 100%; background: #fff; border: 1px solid #bcd; border-radius: 6px; z-index: 10; box-shadow: 0 2px 8px #0001;">
              <div style="max-height: 220px; overflow-y: auto;">
                {#each listaSetoresDropdown as setor}
                  <div style="padding: 4px 8px; display: flex; align-items: center; gap: 8px; cursor: pointer; {setorFiltroTemp.includes('Todos') && setor !== 'Todos' ? 'color: #bbb;' : ''}" on:click|stopPropagation={() => handleSetorCheckboxTemp(setor)}>
                    <input type="checkbox" checked={setorSelecionadoTemp(setor)} disabled={setorFiltroTemp.includes('Todos') && setor !== 'Todos'} />
                    <span>{setor}</span>
                  </div>
                {/each}
              </div>
              <div style="padding: 8px 0 4px 0; text-align: right; border-top: 1px solid #e3eaf6; background: #fff; position: sticky; bottom: 0; z-index: 2;">
                <button on:click|stopPropagation={aplicarFiltroSetor} style="background: none; border: none; cursor: pointer; color: #1976d2; font-size: 1.2em; padding: 4px 12px; display: flex; align-items: center; gap: 4px;">
                  <svg xmlns='http://www.w3.org/2000/svg' width='20' height='20' fill='none' viewBox='0 0 24 24' stroke='currentColor'><path stroke-linecap='round' stroke-linejoin='round' stroke-width='2' d='M21 21l-4.35-4.35m0 0A7.5 7.5 0 104.5 4.5a7.5 7.5 0 0012.15 12.15z'/></svg>
                  <span style="font-size: 1em;">Aplicar</span>
                </button>
              </div>
            </div>
          {/if}
        </div>
      </div>

      {#if isLoading || loadingError || sankeyNodes.length === 0 || sankeyLinks.length === 0}
        <div class="sankey-placeholder" style="height: 400px; display: flex; align-items: center; justify-content: center; background: #f5f7fa; border-radius: 12px; margin: 32px 32px 0 32px;">
          <span style="color: #bbb; font-size: 1.2em;">[Aqui aparecerá o Diagrama de Sankey]</span>
        </div>
      {/if}
      {#if !isLoading && !loadingError && sankeyNodes.length > 0 && sankeyLinks.length > 0}
        <div style="margin: 32px 32px 0 32px; position: relative; padding-bottom: 48px;">
          <SankeyDiagram nodes={sankeyNodes} links={sankeyLinks} linkColors={sankeyLinkColors} />
          <div class="sankey-column-labels" style="position: absolute; left: 0; width: 100%; bottom: -32px; pointer-events: none; z-index: 2; display: flex; justify-content: space-between;">
            <span style="flex: 1; text-align: left; color: #1976d2; font-weight: 600;">Origem</span>
            <span style="flex: 1; text-align: center; color: #1976d2; font-weight: 600;">País</span>
            <span style="flex: 1; text-align: center; color: #1976d2; font-weight: 600;">Setor</span>
            <span style="flex: 1; text-align: right; color: #1976d2; font-weight: 600;">Faixa de Patrimônio</span>
          </div>
        </div>
      {/if}
    </div>

    <!-- Como interpretar -->
    <div class="info-grid" style="display: grid; grid-template-columns: 1fr 1fr; gap: 24px; margin-top: 32px;">
      <div class="info-card" style="background: #f8fafc; border-radius: 12px; padding: 24px; box-shadow: 0 2px 12px rgba(44,62,80,0.06); border: 1px solid #e3e8ee;">
        <h3 style="font-size: 1.4em; margin: 0 0 16px 0; color: #333; font-weight: 600;">Como Interpretar a Visualização</h3>
        <ul style="padding-left: 20px; margin-bottom: 16px; color: #4a5568;">
          <li><b>Coluna 1:</b> Origem da fortuna (Self-Made ou Herdeiro)</li>
          <li><b>Coluna 2:</b> País de residência</li>
          <li><b>Coluna 3:</b> Setor/Categoria principal de atuação</li>
          <li><b>Coluna 4:</b> Faixa de patrimônio estimado</li>
          <li><b>Fluxos:</b> A largura de cada fluxo representa o número de bilionários que seguem aquele caminho.</li>
        </ul>
        <p style="font-size: 0.98em; color: #4a5568;">Use os filtros acima para explorar diferentes origens, países e setores. Passe o mouse sobre os fluxos para ver detalhes.</p>
      </div>
      <div class="info-card" style="background: #f8fafc; border-radius: 12px; padding: 24px; box-shadow: 0 2px 12px rgba(44,62,80,0.06); border: 1px solid #e3e8ee;">
        <h3 style="font-size: 1.4em; margin: 0 0 16px 0; color: #333; font-weight: 600;">Principais Insights</h3>
        <ul style="padding-left: 20px; margin-bottom: 16px; color: #4a5568;">
          <li>Setores como tecnologia, finanças e manufatura concentram a maior parte das fortunas self-made.</li>
          <li>Em alguns países, a herança ainda predomina entre os mais ricos, enquanto em outros a mobilidade self-made é mais comum.</li>
          <li>Os fluxos revelam como fatores geográficos e setoriais influenciam as trajetórias de enriquecimento.</li>
        </ul>
        <div class="insight-box" style="margin-top: 10px; background: #e3f0ff; border-radius: 8px; padding: 12px;">
          <b>Exemplo:</b> É possível observar em quais setores a ascensão self-made é mais comum, ou em quais países a herança ainda predomina entre os mais ricos.
        </div>
      </div>
    </div>

    <div class="info-card" style="background: #f8fafc; border-radius: 12px; padding: 24px; box-shadow: 0 2px 12px rgba(44,62,80,0.06); border: 1px solid #e3e8ee; margin-top: 32px;">
      <h3 style="font-size: 1.4em; margin: 0 0 16px 0; color: #333; font-weight: 600;">Metodologia</h3>
      <p style="font-size: 1.05em; color: #4a5568;">As faixas de patrimônio foram ajustadas para melhor equilíbrio visual. Os dados refletem a distribuição real dos bilionários no dataset analisado. O diagrama Sankey utiliza as informações de origem da fortuna, país, setor e patrimônio para construir os fluxos. Países e setores menos representativos são agrupados para facilitar a visualização.</p>
    </div>
  </section>
</div>

<style>
  .nav-link.active {
    background: #e3f0ff;
    border-radius: 6px;
    color: #1976d2;
    font-weight: bold;
  }
</style> 