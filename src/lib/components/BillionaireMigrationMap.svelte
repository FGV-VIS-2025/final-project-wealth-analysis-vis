<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';

  export let data = [];
  
  let svgElement;
  let tooltip;
  let selectedCountry = null;
  let selectedIndustry = 'Todos';
  let selectedGender = 'Todos';
  let showStatisticsPanel = false;
  
  const width = 1000;
  const height = 500;
  
  const worldCountries = [
    {name: "United States", center: [-95.7129, 37.0902]},
    {name: "China", center: [104.1954, 35.8617]},
    {name: "United Kingdom", center: [-3.4360, 55.3781]},
    {name: "Germany", center: [10.4515, 51.1657]},
    {name: "France", center: [2.2137, 46.2276]},
    {name: "India", center: [78.9629, 20.5937]},
    {name: "Japan", center: [138.2529, 36.2048]},
    {name: "Russia", center: [105.3188, 61.5240]},
    {name: "Brazil", center: [-51.9253, -14.2350]},
    {name: "Canada", center: [-106.3468, 56.1304]},
    {name: "Australia", center: [133.7751, -25.2744]},
    {name: "Mexico", center: [-102.5528, 23.6345]},
    {name: "Spain", center: [-3.7492, 40.4637]},
    {name: "Italy", center: [12.5674, 41.8719]},
    {name: "Switzerland", center: [8.2275, 46.8182]},
    {name: "South Korea", center: [127.7669, 35.9078]},
    {name: "Netherlands", center: [5.2913, 52.1326]},
    {name: "Sweden", center: [18.6435, 60.1282]},
    {name: "Hong Kong", center: [114.1694, 22.3193]},
    {name: "Singapore", center: [103.8198, 1.3521]},
    {name: "Belgium", center: [4.4699, 50.5039]},
    {name: "Israel", center: [34.8516, 31.0461]},
    {name: "Norway", center: [8.4689, 60.4720]},
    {name: "Thailand", center: [100.9925, 15.8700]},
    {name: "Indonesia", center: [113.9213, -0.7893]},
    {name: "South Africa", center: [22.9375, -30.5595]},
    {name: "Chile", center: [-71.5430, -35.6751]},
    {name: "Czech Republic", center: [15.4730, 49.8175]}
  ];

  let filteredData = [];
  let migrationData = [];
  let countryStats = {};
  let industries = [];
  let genders = [];
  let simulation;

  // Adicione o mapeamento de país para continente (copiado do +page.svelte)
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
    'Norway': 'Europa',
    'Belgium': 'Europa',
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

  // Defina as cores para cada continente
  const continentColors = {
    'América do Norte': '#1f77b4', // azul
    'América do Sul': '#ff7f0e',   // laranja
    'Europa': '#2ca02c',           // verde
    'Ásia': '#d62728',             // vermelho
    'Oceania': '#9467bd',          // roxo
    'África': '#8c564b',           // marrom
    'Desconhecido': '#cccccc'      // cinza
  };

  onMount(() => {
    if (data.length > 0) {
      extractFilters();
      updateFilteredData();
    }
  });

  function extractFilters() {
    const industriesSet = new Set();
    const gendersSet = new Set();
    
    data.forEach(person => {
      if (person.industries) industriesSet.add(person.industries);
      if (person.gender) gendersSet.add(person.gender);
    });
    
    industries = ['Todos', ...Array.from(industriesSet).sort()];
    genders = ['Todos', ...Array.from(gendersSet).sort()];
  }

  function updateFilteredData() {
    if (!data || data.length === 0) return;
    
    filteredData = data.filter(person => {
      let industryMatch = false;
      if (selectedIndustry === 'Todos') {
        industryMatch = true;
      } else if (person.industries) {
        industryMatch = person.industries.toString().trim() === selectedIndustry.toString().trim();
      }
      
      let genderMatch = false;
      if (selectedGender === 'Todos') {
        genderMatch = true;
      } else if (person.gender) {
        genderMatch = person.gender.toString().trim() === selectedGender.toString().trim();
      }
      
      return industryMatch && genderMatch;
    });
    
    processData();
    createVisualization();
  }

  function processData() {
    countryStats = {};
    
    filteredData.forEach(person => {
      const birthCountry = person.countryOfCitizenship || 'Unknown';
      const residenceCountry = person.country || 'Unknown';
      
      if (residenceCountry !== 'Unknown') {
        if (!countryStats[residenceCountry]) {
          countryStats[residenceCountry] = { 
            residents: 0, natives: 0, immigrants: 0, totalWealth: 0, avgAge: 0
          };
        }
        
        countryStats[residenceCountry].residents++;
        countryStats[residenceCountry].totalWealth += parseFloat(person.finalWorth || 0);
        
        if (birthCountry !== residenceCountry && birthCountry !== 'Unknown') {
          countryStats[residenceCountry].immigrants++;
        } else if (birthCountry === residenceCountry) {
          countryStats[residenceCountry].natives++;
        }
      }
    });

    Object.keys(countryStats).forEach(country => {
      const ages = filteredData
        .filter(p => p.country === country && p.age)
        .map(p => parseFloat(p.age));
      
      countryStats[country].avgAge = ages.length > 0 ? 
        ages.reduce((sum, age) => sum + age, 0) / ages.length : 0;
    });

    const flows = {};
    filteredData.forEach(person => {
      const birthCountry = person.countryOfCitizenship || 'Unknown';
      const residenceCountry = person.country || 'Unknown';
      
      if (birthCountry !== residenceCountry && birthCountry !== 'Unknown' && residenceCountry !== 'Unknown') {
        const flowKey = `${birthCountry}->${residenceCountry}`;
        flows[flowKey] = (flows[flowKey] || 0) + 1;
      }
    });

    migrationData = Object.entries(flows)
      .map(([flow, count]) => {
        const [source, target] = flow.split('->');
        return { source, target, count };
      })
      .filter(d => d.count >= 1)
      .sort((a, b) => b.count - a.count);
  }

  function createVisualization() {
    const svg = d3.select(svgElement);
    svg.selectAll("*").remove();

    if (Object.keys(countryStats).length === 0) {
      svg.append("text")
        .attr("x", width / 2)
        .attr("y", height / 2)
        .attr("text-anchor", "middle")
        .attr("fill", "#e0e0e0")
        .attr("font-size", "18px")
        .text("Nenhum dado encontrado para os filtros selecionados");
      return;
    }

    const projection = d3.geoNaturalEarth1()
      .scale(170)
      .translate([width / 2, height / 2]);

    const countryData = worldCountries
      .filter(country => countryStats[country.name])
      .map(country => ({
        ...country,
        stats: countryStats[country.name],
        projected: projection(country.center)
      }))
      .filter(d => d.projected);

    const maxResidents = Math.max(...Object.values(countryStats).map(d => d.residents));
    const circleScale = d3.scaleSqrt().domain([0, maxResidents]).range([8, 45]);

    const nodes = countryData.map(d => ({
      ...d,
      x: d.projected[0],
      y: d.projected[1],
      r: circleScale(d.stats.residents),
      continent: countryToContinent[d.name] || 'Desconhecido'
    }));

    svg.append("rect")
      .attr("width", width)
      .attr("height", height)
      .attr("fill", "#1a1a1a")
      .attr("stroke", "#333");

    const defs = svg.append("defs");
    
    defs.append("marker")
      .attr("id", "arrowhead")
      .attr("viewBox", "0 -5 10 10")
      .attr("refX", 8)
      .attr("refY", 0)
      .attr("markerWidth", 4)
      .attr("markerHeight", 4)
      .attr("orient", "auto")
      .append("path")
      .attr("d", "M0,-5L10,0L0,5")
      .attr("fill", "#ffd700");

    defs.append("marker")
      .attr("id", "arrowhead-highlighted")
      .attr("viewBox", "0 -5 10 10")
      .attr("refX", 8)
      .attr("refY", 0)
      .attr("markerWidth", 5)
      .attr("markerHeight", 5)
      .attr("orient", "auto")
      .append("path")
      .attr("d", "M0,-5L10,0L0,5")
      .attr("fill", "#ff6b6b");

    simulation = d3.forceSimulation(nodes)
      .force("collision", d3.forceCollide().radius(d => d.r + 15).strength(0.9))
      .force("center", d3.forceCenter(width / 2, height / 2).strength(0.05))
      .force("position", d3.forceRadial(0, width / 2, height / 2).strength(0.05))
      .stop();

    for (let i = 0; i < 200; i++) simulation.tick();

    const flows = svg.append("g").selectAll("path")
      .data(migrationData)
      .enter()
      .append("path")
      .attr("fill", "none")
      .attr("stroke", "#ffd700")
      .attr("stroke-width", d => Math.max(1, Math.sqrt(d.count) * 1.2))
      .attr("stroke-opacity", 0.6)
      .attr("marker-end", "url(#arrowhead)")
      .style("cursor", "pointer")
      .attr("d", d => {
        const sourceNode = nodes.find(n => n.name === d.source);
        const targetNode = nodes.find(n => n.name === d.target);
        
        if (sourceNode && targetNode) {
          const dx = targetNode.x - sourceNode.x;
          const dy = targetNode.y - sourceNode.y;
          const dr = Math.sqrt(dx * dx + dy * dy) * 0.5;
          return `M${sourceNode.x},${sourceNode.y}A${dr},${dr} 0 0,1 ${targetNode.x},${targetNode.y}`;
        }
        return "";
      })
      .on("mouseover", function(event, d) {
        d3.select(this)
          .attr("stroke-opacity", 0.9)
          .attr("stroke-width", Math.max(2, Math.sqrt(d.count) * 1.8));
        showTooltip(event, `${d.source} → ${d.target}\n${d.count} bilionário${d.count > 1 ? 's' : ''} migraram`);
      })
      .on("mouseout", function(event, d) {
        if (!selectedCountry || (d.source !== selectedCountry && d.target !== selectedCountry)) {
          d3.select(this)
            .attr("stroke-opacity", 0.6)
            .attr("stroke-width", Math.max(1, Math.sqrt(d.count) * 1.2));
        }
        hideTooltip();
      });

    const countryCircles = svg.append("g").selectAll("circle")
      .data(nodes)
      .enter()
      .append("circle")
      .attr("cx", d => d.x)
      .attr("cy", d => d.y)
      .attr("r", d => d.r)
      .attr("fill", d => continentColors[d.continent] || continentColors['Desconhecido'])
      .attr("fill-opacity", 0.8)
      .attr("stroke", "#fff")
      .attr("stroke-width", 2)
      .style("cursor", "pointer")
      .on("mouseover", function(event, d) {
        if (!selectedCountry) {
          d3.select(this).transition().duration(200).attr("fill-opacity", 1).attr("stroke-width", 3);
        }
        const tooltipText = `${d.name}\nTotal: ${d.stats.residents} bilionários\nNativos: ${d.stats.natives}\nImigrantes: ${d.stats.immigrants}\nRiqueza total: $${(d.stats.totalWealth).toFixed(1)}B\nIdade média: ${d.stats.avgAge.toFixed(1)} anos`;
        showTooltip(event, tooltipText);
      })
      .on("mouseout", function(event, d) {
        if (!selectedCountry || selectedCountry !== d.name) {
          d3.select(this).transition().duration(200).attr("fill-opacity", 0.8).attr("stroke-width", 2);
        }
        hideTooltip();
      })
      .on("click", function(event, d) {
        selectCountry(d.name);
      });

    svg.append("g").selectAll("text")
      .data(nodes.filter(d => d.stats.residents >= 3))
      .enter()
      .append("text")
      .attr("x", d => d.x)
      .attr("y", d => d.y + d.r + 18)
      .attr("text-anchor", "middle")
      .attr("font-size", "12px")
      .attr("font-weight", "600")
      .attr("fill", "#e0e0e0")
      .style("pointer-events", "none")
      .text(d => d.name);

    function selectCountry(countryName) {
      if (selectedCountry === countryName) {
        selectedCountry = null;
        resetVisualization();
      } else {
        selectedCountry = countryName;
        highlightCountryFlows(countryName);
      }
    }

    function resetVisualization() {
      countryCircles.attr("fill", d => continentColors[d.continent] || continentColors['Desconhecido'])
                    .attr("fill-opacity", 0.8)
                    .attr("stroke-width", 2);
      flows.attr("stroke", "#ffd700")
           .attr("stroke-opacity", 0.6)
           .attr("stroke-width", d => Math.max(1, Math.sqrt(d.count) * 1.2))
           .attr("marker-end", "url(#arrowhead)");
    }

    function highlightCountryFlows(countryName) {
      countryCircles
        .attr("fill", d => d.name === countryName ? "#ff6b6b" : continentColors[d.continent] || continentColors['Desconhecido'])
        .attr("fill-opacity", d => d.name === countryName ? 1 : 0.3)
        .attr("stroke-width", d => d.name === countryName ? 4 : 2);

      flows
        .attr("stroke", d => (d.source === countryName || d.target === countryName) ? "#ff6b6b" : continentColors[d.continent] || continentColors['Desconhecido'])
        .attr("stroke-opacity", d => (d.source === countryName || d.target === countryName) ? 0.9 : 0.1)
        .attr("stroke-width", d => (d.source === countryName || d.target === countryName) ? Math.max(2, Math.sqrt(d.count) * 2) : Math.max(1, Math.sqrt(d.count) * 1.2))
        .attr("marker-end", d => (d.source === countryName || d.target === countryName) ? "url(#arrowhead-highlighted)" : "url(#arrowhead)");
    }
  }

  function showTooltip(event, text) {
    if (!tooltip) {
      tooltip = d3.select("body").append("div")
        .attr("class", "map-tooltip")
        .style("position", "absolute")
        .style("background", "rgba(0, 0, 0, 0.95)")
        .style("color", "#ffd700")
        .style("padding", "12px")
        .style("border-radius", "6px")
        .style("font-size", "12px")
        .style("font-weight", "500")
        .style("pointer-events", "none")
        .style("white-space", "pre-line")
        .style("z-index", "1000")
        .style("border", "1px solid #ffd700")
        .style("box-shadow", "0 4px 8px rgba(0,0,0,0.3)");
    }
    
    tooltip.html(text)
      .style("opacity", 1)
      .style("left", (event.pageX + 10) + "px")
      .style("top", (event.pageY - 10) + "px");
  }

  function hideTooltip() {
    if (tooltip) tooltip.style("opacity", 0);
  }

  $: if (data.length > 0 && svgElement) {
    updateFilteredData();
  }

  $: if (selectedIndustry || selectedGender) {
    if (data.length > 0 && svgElement) {
      updateFilteredData();
    }
  }
</script>

<div class="map-container">
  <div class="controls-panel controls-center">
    <div class="control-group">
      <label for="industry-select">Filtro por Indústria:</label>
      <select id="industry-select" bind:value={selectedIndustry}>
        {#each industries as industry}
          <option value={industry}>{industry}</option>
        {/each}
      </select>
    </div>
    <div class="control-group">
      <label for="gender-select">Filtro por Gênero:</label>
      <select id="gender-select" bind:value={selectedGender}>
        {#each genders as gender}
          <option value={gender}>{gender}</option>
        {/each}
      </select>
    </div>
    <div class="control-group">
      <button 
        class="stats-toggle"
        on:click={() => showStatisticsPanel = !showStatisticsPanel}
      >
        {showStatisticsPanel ? 'Ocultar' : 'Mostrar'} Estatísticas
      </button>
    </div>
  </div>
  {#if showStatisticsPanel}
    <div class="statistics-panel">
      <h3>Estatísticas Globais</h3>
      <div class="stats-grid">
        <div class="stat-item">
          <span class="stat-label">Total de Bilionários:</span>
          <span class="stat-value">{filteredData.length}</span>
        </div>
        <div class="stat-item">
          <span class="stat-label">Países com Bilionários:</span>
          <span class="stat-value">{Object.keys(countryStats).length}</span>
        </div>
        <div class="stat-item">
          <span class="stat-label">Fluxos Migratórios:</span>
          <span class="stat-value">{migrationData.length}</span>
        </div>
        <div class="stat-item">
          <span class="stat-label">Riqueza Total:</span>
          <span class="stat-value">
            ${Object.values(countryStats).reduce((sum, stats) => sum + stats.totalWealth, 0).toFixed(1)}B
          </span>
        </div>
        <div class="stat-item">
          <span class="stat-label">Filtros Ativos:</span>
          <span class="stat-value">{selectedIndustry} | {selectedGender}</span>
        </div>
        {#if selectedCountry && countryStats[selectedCountry]}
          <div class="stat-item selected-country">
            <span class="stat-label">{selectedCountry}:</span>
            <span class="stat-value">
              {countryStats[selectedCountry].residents} bilionários
              ({countryStats[selectedCountry].natives} nativos, {countryStats[selectedCountry].immigrants} imigrantes)
            </span>
          </div>
        {/if}
      </div>
    </div>
  {/if}
  <div class="map-visualization">
    <svg bind:this={svgElement} {width} {height}></svg>
    <div class="continent-legend legend-overlay">
      {#each Object.entries(continentColors) as [continent, color]}
        {#if continent !== 'Desconhecido'}
          <div class="legend-item">
            <span class="legend-color" style="background: {color}"></span>
            <span class="legend-label">{continent}</span>
          </div>
        {/if}
      {/each}
    </div>
  </div>
</div>

<style>
  .map-container {
    width: 100%;
    max-width: 1050px;
    margin: 0 auto;
    font-family: 'Arial', sans-serif;
    display: flex;
    flex-direction: column;
  }
  .controls-panel {
    display: flex;
    flex-wrap: wrap;
    gap: 20px;
    padding: 20px;
    background: #2a2a2a;
    border-radius: 8px;
    margin-bottom: 20px;
    border: 1px solid #444;
  }
  .controls-center {
    justify-content: center;
  }
  .control-group {
    display: flex;
    align-items: center;
    gap: 10px;
  }
  .control-group label {
    font-size: 14px;
    color: #e0e0e0;
    font-weight: 500;
  }
  .stats-toggle {
    padding: 8px 16px;
    background: #ffd700;
    color: #000;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 14px;
    font-weight: 500;
    transition: background 0.2s;
  }
  .stats-toggle:hover {
    background: #ffed4e;
  }
  select {
    padding: 8px 12px;
    border: 1px solid #555;
    border-radius: 4px;
    background: #1a1a1a;
    color: #e0e0e0;
    font-size: 14px;
    min-width: 150px;
  }
  select:focus {
    outline: none;
    border-color: #ffd700;
  }
  .statistics-panel {
    background: #2a2a2a;
    border: 1px solid #444;
    border-radius: 8px;
    padding: 20px;
    margin-bottom: 20px;
  }
  .statistics-panel h3 {
    color: #ffd700;
    margin: 0 0 15px 0;
    font-size: 18px;
  }
  .stats-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
    gap: 15px;
  }
  .stat-item {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 10px;
    background: #1a1a1a;
    border-radius: 6px;
    border: 1px solid #444;
  }
  .stat-item.selected-country {
    border-color: #ff6b6b;
    background: rgba(255, 107, 107, 0.1);
  }
  .stat-label {
    color: #b0b0b0;
    font-size: 14px;
  }
  .stat-value {
    color: #ffd700;
    font-weight: 600;
    font-size: 14px;
  }
  .map-visualization {
    background: #1a1a1a;
    border: 1px solid #444;
    border-radius: 8px;
    overflow: hidden;
    box-shadow: 0 2px 4px rgba(0,0,0,0.5);
    margin-bottom: 20px;
    position: relative;
    display: flex;
    flex-direction: row;
    align-items: center;
    justify-content: flex-start;
    min-height: 520px;
  }
  .map-visualization svg {
    margin-left: 0;
    margin-right: 0;
    flex: 1 1 0%;
    display: block;
  }
  .continent-legend.legend-overlay {
    position: absolute;
    top: 50%;
    right: 24px;
    transform: translateY(-50%);
    display: flex;
    flex-direction: column;
    gap: 18px;
    align-items: flex-start;
    min-width: 180px;
    background: #181818;
    border-radius: 8px;
    padding: 18px 18px 18px 18px;
    border: 1px solid #333;
    height: fit-content;
    box-shadow: 0 2px 4px rgba(0,0,0,0.2);
    z-index: 2;
    opacity: 0.97;
  }
  .legend-item {
    display: flex;
    align-items: center;
    gap: 8px;
    font-size: 1.1em;
    color: #e0e0e0;
  }
  .legend-color {
    width: 20px;
    height: 20px;
    border-radius: 4px;
    display: inline-block;
    border: 2px solid #222;
  }
  @media (max-width: 900px) {
    .continent-legend.legend-overlay {
      position: static;
      transform: none;
      margin: 18px auto 0 auto;
      flex-direction: row;
      justify-content: center;
      align-items: center;
      min-width: 0;
      width: 100%;
      padding: 12px 0 0 0;
      background: none;
      border: none;
      box-shadow: none;
    }
    .map-visualization {
      flex-direction: column;
      min-height: 520px;
    }
  }
  :global(.map-tooltip) {
    font-family: 'Arial', sans-serif;
  }
</style> 