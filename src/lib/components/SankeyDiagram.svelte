<script>
  import { onMount, onDestroy } from 'svelte';
  let Plotly;
  export let nodes = [];
  export let links = [];
  export let linkColors = [];
  let chartDiv;

  // Todos os nós são cinza claro
  function getNodeColors() {
    return nodes.map(() => '#e3eaf6');
  }

  // Encontrar a origem raiz (Self-Made ou Herdeiro) para cada nó, usando cache para performance
  let rootOriginCache = {};
  function findRootOriginNode(nodeIdx) {
    if (rootOriginCache[nodeIdx] !== undefined) return rootOriginCache[nodeIdx];
    const name = nodes[nodeIdx]?.name;
    if (name === 'Self-Made' || name === 'Herdeiro') {
      rootOriginCache[nodeIdx] = name;
      return name;
    }
    // Procura todos os links que chegam nesse nó
    const parentLinks = links.filter(l => l.target === nodeIdx);
    for (const parentLink of parentLinks) {
      const parentOrigin = findRootOriginNode(parentLink.source);
      if (parentOrigin) {
        rootOriginCache[nodeIdx] = parentOrigin;
        return parentOrigin;
      }
    }
    rootOriginCache[nodeIdx] = null;
    return null;
  }

  // Tooltips customizados para links
  function getLinkLabels() {
    return links.map(l => {
      const source = nodes[l.source]?.name || '';
      const target = nodes[l.target]?.name || '';
      return `${source} → ${target}<br>Quantidade: ${l.value}`;
    });
  }

  function getSankeyData() {
    return {
      type: 'sankey',
      orientation: 'h',
      node: {
        pad: 30,
        thickness: 20,
        line: { color: 'black', width: 0.5 },
        label: nodes.map(n => n.name),
        color: getNodeColors(),
      },
      link: {
        source: links.map(l => l.source),
        target: links.map(l => l.target),
        value: links.map(l => l.value),
        color: linkColors,
        label: getLinkLabels(),
        customdata: getLinkLabels(),
        hovertemplate: '%{customdata}<extra></extra>',
      }
    };
  }

  function getLayout() {
    return {
      margin: { t: 30, l: 0, r: 0, b: 80 },
      font: { size: 11 },
      height: 600,
      width: undefined,
      autosize: true,
    };
  }

  onMount(async () => {
    Plotly = (await import('plotly.js-dist-min')).default;
    if (chartDiv && nodes.length && links.length) {
      Plotly.newPlot(chartDiv, [{ ...getSankeyData() }], getLayout(), { responsive: true });
    }
  });

  // Atualiza o gráfico se os dados mudarem
  $: if (chartDiv && nodes.length && links.length) {
    Plotly.react(chartDiv, [{ ...getSankeyData() }], getLayout(), { responsive: true });
  }

  onDestroy(() => {
    if (chartDiv) Plotly.purge(chartDiv);
  });

  // Expor labels para uso externo
  export let labels = nodes.map(n => n.name);
</script>

<div bind:this={chartDiv} style="width: 100%; min-width: 320px; min-height: 500px; position: relative;"></div> 