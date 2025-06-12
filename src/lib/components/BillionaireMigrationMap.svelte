<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';

  export let data = [];
	export let selectedIndustries = [];
	export let selectedGender = 'Todos';
  
  // Filtros locais
  let filterIndustry = 'all';
  
  let svgElement;
	let worldData = null;
	let filteredData = [];
	let migrationFlows = [];
  let selectedCountry = null;
	let tooltip = { show: false, x: 0, y: 0, content: '' };
	let countryCoordinates = new Map();
	
	// Controles para filtrar fluxos
	let showEmigration = true;
	let showImmigration = true;

	// Mapeamento de nomes de países (dados → GeoJSON)
	const countryNameMap = {
		'United States': 'United States of America',
		'United Kingdom': 'United Kingdom',
		'South Korea': 'South Korea',
		'Hong Kong': 'Hong Kong S.A.R.',
		'Russia': 'Russia',
		'China': 'China',
		'Germany': 'Germany',
		'India': 'India',
		'Brazil': 'Brazil',  
		'Canada': 'Canada',
		'France': 'France',
		'Switzerland': 'Switzerland',
		'Italy': 'Italy',
		'Japan': 'Japan',
		'Australia': 'Australia',
		'Singapore': 'Singapore',
		'Israel': 'Israel',
		'Turkey': 'Turkey',
		'Taiwan': 'Taiwan',
		'Thailand': 'Thailand',
		'Netherlands': 'Netherlands',
		'Sweden': 'Sweden',
		'Belgium': 'Belgium',
		'Norway': 'Norway',
		'Spain': 'Spain',
		'Mexico': 'Mexico',
		'Indonesia': 'Indonesia',
		'Philippines': 'Philippines',
		'Ireland': 'Ireland',
		'Finland': 'Finland',
		'Denmark': 'Denmark',
		'Austria': 'Austria',
		'Chile': 'Chile',
		'Czech Republic': 'Czech Republic',
		'New Zealand': 'New Zealand',
		'Poland': 'Poland',
		'Ukraine': 'Ukraine',
		'Colombia': 'Colombia',
		'Argentina': 'Argentina',
		'Portugal': 'Portugal',
		'Greece': 'Greece',
		'South Africa': 'South Africa',
		'Egypt': 'Egypt',
		'Nigeria': 'Nigeria',
		'Morocco': 'Morocco',
		'United Arab Emirates': 'United Arab Emirates',
		'Saudi Arabia': 'Saudi Arabia',
		'Kuwait': 'Kuwait',
		'Qatar': 'Qatar',
		'Oman': 'Oman',
		'Lebanon': 'Lebanon',
		'Cyprus': 'Cyprus',
		'Malta': 'Malta',
		'Iceland': 'Iceland',
		'Luxembourg': 'Luxembourg',
		'Liechtenstein': 'Liechtenstein',
		'Monaco': 'Monaco',
		'Kazakhstan': 'Kazakhstan',
		'Uzbekistan': 'Uzbekistan',
		'Georgia': 'Georgia',
		'Armenia': 'Armenia',
		'Estonia': 'Estonia',
		'Latvia': 'Latvia',
		'Lithuania': 'Lithuania',
		'Slovakia': 'Slovakia',
		'Slovenia': 'Slovenia',
		'Croatia': 'Croatia',
		'Hungary': 'Hungary',
		'Romania': 'Romania',
		'Bulgaria': 'Bulgaria',
		'Serbia': 'Serbia',
		'Montenegro': 'Montenegro',
		'North Macedonia': 'North Macedonia',
		'Albania': 'Albania',
		'Bosnia and Herzegovina': 'Bosnia and Herzegovina',
		'Moldova': 'Moldova',
		'Belarus': 'Belarus',
		'Vietnam': 'Vietnam',
		'Myanmar': 'Myanmar',
		'Cambodia': 'Cambodia',
		'Laos': 'Laos',
		'Mongolia': 'Mongolia',
		'Nepal': 'Nepal',
		'Bangladesh': 'Bangladesh',
		'Sri Lanka': 'Sri Lanka',
		'Maldives': 'Maldives',
		'Bhutan': 'Bhutan',
		'Pakistan': 'Pakistan',
		'Afghanistan': 'Afghanistan',
		'Iran': 'Iran',
		'Iraq': 'Iraq',
		'Jordan': 'Jordan',
		'Syria': 'Syria',
		'Yemen': 'Yemen',
		'Bahrain': 'Bahrain',
		'Algeria': 'Algeria',
		'Tunisia': 'Tunisia',
		'Libya': 'Libya',
		'Sudan': 'Sudan',
		'Ethiopia': 'Ethiopia',
		'Kenya': 'Kenya',
		'Tanzania': 'Tanzania',
		'Uganda': 'Uganda',
		'Rwanda': 'Rwanda',
		'Burundi': 'Burundi',
		'Madagascar': 'Madagascar',
		'Mauritius': 'Mauritius',
		'Seychelles': 'Seychelles',
		'Comoros': 'Comoros',
		'Djibouti': 'Djibouti',
		'Eritrea': 'Eritrea',
		'Somalia': 'Somalia',
		'Ghana': 'Ghana',
		'Ivory Coast': "Côte d'Ivoire",
		'Senegal': 'Senegal',
		'Mali': 'Mali',
		'Burkina Faso': 'Burkina Faso',
		'Niger': 'Niger',
		'Chad': 'Chad',
		'Central African Republic': 'Central African Republic',
		'Cameroon': 'Cameroon',
		'Equatorial Guinea': 'Equatorial Guinea',
		'Gabon': 'Gabon',
		'Republic of the Congo': 'Republic of the Congo',
		'Democratic Republic of the Congo': 'Democratic Republic of the Congo',
		'Angola': 'Angola',
		'Zambia': 'Zambia',
		'Zimbabwe': 'Zimbabwe',
		'Botswana': 'Botswana',
		'Namibia': 'Namibia',
		'Lesotho': 'Lesotho',
		'Swaziland': 'Eswatini',
		'Malawi': 'Malawi',
		'Mozambique': 'Mozambique',
		'Venezuela': 'Venezuela',
		'Guyana': 'Guyana',
		'Suriname': 'Suriname',
		'Uruguay': 'Uruguay',
		'Paraguay': 'Paraguay',
		'Bolivia': 'Bolivia',
		'Peru': 'Peru',
		'Ecuador': 'Ecuador',
		'Panama': 'Panama',
		'Costa Rica': 'Costa Rica',
		'Nicaragua': 'Nicaragua',
		'Honduras': 'Honduras',
		'El Salvador': 'El Salvador',
		'Guatemala': 'Guatemala',
		'Belize': 'Belize',
		'Jamaica': 'Jamaica',
		'Haiti': 'Haiti',
		'Dominican Republic': 'Dominican Republic',
		'Cuba': 'Cuba',
		'Bahamas': 'Bahamas',
		'Barbados': 'Barbados',
		'Trinidad and Tobago': 'Trinidad and Tobago',
		'Grenada': 'Grenada',
		'Saint Vincent and the Grenadines': 'Saint Vincent and the Grenadines',
		'Saint Lucia': 'Saint Lucia',
		'Dominica': 'Dominica',
		'Antigua and Barbuda': 'Antigua and Barbuda',
		'Saint Kitts and Nevis': 'Saint Kitts and Nevis'
	};

	// Cores por continente harmonizadas com o radar chart
  const continentColors = {
		'North America': '#45b7d1',    // Azul claro (do radar)
		'South America': '#f39c12',    // Laranja dourado (do radar)
		'Europe': '#2ecc71',           // Verde (do radar)
		'Asia': '#e74c3c',            // Vermelho (do radar)
		'Africa': '#9b59b6',          // Roxo (do radar)
		'Oceania': '#d35400',         // Laranja escuro
		'Middle East': '#4ecdc4'      // Turquesa (do radar)
	};

	// Extrair indústrias dos dados
	$: uniqueIndustries = data && data.length > 0 
		? ['all', ...new Set(data.map(d => d.industries).filter(Boolean)).values()].sort()
		: ['all'];

	// Carregar dados do mundo
	onMount(async () => {
		try {
			// Importar base path do SvelteKit
			const { base } = await import('$app/paths');
			const response = await fetch(`${base}/countries.geojson`);
			worldData = await response.json();
		} catch (error) {
			console.error('Erro ao carregar dados do mundo:', error);
		}
	});

	// Filtrar dados
	$: {
		if (data && data.length > 0) {
			filteredData = data.filter(d => {
				// Filtro local por indústria
				const localIndustryMatch = filterIndustry === 'all' || d.industries === filterIndustry;
				// Se selectedIndustries está vazio, incluir todos
				const industryMatch = !selectedIndustries || selectedIndustries.length === 0 || 
					selectedIndustries.includes(d.industries);
				// Se selectedGender não está definido ou é 'Todos', incluir todos
				const genderMatch = !selectedGender || selectedGender === 'Todos' || 
					d.gender === selectedGender;
				return localIndustryMatch && industryMatch && genderMatch && d.country && 
					d.country !== 'Unknown' && d.country.trim() !== '';
			});
		} else {
			filteredData = [];
		}
		

	}

	// Calcular migração (cidadania → residência)
	// LÓGICA:
	// - countryOfCitizenship = país de nascimento/cidadania (ORIGEM)
	// - country = país atual de residência (DESTINO)
	// - Se diferentes = houve migração de origem → destino
	$: {
		if (filteredData && filteredData.length > 0) {
			const migrationCounts = new Map();
			
			filteredData.forEach(d => {
				if (d.countryOfCitizenship && d.country && 
					d.countryOfCitizenship !== d.country && 
					d.country !== 'Unknown' && d.countryOfCitizenship !== 'Unknown') {
					
					const key = `${d.countryOfCitizenship}-${d.country}`;
					migrationCounts.set(key, (migrationCounts.get(key) || 0) + 1);
				}
			});

			migrationFlows = Array.from(migrationCounts.entries())
				.filter(([, count]) => count >= 1)
				.map(([key, count]) => {
					const [origin, destination] = key.split('-');
					return { 
						origin,          // País de cidadania (saiu daqui)
						destination,     // País atual (chegou aqui)
						count 
					};
				})
				.sort((a, b) => b.count - a.count); // Ordenar por volume
		} else {
			migrationFlows = [];
		}
	}

	// Limpar seleção quando filtros mudarem
	$: {
		if (filterIndustry || selectedIndustries || selectedGender) {
			selectedCountry = null; // Reset quando filtros mudarem
		}
	}

	// Renderizar mapa
	$: {
		if (svgElement && worldData && filteredData.length > 0) {
			renderMap();
		}
	}

	// Limpar fluxos quando filtros mudarem
	$: {
		if (svgElement && (filterIndustry || selectedIndustries || selectedGender)) {
			renderFlows(); // Vai limpar automaticamente se selectedCountry for null
		}
	}

	function normalizeCountryName(name) {
		return countryNameMap[name] || name;
	}

	function isCountryMatch(geoName, dataCountry) {
		// Normalizar nomes
		const normalizedGeoName = geoName.toLowerCase().trim();
		const normalizedDataCountry = dataCountry.toLowerCase().trim();
		const mappedDataCountry = normalizeCountryName(dataCountry).toLowerCase().trim();
		
		// Correspondências exatas
		if (normalizedGeoName === normalizedDataCountry || 
			normalizedGeoName === mappedDataCountry) {
			return true;
		}
		
		// Correspondências especiais
		const specialMatches = {
			'united states of america': ['united states', 'usa', 'us'],
			'united kingdom': ['uk', 'britain', 'great britain'],
			'south korea': ['korea', 'republic of korea'],
			'hong kong s.a.r.': ['hong kong'],
			'russia': ['russian federation'],
			'china': ['people\'s republic of china'],
			'taiwan': ['republic of china'],
			'iran': ['islamic republic of iran'],
			'syria': ['syrian arab republic'],
			'vietnam': ['viet nam'],
			'laos': ['lao people\'s democratic republic'],
			'north korea': ['democratic people\'s republic of korea'],
			'macedonia': ['north macedonia'],
			'czech republic': ['czechia'],
			'ivory coast': ['côte d\'ivoire'],
			'swaziland': ['eswatini'],
			'myanmar': ['burma'],
			'democratic republic of the congo': ['congo', 'dr congo', 'zaire'],
			'republic of the congo': ['congo-brazzaville']
		};
		
		// Verificar correspondências especiais
		for (let [standard, alternatives] of Object.entries(specialMatches)) {
			if (normalizedGeoName === standard && 
				alternatives.some(alt => normalizedDataCountry.includes(alt) || alt.includes(normalizedDataCountry))) {
				return true;
			}
			if (alternatives.includes(normalizedGeoName) && 
				(normalizedDataCountry.includes(standard) || standard.includes(normalizedDataCountry))) {
				return true;
			}
		}
		
		// Correspondências parciais (com mais de 3 caracteres)
		if (normalizedDataCountry.length > 3 && normalizedGeoName.length > 3) {
			if (normalizedGeoName.includes(normalizedDataCountry) || 
				normalizedDataCountry.includes(normalizedGeoName)) {
				return true;
			}
		}
		
		return false;
	}

	function renderMap() {
		const svg = d3.select(svgElement);
		svg.selectAll('*').remove();

		// Dimensões otimizadas para tela 918x857
		const width = 750; // Reduzido para caber melhor
		const height = 400; // Mantém proporção adequada

		// Projeção do mapa
		const projection = d3.geoNaturalEarth1()
			.scale(115) // Ajustado proporcionalmente
			.translate([width / 2, height / 2]);

		const path = d3.geoPath().projection(projection);

		// Contar bilionários por país de residência atual (não de cidadania)
		// IMPORTANTE: country = onde moram ATUALMENTE, countryOfCitizenship = origem
		const countryCounts = new Map();
		countryCoordinates.clear();
		
		filteredData.forEach(d => {
			const country = d.country; // País atual de residência
			if (country && country !== 'Unknown') {
				countryCounts.set(country, (countryCounts.get(country) || 0) + 1);
				
				// Armazenar coordenadas se disponíveis
				if (d.latitude_country && d.longitude_country) {
					countryCoordinates.set(country, {
						lat: +d.latitude_country,
						lng: +d.longitude_country
					});
        }
      }
    });





		// Grupo principal
		const g = svg.append('g');

		// Fundo do oceano harmonizado
		g.append('rect')
			.attr('width', width)
			.attr('height', height)
			.attr('fill', '#1a1a2e')
			.attr('stroke', 'none');

		// Renderizar países
		g.selectAll('.country')
			.data(worldData.features)
			.enter()
			.append('path')
			.attr('class', 'country')
			.attr('d', path)
			.attr('fill', d => {
				const geoName = d.properties.NAME || d.properties.name;
				
				// Encontrar correspondência nos dados
				let billionaireCount = 0;
				let continent = null;
				
				for (let [dataCountry, count] of countryCounts.entries()) {
					const normalizedDataCountry = normalizeCountryName(dataCountry);
					if (isCountryMatch(geoName, dataCountry)) {
						billionaireCount = count;
						// Determinar continente baseado na localização
						const centroid = d3.geoCentroid(d);
						continent = getContinentFromCoordinates(centroid);
						break;
					}
				}
				
				if (billionaireCount === 0) return '#34495e';
				
				const baseColor = continentColors[continent] || '#666666';
				
				// Intensidade baseada no número de bilionários
				const intensity = Math.min(billionaireCount / 10, 1);
				return d3.color(baseColor).brighter(0.3 - intensity * 0.3);
			})
			.attr('stroke', '#2c3e50')
			.attr('stroke-width', 0.8)
			.style('cursor', 'pointer')
			.on('click', function(event, d) {
				const geoName = d.properties.NAME || d.properties.name;
				
				// Encontrar país correspondente nos dados
				let matchedCountry = null;
				for (let dataCountry of countryCounts.keys()) {
					const normalizedDataCountry = normalizeCountryName(dataCountry);
					if (isCountryMatch(geoName, dataCountry)) {
						matchedCountry = dataCountry;
						break;
					}
				}
				
				if (matchedCountry && countryCounts.get(matchedCountry) > 0) {
					// Toggle: se é o mesmo país, desmarca e limpa fluxos
					if (selectedCountry === matchedCountry) {
						selectedCountry = null;
					} else {
						selectedCountry = matchedCountry;
					}
					// Sempre renderizar fluxos (incluindo limpeza quando selectedCountry é null)
					renderFlows();
				}
			})
			.on('mouseover', function(event, d) {
				// Remover highlight de todos os países, exceto o selecionado
				g.selectAll('.country')
					.attr('stroke', function(d) {
						const geoName = d.properties.NAME || d.properties.name;
						let matchedCountry = null;
						for (let dataCountry of countryCounts.keys()) {
							if (isCountryMatch(geoName, dataCountry) && dataCountry === selectedCountry) {
								matchedCountry = dataCountry;
								break;
							}
						}
						return matchedCountry ? '#ffffff' : '#2c3e50';
					})
					.attr('stroke-width', function(d) {
						const geoName = d.properties.NAME || d.properties.name;
						let matchedCountry = null;
						for (let dataCountry of countryCounts.keys()) {
							if (isCountryMatch(geoName, dataCountry) && dataCountry === selectedCountry) {
								matchedCountry = dataCountry;
								break;
							}
						}
						return matchedCountry ? 2 : 0.8;
					});

				// Highlight do país atual
				d3.select(this)
					.attr('stroke', '#ffffff')
					.attr('stroke-width', 1);

				// Tooltip mais clean
				const geoNameHover = d.properties.NAME || d.properties.name;
				let billionaireCount = 0;
				for (let [dataCountry, count] of countryCounts.entries()) {
					const normalizedDataCountry = normalizeCountryName(dataCountry);
					if (isCountryMatch(geoNameHover, dataCountry)) {
						billionaireCount = count;
						break;
					}
				}
				if (billionaireCount > 0) {
					tooltip = {
						show: true,
						x: event.offsetX + 10,
						y: event.offsetY - 10,
						content: `${geoNameHover}<br/><span style=\"color: #ffd700; font-weight: bold;\">${billionaireCount}</span> bilionários`
					};
				}
			})
			.on('mouseout', function() {
				const geoName = d.properties.NAME || d.properties.name;
				let matchedCountry = null;
				for (let dataCountry of countryCounts.keys()) {
					const normalizedDataCountry = normalizeCountryName(dataCountry);
					if (isCountryMatch(geoName, dataCountry)) {
						matchedCountry = dataCountry;
						break;
					}
				}
				
				// Só resetar estilo se não for o país selecionado
				if (selectedCountry !== matchedCountry) {
					d3.select(this)
						.attr('stroke', '#2c3e50')
						.attr('stroke-width', 0.8);
				} else {
					// Manter highlight do país selecionado
					d3.select(this)
						.attr('stroke', '#ffffff')
						.attr('stroke-width', 2);
				}
				// Garantir que o tooltip desapareça sempre
				tooltip = { show: false, x: 0, y: 0, content: '' };
			});

		// Renderizar fluxos se houver país selecionado
		renderFlows();

		// Atualizar highlight visual dos países
		updateCountryHighlights(g, countryCounts);
	}

	function updateCountryHighlights(g, countryCounts) {
		// Resetar todos os países para o estilo padrão
		g.selectAll('.country')
			.attr('stroke', '#2c3e50')
			.attr('stroke-width', 0.8);

		// Highlight do país selecionado, se houver
		if (selectedCountry) {
			g.selectAll('.country')
				.filter(function(d) {
					const geoName = d.properties.NAME || d.properties.name;
					for (let dataCountry of countryCounts.keys()) {
						if (isCountryMatch(geoName, dataCountry) && dataCountry === selectedCountry) {
							return true;
						}
					}
					return false;
				})
				.attr('stroke', '#ffffff')
				.attr('stroke-width', 4);
		}
	}

	function getContinentFromCoordinates([lng, lat]) {
		// Determinar continente baseado em coordenadas aproximadas mais precisas
		if (lng >= -170 && lng <= -30) {
			if (lat >= 15) return 'North America';
			if (lat >= -60) return 'South America';
		}
		if (lng >= -30 && lng <= 70) {
			if (lat >= 35) return 'Europe';
			if (lat >= -40) return 'Africa';
		}
		if (lng >= 70 && lng <= 180) {
			if (lat >= -10) return 'Asia';
			if (lat < -10) return 'Oceania';
		}
		if (lng >= 25 && lng <= 70 && lat >= 10 && lat <= 45) return 'Middle East';
		
		// Casos especiais
		if (lng >= -15 && lng <= 45 && lat >= 35 && lat <= 75) return 'Europe';
		
		return 'Asia'; // default
	}

	function renderFlows() {
		const svg = d3.select(svgElement);
		
		// Sempre limpar fluxos existentes primeiro
		svg.selectAll('.flow-line').remove();
		svg.selectAll('.flow-shadow').remove();
		svg.selectAll('.flow-main').remove();
		svg.selectAll('.flow-volume').remove();
		svg.selectAll('.flow-label').remove();
		
		// Se não há país selecionado ou dados do mundo, parar aqui
		if (!selectedCountry || !worldData) {
			return;
		}
		
		// Usar a mesma projeção do mapa principal - ajustada
		const width = 750;
		const height = 400;
		const projection = d3.geoNaturalEarth1()
			.scale(115)
			.translate([width / 2, height / 2]);

		// Filtrar fluxos do país selecionado com base nos controles
		// LÓGICA DE FILTRO:
		// - EMIGRAÇÃO: flow.origin === selectedCountry (pessoas que SAÍRAM do país selecionado)
		// - IMIGRAÇÃO: flow.destination === selectedCountry (pessoas que CHEGARAM ao país selecionado)
		let relevantFlows = migrationFlows.filter(flow => {
			const isEmigration = flow.origin === selectedCountry;      // Saiu deste país
			const isImmigration = flow.destination === selectedCountry; // Chegou neste país
			
			// Aplicar filtros de exibição
			if (isEmigration && !showEmigration) return false;
			if (isImmigration && !showImmigration) return false;
			
			return isEmigration || isImmigration;
		});





		// Criar defs para gradientes e marcadores (já limpamos os fluxos acima)
		const defs = svg.select('defs').empty() ? svg.append('defs') : svg.select('defs');
		defs.selectAll('[id^="arrowhead"], [id^="flow-gradient"]').remove();

		// Função para obter coordenadas do centro do país
		function getCountryCenter(countryName) {
			// Primeiro: buscar no GeoJSON pois tem coordenadas mais precisas
			const feature = worldData.features.find(f => {
				const geoName = f.properties.NAME || f.properties.name;
				return isCountryMatch(geoName, countryName);
			});
			
			if (feature) {
				const centroid = d3.geoCentroid(feature);
				return projection(centroid);
			}
			
			// Fallback: tentar coordenadas do dataset
			const countryData = filteredData.find(d => 
				d.country === countryName || d.countryOfCitizenship === countryName
			);
			if (countryData && countryData.latitude_country && countryData.longitude_country) {
				const coords = projection([+countryData.longitude_country, +countryData.latitude_country]);
				if (coords && !isNaN(coords[0]) && !isNaN(coords[1])) {
					return coords;
				}
			}
			
			return null;
		}

		// Renderizar linhas de fluxo melhoradas
		relevantFlows.forEach((flow, index) => {
			const originCoords = getCountryCenter(flow.origin);
			const destCoords = getCountryCenter(flow.destination);
			
			if (originCoords && destCoords) {
				const line = d3.line()
					.x(d => d[0])
					.y(d => d[1])
					.curve(d3.curveCardinal);

				// Calcular curva com offset para evitar sobreposição
				const offsetY = index % 2 === 0 ? -20 - (index * 5) : -40 - (index * 5);
				const midX = (originCoords[0] + destCoords[0]) / 2;
				const midY = (originCoords[1] + destCoords[1]) / 2 + offsetY;
				
				const pathData = line([originCoords, [midX, midY], destCoords]);

				// Determinar cor e largura baseado no fluxo e volume
				const isOutgoing = flow.origin === selectedCountry;
				const baseColor = isOutgoing ? '#ffd700' : '#ff6b6b'; // Dourado (emigração) e coral (imigração) do radar
				const strokeWidth = Math.max(2, Math.min(8, Math.sqrt(flow.count) * 2));
				
				// Criar gradiente único para cada fluxo
				const gradientId = `flow-gradient-${index}`;
				const gradient = defs.append('linearGradient')
					.attr('id', gradientId)
					.attr('x1', '0%')
					.attr('y1', '0%')
					.attr('x2', '100%')
					.attr('y2', '0%');
				
				gradient.append('stop')
					.attr('offset', '0%')
					.attr('stop-color', baseColor)
					.attr('stop-opacity', 0.7);
				
				gradient.append('stop')
					.attr('offset', '50%')
					.attr('stop-color', baseColor)
					.attr('stop-opacity', 1);
					
				gradient.append('stop')
					.attr('offset', '100%')
					.attr('stop-color', d3.color(baseColor).darker(0.5))
					.attr('stop-opacity', 0.8);

				// Criar sombra sutil
				svg.append('path')
					.attr('class', 'flow-shadow')
					.attr('d', pathData)
					.attr('fill', 'none')
					.attr('stroke', '#000000')
					.attr('stroke-width', strokeWidth + 1)
					.attr('opacity', 0.3)
					.attr('transform', 'translate(1, 1)')
					.style('pointer-events', 'none');

				// Criar linha principal
				const mainPath = svg.append('path')
					.attr('class', 'flow-main')
					.attr('d', pathData)
					.attr('fill', 'none')
					.attr('stroke', `url(#${gradientId})`)
					.attr('stroke-width', strokeWidth)
					.attr('opacity', 0.9)
					.attr('marker-end', `url(#arrowhead-${isOutgoing ? 'out' : 'in'})`)
					.style('cursor', 'pointer')
					.on('mouseover', function(event) {
						// Highlight da linha
						d3.select(this)
							.transition()
							.duration(150)
							.attr('opacity', 1)
							.attr('stroke-width', strokeWidth + 1);
						
						// Mostrar tooltip com lógica correta
						const isOutgoing = flow.origin === selectedCountry;
						let directionText, directionDetail;
						
						if (isOutgoing) {
							// Emigração: pessoas saíram do país selecionado
							directionText = 'emigração';
							directionDetail = `Saíram de ${flow.origin}`;
						} else {
							// Imigração: pessoas chegaram ao país selecionado  
							directionText = 'imigração';
							directionDetail = `Chegaram em ${flow.destination}`;
						}
						
						const percentage = ((flow.count / filteredData.length) * 100).toFixed(1);
						
						tooltip = {
							show: true,
							x: event.offsetX + 10,
							y: event.offsetY - 10,
							content: `
								<div style="text-align: center;">
									<div style="font-weight: bold; color: #ffd700; margin-bottom: 4px;">${flow.origin} → ${flow.destination}</div>
									<div style="color: ${baseColor}; font-weight: 600; margin-bottom: 2px;">${flow.count} bilionários</div>
									<div style="font-size: 10px; color: #bdc3c7;">${percentage}% do total</div>
									<div style="font-size: 9px; color: #95a5a6; margin-top: 2px;">${directionDetail} (${directionText})</div>
								</div>
							`
						};
					})
					.on('mouseout', function() {
						// Remover highlight
						d3.select(this)
							.transition()
							.duration(150)
							.attr('opacity', 0.9)
							.attr('stroke-width', strokeWidth);
						
						tooltip = { show: false, x: 0, y: 0, content: '' };
					});

				// Adicionar ponto no meio para volume (menor e mais discreto)
				const volumeRadius = Math.max(2, Math.min(6, flow.count * 0.8));
				svg.append('circle')
					.attr('class', 'flow-volume')
					.attr('cx', midX)
					.attr('cy', midY)
					.attr('r', volumeRadius)
					.attr('fill', baseColor)
					.attr('stroke', '#ffffff')
					.attr('stroke-width', 1)
					.attr('opacity', 0.9)
					.style('cursor', 'pointer')
					.on('mouseover', function(event) {
						d3.select(this)
							.transition()
							.duration(150)
							.attr('r', volumeRadius + 1)
							.attr('opacity', 1);
						
						tooltip = {
							show: true,
							x: event.offsetX + 10,
							y: event.offsetY - 10,
							content: `
								<div style="text-align: center;">
									<div style="font-weight: bold; color: #ffd700; margin-bottom: 4px;">Volume</div>
									<div style="color: ${baseColor}; font-weight: 600; margin-bottom: 2px;">${flow.count} bilionários</div>
									<div style="font-size: 10px; color: #bdc3c7;">${flow.origin} → ${flow.destination}</div>
								</div>
							`
						};
					})
					.on('mouseout', function() {
						d3.select(this)
							.transition()
							.duration(150)
							.attr('r', volumeRadius)
							.attr('opacity', 0.9);
						
						tooltip = { show: false, x: 0, y: 0, content: '' };
					});

				// Label com número (apenas para fluxos maiores) - fonte menor
				if (flow.count >= 3) {
					svg.append('text')
						.attr('class', 'flow-label')
						.attr('x', midX)
						.attr('y', midY + 2)
						.attr('text-anchor', 'middle')
						.attr('fill', '#ffffff')
						.attr('font-size', '8px') // Reduzido de 9px
						.attr('font-weight', 'bold')
						.style('text-shadow', '1px 1px 2px rgba(0,0,0,0.8)')
						.style('pointer-events', 'none')
						.text(flow.count);
				}
			}
		});

		// Criar marcadores de seta melhorados - menores
		// Seta para emigração (dourada)
		const arrowOut = defs.append('marker')
			.attr('id', 'arrowhead-out')
			.attr('viewBox', '0 -5 8 10') // Reduzido
			.attr('refX', 7) // Ajustado
			.attr('refY', 0)
			.attr('markerWidth', 5) // Reduzido de 6
			.attr('markerHeight', 5) // Reduzido de 6
			.attr('orient', 'auto');
		
		arrowOut.append('path')
			.attr('d', 'M0,-3L6,0L0,3L1,0Z') // Ajustado
			.attr('fill', '#ffd700')
			.attr('stroke', '#2c3e50')
			.attr('stroke-width', 0.5);
		
		// Seta para imigração (coral)
		const arrowIn = defs.append('marker')
			.attr('id', 'arrowhead-in')
			.attr('viewBox', '0 -5 8 10') // Reduzido
			.attr('refX', 7) // Ajustado
			.attr('refY', 0)
			.attr('markerWidth', 5) // Reduzido de 6
			.attr('markerHeight', 5) // Reduzido de 6
			.attr('orient', 'auto');
		
		arrowIn.append('path')
			.attr('d', 'M0,-3L6,0L0,3L1,0Z') // Ajustado
			.attr('fill', '#ff6b6b')
			.attr('stroke', '#2c3e50')
			.attr('stroke-width', 0.5);
	}
</script>

<div class="migration-map-container">
	<div class="map-header">
		<h3>Mapa de Migração de Bilionários</h3>
		<p>Clique nos países para visualizar fluxos migratórios</p>
		
		<!-- Controles organizados -->
		<div class="controls-container">
			<!-- Filtro por indústria -->
			<div class="filter-section">
				<label for="industry-filter">Indústria:</label>
				<select id="industry-filter" bind:value={filterIndustry}>
					<option value="all">Todas</option>
					{#each uniqueIndustries.slice(1) as industry}
						<option value={industry}>{industry}</option>
					{/each}
				</select>
			</div>
			
			<!-- Controles de fluxo -->
			<div class="flow-controls">
				<label class="control-toggle">
					<input 
						type="checkbox" 
						bind:checked={showEmigration}
						on:change={() => renderFlows()}
					/>
					<span class="toggle-switch emigration"></span>
					Emigração
				</label>
				<label class="control-toggle">
					<input 
						type="checkbox" 
						bind:checked={showImmigration}
						on:change={() => renderFlows()}
					/>
					<span class="toggle-switch immigration"></span>
					Imigração
				</label>
			</div>
		</div>
	</div>
    
	<div class="map-wrapper">
		<svg bind:this={svgElement} width="750" height="400"></svg>
		
		{#if tooltip.show}
			<div 
				class="tooltip" 
				style="left: {tooltip.x}px; top: {tooltip.y}px;"
			>
				{@html tooltip.content}
			</div>
		{/if}
	</div>

	{#if selectedCountry}
		<div class="selection-info">
			<p>País selecionado: <strong>{selectedCountry}</strong></p>
			<p><small>
				<strong>Dourado (Emigração):</strong> bilionários que <u>saíram</u> de {selectedCountry} • 
				<strong>Coral (Imigração):</strong> bilionários que <u>chegaram</u> em {selectedCountry}
			</small></p>
			<p><small>
				Base: país de cidadania → país de residência atual • Use os controles para filtrar tipos de fluxo
			</small></p>
		</div>
	{/if}
</div>

<style>
	.migration-map-container {
		background: linear-gradient(135deg, #1a1a2e 0%, #16213e 50%, #0f1419 100%);
		border-radius: 10px; /* Mais compacto */
		padding: 16px; /* Mais compacto */
		margin: 12px 0; /* Mais compacto */
		box-shadow: 0 8px 24px rgba(0, 0, 0, 0.4);
		border: 1px solid rgba(255, 215, 0, 0.2);
		max-width: 800px; /* Limitar largura máxima */
		margin: 12px auto; /* Centralizar */
	}

	.map-header {
		text-align: center;
		margin-bottom: 12px; /* Mais compacto */
	}

	.map-header h3 {
		background: linear-gradient(135deg, #ffd700 0%, #f39c12 100%);
		-webkit-background-clip: text;
		-webkit-text-fill-color: transparent;
		background-clip: text;
		font-size: 18px; /* Menor para caber melhor */
		font-weight: 700;
		margin: 0 0 6px 0; /* Mais compacto */
		text-shadow: 0 2px 4px rgba(0, 0, 0, 0.3);
	}

	.map-header p {
		color: #cbd5e1;
		font-size: 12px; /* Menor */
		margin: 0;
		opacity: 0.9;
	}



	.map-wrapper {
		position: relative;
		display: flex;
		justify-content: center;
		background: linear-gradient(135deg, #1a1a2e 0%, #2c3e50 100%);
		border-radius: 6px; /* Mais compacto */
		padding: 8px; /* Mais compacto */
		border: 1px solid rgba(255, 215, 0, 0.3); /* Borda mais sutil */
		width: 100%;
		overflow-x: auto;
		box-shadow: inset 0 2px 8px rgba(0, 0, 0, 0.2); /* Sombra mais sutil */
	}

	.tooltip {
		position: absolute;
		background: rgba(15, 23, 42, 0.95);
		color: white;
		padding: 8px 12px; /* Mais compacto */
		border-radius: 6px; /* Mais clean */
		font-size: 11px; /* Menor */
		font-weight: 500;
		pointer-events: none;
		z-index: 1000;
		border: 1px solid rgba(255, 215, 0, 0.5);
		box-shadow: 0 4px 12px rgba(0, 0, 0, 0.6);
		backdrop-filter: blur(8px);
		max-width: 200px;
		line-height: 1.3;
		transition: opacity 0.2s ease;
	}



	.selection-info {
		text-align: center;
		margin-top: 8px; /* Mais compacto */
		padding: 6px 12px; /* Mais compacto */
		background: rgba(255, 215, 0, 0.1); /* Mais sutil */
		border-radius: 4px; /* Mais compacto */
		border: 1px solid rgba(255, 215, 0, 0.25); /* Mais sutil */
	}

	.selection-info p {
		color: #e2e8f0;
		margin: 3px 0; /* Mais compacto */
		font-size: 11px; /* Menor */
	}

	.selection-info strong {
		color: #ffd700;
		font-weight: 600;
	}

	.selection-info small {
		color: #94a3b8;
		font-size: 9px; /* Menor para economizar espaço */
	}

	/* Controles organizados */
	.controls-container {
		display: flex;
		justify-content: center;
		align-items: center;
		gap: 16px; /* Menor espaçamento */
		margin-top: 8px; /* Mais compacto */
		flex-wrap: wrap;
	}

	.filter-section {
		display: flex;
		align-items: center;
		gap: 8px;
	}

	.filter-section label {
		color: #e2e8f0;
		font-size: 11px;
		font-weight: 500;
		margin: 0;
	}

	.filter-section select {
		padding: 4px 8px;
		border-radius: 6px;
		border: 1px solid rgba(255, 255, 255, 0.2);
		background: rgba(0, 0, 0, 0.3);
		color: #e0e0e0;
		font-size: 10px;
		cursor: pointer;
		min-width: 120px;
		transition: all 0.2s ease;
	}

	.filter-section select:hover {
		background: rgba(0, 0, 0, 0.4);
		border-color: rgba(255, 255, 255, 0.3);
	}

	.filter-section select:focus {
		outline: none;
		border-color: #4ecdc4;
		box-shadow: 0 0 0 2px rgba(78, 205, 196, 0.2);
	}

	/* Responsividade otimizada para 918x857 */
	@media (max-width: 920px) {
		.migration-map-container {
			max-width: 100%;
			padding: 12px;
		}
		
		.map-wrapper {
			overflow-x: auto;
		}
		
		.map-wrapper svg {
			min-width: 750px;
		}
		
		.map-header h3 {
			font-size: 16px;
		}

		.controls-container {
			flex-direction: column;
			gap: 8px;
		}
	}

	.flow-controls {
		display: flex;
		justify-content: center;
		gap: 12px; /* Reduzido de 16px */
		flex-wrap: wrap;
	}

	.control-toggle {
		display: flex;
		align-items: center;
		gap: 6px; /* Reduzido de 8px */
		color: #e2e8f0;
		font-size: 11px; /* Reduzido de 13px */
		font-weight: 500;
		cursor: pointer;
		padding: 4px 10px; /* Reduzido de 6px 12px */
		background: rgba(255, 215, 0, 0.1);
		border-radius: 12px; /* Reduzido de 16px */
		border: 1px solid rgba(255, 215, 0, 0.2);
		transition: all 0.2s ease;
	}

	.control-toggle:hover {
		background: rgba(255, 215, 0, 0.2);
		border-color: rgba(255, 215, 0, 0.4);
	}

	.control-toggle input[type="checkbox"] {
		display: none;
	}

	.toggle-switch {
		width: 28px; /* Reduzido de 32px */
		height: 14px; /* Reduzido de 16px */
		border-radius: 7px; /* Ajustado */
		border: 1px solid;
		position: relative;
		transition: all 0.2s ease;
	}

	.toggle-switch.emigration {
		border-color: #ffd700;
		background: rgba(255, 215, 0, 0.3);
	}

	.toggle-switch.immigration {
		border-color: #ff6b6b;
		background: rgba(255, 107, 107, 0.3);
	}

	.control-toggle input:checked + .toggle-switch.emigration {
		background: #ffd700;
		border-color: #ffd700;
		box-shadow: 0 0 8px rgba(255, 215, 0, 0.5);
	}

	.control-toggle input:checked + .toggle-switch.immigration {
		background: #ff6b6b;
		border-color: #ff6b6b;
		box-shadow: 0 0 8px rgba(255, 107, 107, 0.5);
	}

	.toggle-switch::before {
		content: '';
		position: absolute;
		width: 10px; /* Reduzido de 12px */
		height: 10px; /* Reduzido de 12px */
		border-radius: 50%;
		background: #ffffff;
		top: 1px;
		left: 1px;
		transition: all 0.2s ease;
		box-shadow: 0 1px 3px rgba(0, 0, 0, 0.3);
	}

	.control-toggle input:checked + .toggle-switch::before {
		transform: translateX(14px); /* Ajustado de 16px */
	}
</style> 