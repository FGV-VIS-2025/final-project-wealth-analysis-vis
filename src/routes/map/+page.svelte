<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import { base } from '$app/paths';
  
  let mapContainer;
  let billionairesData = [];
  let mapLoaded = false;
  let Plotly;
  let loadingError = false;
  let errorMessage = '';
  
  // Função para processar dados e construir a visualização
  function buildMapVisualization(data) {
    try {
      console.log("Iniciando construção do mapa com", data.length, "registros");
      
      // Extrair coordenadas geográficas
      const geoCoordinates = {};
      
      // Primeiro, colete todas as coordenadas disponíveis
      data.forEach(b => {
        if (b.country && b.latitude && b.longitude) {
          geoCoordinates[b.country] = {
            lat: +b.latitude,
            lon: +b.longitude
          };
        }
      });
      
      // Verificar campos alternativos de coordenadas
      const possibleLatFields = ['latitude', 'latitude_country', 'lat'];
      const possibleLonFields = ['longitude', 'longitude_country', 'lon', 'lng'];
      
      let latField = null;
      let lonField = null;
      
      // Detectar quais campos de coordenadas estão disponíveis
      for (const field of possibleLatFields) {
        if (data[0] && data[0][field] !== undefined) {
          latField = field;
          break;
        }
      }
      
      for (const field of possibleLonFields) {
        if (data[0] && data[0][field] !== undefined) {
          lonField = field;
          break;
        }
      }
      
      console.log("Campos de coordenadas detectados:", latField, lonField);
      
      // Se encontramos campos de coordenadas, usá-los para popular geoCoordinates
      if (latField && lonField) {
        data.forEach(b => {
          if (b.country && b[latField] && b[lonField]) {
            geoCoordinates[b.country] = {
              lat: +b[latField],
              lon: +b[lonField]
            };
          }
        });
      }
      
      console.log("Coordenadas coletadas para", Object.keys(geoCoordinates).length, "países");
      
      // Agrupar bilionários por país atual e país de cidadania
      const countryConnections = [];
      
      // Estrutura para rastrear coordenadas por país
      const citizenshipCoordinates = {...geoCoordinates};
      
      data.forEach(billionaire => {
        const country = billionaire.country;
        const citizenship = billionaire.countryOfCitizenship;
        
        // Verifique se temos pelo menos país e cidadania
        if (country && citizenship && country !== citizenship) {
          // Se temos coordenadas para o país atual
          if (geoCoordinates[country]) {
            // Adicionar à lista de conexões
            countryConnections.push({
              name: billionaire.personName || "Anônimo",
              currentCountry: country,
              citizenshipCountry: citizenship,
              worth: +billionaire.finalWorth || 0,
              lat: geoCoordinates[country].lat,
              lon: geoCoordinates[country].lon
            });
            
            // Se o país de cidadania não tem coordenadas ainda, mas temos os campos de lat/long
            if (!citizenshipCoordinates[citizenship] && latField && lonField && billionaire[latField] && billionaire[lonField]) {
              citizenshipCoordinates[citizenship] = {
                lat: +billionaire[latField],
                lon: +billionaire[lonField]
              };
            }
          }
        }
      });
      
      // Fazer uma busca mais agressiva para coordenadas de países de cidadania
      data.forEach(b => {
        const citizenship = b.countryOfCitizenship;
        if (citizenship && !citizenshipCoordinates[citizenship] && latField && lonField && b[latField] && b[lonField]) {
          citizenshipCoordinates[citizenship] = {
            lat: +b[latField],
            lon: +b[lonField]
          };
        }
      });
      
      console.log("Conexões entre países:", countryConnections.length);
      console.log("Coordenadas para países de cidadania:", Object.keys(citizenshipCoordinates).length);
      
      // Se não tivermos conexões suficientes, tentar ser menos restritivo
      if (countryConnections.length < 10) {
        console.log("Poucas conexões encontradas, tentando abordagem alternativa");
        
        countryConnections.length = 0; // Limpar conexões anteriores
        
        data.forEach(billionaire => {
          const country = billionaire.country;
          const citizenship = billionaire.countryOfCitizenship;
          
          if (country && citizenship && country !== citizenship && latField && lonField && billionaire[latField] && billionaire[lonField]) {
            // Usar a mesma coordenada para ambos (simplificação)
            const lat = +billionaire[latField];
            const lon = +billionaire[lonField];
            
            countryConnections.push({
              name: billionaire.personName || "Anônimo",
              currentCountry: country,
              citizenshipCountry: citizenship,
              worth: +billionaire.finalWorth || 0,
              lat: lat,
              lon: lon
            });
            
            // Adicionar coordenadas para o país de cidadania e residência
            citizenshipCoordinates[country] = { lat, lon };
            citizenshipCoordinates[citizenship] = { lat, lon };
          }
        });
        
        console.log("Novas conexões encontradas:", countryConnections.length);
      }
      
      // Se ainda não temos dados suficientes, criar um mapa simples de países
      if (countryConnections.length < 5) {
        console.log("Criando mapa simplificado com os dados disponíveis");
        return createSimpleCountryMap(data);
      }
      
      // Dados para o mapa
      const mapData = [];
      
      // Adicionar linhas de conexão (migrações)
      if (countryConnections.length > 0) {
        // Criar linhas de conexão para os top 100 bilionários (ou menos se não tivermos tantos)
        const topConnections = countryConnections
          .sort((a, b) => b.worth - a.worth)
          .slice(0, Math.min(100, countryConnections.length));
        
        // Adicionar linhas entre países apenas se tivermos coordenadas
        const validLines = {
          lat: [],
          lon: []
        };
        
        topConnections.forEach(conn => {
          // Só adicionar linhas se tivermos coordenadas para o país de cidadania
          const startCoord = citizenshipCoordinates[conn.citizenshipCountry];
          if (startCoord && conn.lat && conn.lon) {
            validLines.lat.push(startCoord.lat, conn.lat, null);
            validLines.lon.push(startCoord.lon, conn.lon, null);
          }
        });
        
        if (validLines.lat.length > 0) {
          console.log("Adicionando", validLines.lat.length / 3, "linhas de conexão");
          mapData.push({
            type: 'scattergeo',
            mode: 'lines',
            lat: validLines.lat,
            lon: validLines.lon,
            line: {
              width: 1.5,
              color: 'rgba(140, 86, 75, 0.5)'
            },
            hoverinfo: 'none',
            name: 'Migrações de Bilionários'
          });
        }
      }
      
      // Adicionar pontos para países de residência atual
      const residenceData = d3.rollups(
        countryConnections,
        v => ({
          count: v.length,
          totalWorth: d3.sum(v, d => d.worth),
          billionaires: v.map(b => b.name).slice(0, 5).join(', ') // Limitar para não ficar muito grande
        }),
        d => d.currentCountry
      ).map(([country, data]) => {
        const sample = countryConnections.find(c => c.currentCountry === country);
        return {
          country,
          lat: sample.lat,
          lon: sample.lon,
          count: data.count,
          totalWorth: data.totalWorth,
          billionaires: data.billionaires
        };
      });
      
      mapData.push({
        type: 'scattergeo',
        lat: residenceData.map(d => d.lat),
        lon: residenceData.map(d => d.lon),
        text: residenceData.map(d => 
          `<b>${d.country}</b><br>` +
          `Bilionários: ${d.count}<br>` +
          `Patrimônio Total: $${(d.totalWorth/1000).toFixed(1)} bilhões`
        ),
        marker: {
          size: residenceData.map(d => Math.max(7, Math.sqrt(d.count) * 7)),
          color: 'rgb(31, 119, 180)',
          line: {
            color: 'white',
            width: 1
          },
          opacity: 0.8
        },
        name: 'Países de Residência',
        hoverinfo: 'text',
        hoverlabel: {
          bgcolor: 'rgba(255, 255, 255, 0.9)',
          bordercolor: '#1f77b4',
          font: { color: '#333', family: 'Arial, sans-serif' }
        }
      });
      
      // Adicionar pontos para países de cidadania
      // Compilar dados por país de cidadania
      const citizenshipByCountry = {};
      
      countryConnections.forEach(conn => {
        if (!citizenshipByCountry[conn.citizenshipCountry]) {
          citizenshipByCountry[conn.citizenshipCountry] = {
            country: conn.citizenshipCountry,
            count: 0,
            totalWorth: 0,
            billionaires: []
          };
        }
        
        citizenshipByCountry[conn.citizenshipCountry].count += 1;
        citizenshipByCountry[conn.citizenshipCountry].totalWorth += conn.worth;
        if (citizenshipByCountry[conn.citizenshipCountry].billionaires.length < 5) {
          citizenshipByCountry[conn.citizenshipCountry].billionaires.push(conn.name);
        }
      });
      
      // Filtrar apenas países com coordenadas e adicionar lat/lon
      const citizenshipData = Object.values(citizenshipByCountry)
        .filter(d => citizenshipCoordinates[d.country])
        .map(d => ({
          ...d,
          lat: citizenshipCoordinates[d.country].lat,
          lon: citizenshipCoordinates[d.country].lon
        }));
      
      console.log("Dados de países de cidadania:", citizenshipData.length);
      
      if (citizenshipData.length > 0) {
        mapData.push({
          type: 'scattergeo',
          lat: citizenshipData.map(d => d.lat),
          lon: citizenshipData.map(d => d.lon),
          text: citizenshipData.map(d => 
            `<b>${d.country}</b><br>` +
            `Cidadania de ${d.count} bilionários<br>` +
            `Patrimônio Total: $${(d.totalWorth/1000).toFixed(1)} bilhões`
          ),
          marker: {
            size: citizenshipData.map(d => Math.max(7, Math.sqrt(d.count) * 5)),
            color: 'rgb(255, 127, 14)',
            symbol: 'circle-open',
            line: {
              color: 'rgb(255, 127, 14)',
              width: 2
            },
            opacity: 0.9
          },
          name: 'Países de Cidadania',
          hoverinfo: 'text',
          hoverlabel: {
            bgcolor: 'rgba(255, 255, 255, 0.9)',
            bordercolor: '#ff7f0e',
            font: { color: '#333', family: 'Arial, sans-serif' }
          }
        });
      } else {
        console.log("ALERTA: Sem dados suficientes para mostrar países de cidadania!");
        // Adicionar aviso na interface
        loadingError = true;
        errorMessage = "Não foi possível exibir países de cidadania devido a dados insuficientes.";
      }
      
      // Configurar layout
      const layout = {
        title: {
          text: 'Mapa de Migração de Bilionários: Cidadania vs. Residência',
          font: { 
            family: 'Arial, sans-serif',
            size: 24,
            color: '#333'
          },
          pad: { t: 20, b: 20, l: 20, r: 20 }
        },
        geo: {
          scope: 'world',
          projection: {
            type: 'natural earth'
          },
          showland: true,
          landcolor: 'rgb(250, 250, 250)',
          showocean: true,
          oceancolor: 'rgba(220, 240, 255, 0.8)',
          showlakes: true,
          lakecolor: 'rgb(220, 240, 255)',
          showcountries: true,
          countrycolor: 'rgb(220, 220, 220)',
          showcoastlines: true,
          coastlinecolor: 'rgb(200, 200, 200)',
          resolution: 50
        },
        paper_bgcolor: 'rgba(250, 250, 250, 0.9)',
        plot_bgcolor: 'rgba(250, 250, 250, 0.9)',
        margin: { l: 0, r: 0, t: 50, b: 20 },
        annotations: [{
          x: 1.0,
          y: -0.05,
          xref: 'paper',
          yref: 'paper',
          text: 'Fonte: Dataset Global de Bilionários',
          showarrow: false,
          font: {
            family: 'Arial, sans-serif',
            size: 12,
            color: '#555'
          }
        }],
        legend: {
          x: 0.01,
          y: 0.99,
          bgcolor: 'rgba(255, 255, 255, 0.7)',
          bordercolor: '#ddd',
          borderwidth: 1
        }
      };
      
      // Opções de configuração
      const config = {
        responsive: true,
        displayModeBar: true,
        modeBarButtonsToRemove: ['lasso2d', 'select2d'],
        displaylogo: false,
        toImageButtonOptions: {
          format: 'png',
          filename: 'migration_billionaires_map',
          height: 800,
          width: 1200,
          scale: 2
        }
      };
      
      // Renderizar o mapa
      Plotly.newPlot(mapContainer, mapData, layout, config);
      mapLoaded = true;
      
    } catch (error) {
      console.error("Erro ao construir visualização:", error);
      loadingError = true;
      errorMessage = error.message || "Ocorreu um erro ao processar os dados.";
    }
  }
  
  // Função para criar um mapa simples quando não temos dados suficientes
  function createSimpleCountryMap(data) {
    try {
      console.log("Criando mapa simples com distribuição de bilionários por país");
      
      // Agrupar bilionários por país
      const countryCounts = d3.rollups(
        data.filter(d => d.country),
        v => ({
          count: v.length,
          totalWorth: d3.sum(v, d => +d.finalWorth || 0)
        }),
        d => d.country
      );
      
      // Coletar coordenadas disponíveis
      const latFields = ['latitude', 'latitude_country', 'lat'];
      const lonFields = ['longitude', 'longitude_country', 'lon', 'lng'];
      
      let latField = null;
      let lonField = null;
      
      // Detectar campos de coordenadas
      for (const field of latFields) {
        if (data.some(d => d[field])) {
          latField = field;
          break;
        }
      }
      
      for (const field of lonFields) {
        if (data.some(d => d[field])) {
          lonField = field;
          break;
        }
      }
      
      if (!latField || !lonField) {
        throw new Error("Dados de coordenadas geográficas não encontrados no dataset.");
      }
      
      // Obter coordenadas para cada país
      const countryCoordinates = {};
      data.forEach(b => {
        if (b.country && b[latField] && b[lonField]) {
          countryCoordinates[b.country] = {
            lat: +b[latField],
            lon: +b[lonField]
          };
        }
      });
      
      // Mapear países com suas coordenadas
      const countryData = countryCounts
        .filter(([country]) => countryCoordinates[country])
        .map(([country, data]) => ({
          country,
          count: data.count,
          totalWorth: data.totalWorth,
          lat: countryCoordinates[country].lat,
          lon: countryCoordinates[country].lon
        }));
      
      if (countryData.length === 0) {
        throw new Error("Não foi possível encontrar dados suficientes para criar o mapa.");
      }
      
      // Criar mapa simples
      const mapData = [{
        type: 'scattergeo',
        lat: countryData.map(d => d.lat),
        lon: countryData.map(d => d.lon),
        text: countryData.map(d => 
          `<b>${d.country}</b><br>` +
          `Número de Bilionários: ${d.count}<br>` +
          `Patrimônio Total: $${(d.totalWorth/1000).toFixed(1)} bilhões`
        ),
        marker: {
          size: countryData.map(d => Math.max(5, Math.sqrt(d.count) * 7)),
          color: 'rgb(31, 119, 180)',
          line: {
            color: 'white',
            width: 1
          },
          opacity: 0.8
        },
        name: 'Países com Bilionários',
        hoverinfo: 'text',
        hoverlabel: {
          bgcolor: 'rgba(255, 255, 255, 0.9)',
          bordercolor: '#1f77b4',
          font: { color: '#333', family: 'Arial, sans-serif' }
        }
      }];
      
      const layout = {
        title: {
          text: 'Distribuição Global de Bilionários por País',
          font: { 
            family: 'Arial, sans-serif',
            size: 24,
            color: '#333'
          }
        },
        geo: {
          scope: 'world',
          projection: { type: 'natural earth' },
          showland: true,
          landcolor: 'rgb(250, 250, 250)',
          showocean: true,
          oceancolor: 'rgba(220, 240, 255, 0.8)',
          showcountries: true,
          countrycolor: 'rgb(220, 220, 220)'
        },
        margin: { l: 0, r: 0, t: 50, b: 0 }
      };
      
      const config = { responsive: true };
      
      Plotly.newPlot(mapContainer, mapData, layout, config);
      mapLoaded = true;
      
    } catch (error) {
      console.error("Erro ao criar mapa simples:", error);
      loadingError = true;
      errorMessage = error.message || "Não foi possível criar o mapa com os dados disponíveis.";
    }
  }
  
  onMount(async () => {
    try {
      console.log("Iniciando carregamento do mapa");
      
      // Carregar Plotly de forma assíncrona
      Plotly = (await import('plotly.js-dist-min')).default;
      console.log("Plotly carregado com sucesso");
      
      // Carregar dados dos bilionários
      const csvPath = `${base}/Billionaires Statistics Dataset.csv`;
      console.log("Carregando dados de:", csvPath);
      
      billionairesData = await d3.csv(csvPath);
      console.log("Dados carregados:", billionairesData.length, "registros");
      
      // Examinar estrutura dos dados
      if (billionairesData.length > 0) {
        console.log("Exemplo de registro:", billionairesData[0]);
        console.log("Propriedades disponíveis:", Object.keys(billionairesData[0]));
        
        // Construir a visualização
        buildMapVisualization(billionairesData);
      } else {
        throw new Error("Nenhum dado encontrado no arquivo CSV.");
      }
    } catch (error) {
      console.error("Erro no carregamento:", error);
      loadingError = true;
      errorMessage = error.message || "Erro ao carregar os dados ou a biblioteca Plotly.";
    }
  });
</script>

<svelte:head>
  <title>Mapa de Bilionários: Cidadania vs. Residência</title>
  <meta name="description" content="Visualização da distribuição global de bilionários, comparando países de cidadania e residência atual" />
</svelte:head>

<div class="story-container">
  <nav class="main-nav">
    <h1 class="site-title">Análise Global de Bilionários</h1>
    <div class="nav-links">
      <a href="{base}/" class="nav-link">Visão Geral</a>
      <a href="{base}/gender-gap" class="nav-link">Gap de Gênero</a>
      <a href="{base}/map" class="nav-link active">Mapa de Migração</a>
      <a href="{base}/network" class="nav-link">Rede de Indústrias</a>
      <a href="{base}/ascensao" class="nav-link">Trajetórias da Fortuna</a>
    </div>
  </nav>

  <section class="map-section">
    <header>
      <h2>Mapa Global de Migração de Bilionários</h2>
      <p class="subtitle">Comparação entre países de cidadania e países de residência atual</p>
    </header>
    
    <div class="map-container">
      {#if !mapLoaded}
        <div class="loading">
          {#if loadingError}
            <div class="error-message">
              <p>Erro ao carregar o mapa:</p>
              <p>{errorMessage}</p>
              <p class="hint">Tente recarregar a página ou verifique o console para mais detalhes.</p>
            </div>
          {:else}
            <p>Carregando mapa...</p>
            <div class="spinner"></div>
          {/if}
        </div>
      {/if}
      <div bind:this={mapContainer} class="map-visualization"></div>
    </div>
    
    <div class="info-grid">
      <div class="info-card">
        <h3>Interpretando o Mapa</h3>
        <ul>
          <li><span class="circle blue"></span> <strong>Pontos Azuis</strong>: Países de residência atual dos bilionários</li>
          <li><span class="circle orange"></span> <strong>Círculos Laranja</strong>: Países de cidadania dos bilionários</li>
          <li><strong>Linhas</strong>: Conexões entre país de cidadania e residência atual</li>
        </ul>
        <p>O tamanho de cada ponto indica a quantidade de bilionários. Passe o mouse sobre os pontos para ver mais detalhes.</p>
      </div>
      
      <div class="info-card">
        <h3>Sobre os Dados</h3>
        <p>Esta visualização mostra como os bilionários se movem globalmente, indicando possíveis "fugas de cérebros" e os fluxos de capital e talento pelo mundo.</p>
        <p>Países com grande presença de bilionários não-nativos podem indicar ambientes favoráveis para negócios, regimes fiscais atrativos ou alta qualidade de vida.</p>
      </div>
    </div>
  </section>
</div>

<style>
  :global(html) {
    font-family: 'Inter', Arial, sans-serif;
    background: #f4f6fa;
    color: #23272f;
  }

  .story-container {
    max-width: 1200px;
    margin: 0 auto;
    padding: 32px 16px;
  }
  
  .main-nav {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 40px;
    padding-bottom: 16px;
    border-bottom: 1px solid #e3e8ee;
  }
  
  .site-title {
    font-size: 1.8em;
    font-weight: 700;
    color: #1a1a1a;
    margin: 0;
  }
  
  .nav-links {
    display: flex;
    gap: 20px;
  }
  
  .nav-link {
    text-decoration: none;
    color: #555;
    font-weight: 500;
    padding: 8px 12px;
    border-radius: 6px;
    transition: all 0.2s ease;
  }
  
  .nav-link:hover {
    background-color: #edf2f7;
    color: #2c5282;
  }
  
  .nav-link.active {
    background-color: #ebf4ff;
    color: #2b6cb0;
  }
  
  .map-section {
    background: #fff;
    border-radius: 18px;
    box-shadow: 0 4px 24px rgba(44,62,80,0.08), 0 1.5px 4px rgba(44,62,80,0.04);
    overflow: hidden;
    border: 1px solid #e3e8ee;
    margin-bottom: 32px;
  }
  
  header {
    text-align: center;
    padding: 32px 24px 24px;
    background: linear-gradient(180deg, #f8fafc 0%, #f1f5f9 100%);
    border-bottom: 1px solid #e3e8ee;
  }
  
  h2 {
    font-size: 2.2em;
    color: #1a1a1a;
    margin: 0 0 12px 0;
    font-weight: 700;
    letter-spacing: -0.5px;
  }
  
  .subtitle {
    font-size: 1.2em;
    color: #555;
    margin: 0;
  }
  
  .map-container {
    position: relative;
    width: 100%;
    height: 600px;
    border-bottom: 1px solid #e3e8ee;
  }
  
  .map-visualization {
    width: 100%;
    height: 100%;
  }
  
  .loading {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    background: rgba(255, 255, 255, 0.8);
    z-index: 10;
  }
  
  .loading p {
    font-size: 1.2em;
    margin-bottom: 16px;
    color: #333;
  }
  
  .error-message {
    padding: 24px;
    text-align: center;
    max-width: 80%;
  }
  
  .error-message p {
    margin-bottom: 12px;
    color: #e53e3e;
  }
  
  .error-message .hint {
    color: #718096;
    font-size: 0.9em;
    margin-top: 8px;
  }
  
  .spinner {
    width: 40px;
    height: 40px;
    border: 4px solid rgba(0, 0, 0, 0.1);
    border-radius: 50%;
    border-top-color: #3498db;
    animation: spin 1s ease-in-out infinite;
  }
  
  @keyframes spin {
    to { transform: rotate(360deg); }
  }
  
  .info-grid {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 24px;
    padding: 32px;
  }
  
  .info-card {
    background: #f8fafc;
    border-radius: 12px;
    padding: 24px;
    box-shadow: 0 2px 12px rgba(44,62,80,0.06);
    border: 1px solid #e3e8ee;
  }
  
  .info-card h3 {
    font-size: 1.4em;
    margin: 0 0 16px 0;
    color: #333;
    font-weight: 600;
  }
  
  .info-card p, .info-card li {
    font-size: 1em;
    line-height: 1.6;
    color: #444;
    margin-bottom: 12px;
  }
  
  .info-card ul {
    padding-left: 0;
    list-style-type: none;
    margin-bottom: 16px;
  }
  
  .info-card li {
    display: flex;
    align-items: center;
    margin-bottom: 12px;
  }
  
  .circle {
    display: inline-block;
    width: 12px;
    height: 12px;
    border-radius: 50%;
    margin-right: 8px;
  }
  
  .blue {
    background-color: rgb(31, 119, 180);
  }
  
  .orange {
    background-color: rgb(255, 127, 14);
    border: 1px solid rgb(255, 127, 14);
  }
  
  @media (max-width: 900px) {
    .main-nav {
      flex-direction: column;
      gap: 12px;
      text-align: center;
    }
    
    .info-grid {
      grid-template-columns: 1fr;
      padding: 24px 16px;
    }
    
    .map-container {
      height: 450px;
    }
    
    h2 {
      font-size: 1.8em;
    }
    
    .subtitle {
      font-size: 1em;
    }
    
    .site-title {
      font-size: 1.6em;
    }
  }
  
  @media (max-width: 600px) {
    .story-container {
      padding: 16px 8px;
    }
    
    .map-container {
      height: 350px;
    }
    
    h2 {
      font-size: 1.5em;
    }
    
    header {
      padding: 24px 16px 20px;
    }
    
    .info-grid {
      padding: 16px 12px;
      gap: 16px;
    }
    
    .info-card {
      padding: 16px;

    }
    
    .info-card h3 {
      font-size: 1.2em;
    }
  }
</style> 
