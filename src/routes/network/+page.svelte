<script>
  import { onMount, onDestroy } from 'svelte';
  import * as d3 from 'd3';
  import { base } from '$app/paths';
  import { fly, fade } from 'svelte/transition';

  // State variables
  let container;
  let isLoading = true;
  let hasError = false;
  let errorMessage = '';
  let width = 0;
  let height = 0;
  let simulation;
  let svg;
  let vizInitialized = false;
  
  $: if (container && !vizInitialized && !isLoading) {
    createVisualization();
  }

  // Process data and create network visualization
  async function loadData() {
    try {
      isLoading = true;
      hasError = false;
      errorMessage = '';
      vizInitialized = false;
      
      // Limpar simulação anterior se existir
      if (simulation) {
        simulation.stop();
        simulation = null;
      }
      
      // Limpar dados anteriores
      window.networkData = null;
      
      // Load the data
      const csvPath = `${base}/Billionaires Statistics Dataset.csv`;
      console.log("Carregando dados do CSV:", csvPath);
      
      const response = await fetch(csvPath);
      if (!response.ok) {
        throw new Error(`HTTP error! Status: ${response.status}`);
      }
      
      const text = await response.text();
      if (!text || text.trim().length === 0) {
        throw new Error("Arquivo CSV vazio ou inválido");
      }
      
      console.log(`CSV carregado com sucesso, tamanho: ${text.length} caracteres`);
      
      const data = d3.csvParse(text);
      
      if (!data || data.length === 0) {
        throw new Error("Não foi possível carregar os dados dos bilionários");
      }
      
      console.log("Dados carregados:", data.length, "registros");
      
      // Filter to valid billionaires
      const validBillionaires = data.filter(d => 
        d.personName && 
        d.industries && 
        d.gender &&
        d.finalWorth
      );
      
      console.log("Bilionários válidos:", validBillionaires.length);
      
      if (validBillionaires.length < 10) {
        throw new Error("Dados insuficientes para criar uma visualização em rede");
      }
      
      // Limit to 150 billionaires for performance and cleaner visualization
      const billionaires = validBillionaires.slice(0, 150);
      
      // Create nodes with more interesting attributes
      const nodes = billionaires.map((d, i) => ({
        id: i,
        name: d.personName,
        industries: d.industries.split(',').map(ind => ind.trim()).filter(ind => ind.length > 0),
        worth: +d.finalWorth || 0,
        gender: d.gender,
        country: d.country || 'Unknown',
        source: d.source || 'Unknown',
        rank: +d.rank || 0,
        age: +d.age || 0
      }));
      
      console.log("Nós criados:", nodes.length);
      
      // Map industries to more general categories for better visualization
      const industryCategories = {
        "Technology": ["Software", "Hardware", "Technology", "Internet", "Computer", "E-commerce", "Social Media", "Online", "Tech"],
        "Finance": ["Finance", "Investment", "Banking", "Hedge Fund", "Private Equity", "Financial Services", "Insurance", "Diversified"],
        "Energy": ["Energy", "Oil", "Gas", "Coal", "Mining", "Petroleum"],
        "Retail": ["Retail", "Fashion", "Clothing", "Consumer Goods", "Supermarket", "Discount Stores"],
        "Real Estate": ["Real Estate", "Property", "Construction", "Building"],
        "Manufacturing": ["Manufacturing", "Automotive", "Industrial", "Electronics", "Machinery"],
        "Healthcare": ["Healthcare", "Pharmaceuticals", "Medical", "Biotech", "Health"],
        "Media": ["Media", "Entertainment", "Broadcasting", "Publishing"],
        "Food & Beverage": ["Food", "Beverage", "Restaurant", "Agriculture"],
        "Telecom": ["Telecommunications", "Telecom", "Mobile"]
      };
      
      // Assign primary industry category to each node
      nodes.forEach(node => {
        const categories = new Set();
        
        // Find matching categories for each industry
        node.industries.forEach(industry => {
          for (const [category, keywords] of Object.entries(industryCategories)) {
            if (keywords.some(keyword => industry.includes(keyword))) {
              categories.add(category);
            }
          }
        });
        
        // If no categories found, use "Other"
        node.mainCategories = categories.size > 0 ? Array.from(categories) : ["Other"];
        // Set primary industry as the first in the list
        node.primaryIndustry = node.mainCategories[0];
      });
      
      // Create connections based on shared industries and countries
      const links = createEnhancedLinks(nodes);
      console.log("Links criados:", links.length);
      
      // Store the data in component state
      window.networkData = { 
        nodes, 
        links, 
        industryCategories: Object.keys(industryCategories).concat(["Other"])
      };
      
      isLoading = false;
      
      // If container is ready, create the visualization
      if (container) {
        console.log("Container pronto, criando visualização");
        createVisualization();
      } else {
        console.log("Container não está pronto");
      }
      
    } catch (error) {
      console.error("Erro ao processar dados:", error);
      hasError = true;
      errorMessage = error.message || "Ocorreu um erro ao processar os dados.";
      isLoading = false;
    }
  }
  
  // Create more interesting but cleaner connections between billionaires
  function createEnhancedLinks(nodes) {
    // Industry-based connections with strengths
    const industryLinks = [];
    const industryMap = new Map();
    const countryLinks = [];
    const countryMap = new Map();
    
    // Industry links (shared industries) - primary connections
    for (let i = 0; i < nodes.length; i++) {
      const nodeA = nodes[i];
      
      for (let j = i + 1; j < nodes.length; j++) {
        const nodeB = nodes[j];
        
        // Industry connections - now based on categorized industries
        const sharedCategories = nodeA.mainCategories.filter(cat => 
          nodeB.mainCategories.includes(cat)
        );
        
        if (sharedCategories.length > 0) {
          const key = `${i}-${j}`;
          industryLinks.push({
            source: i,
            target: j,
            type: 'industry',
            weight: sharedCategories.length * 1.5, // Give more weight to category connections
            categories: sharedCategories
          });
          industryMap.set(key, true);
        }
        
        // Country connections - only for significant countries and if not already connected
        if (nodeA.country === nodeB.country && 
            nodeA.country !== 'Unknown' && 
            !industryMap.has(`${i}-${j}`)) {
          // Only add country links for top wealth countries to reduce clutter
          const significantCountries = ['United States', 'China', 'Russia', 'India', 'Germany', 'France', 'United Kingdom'];
          if (significantCountries.includes(nodeA.country)) {
            countryLinks.push({
              source: i,
              target: j,
              type: 'country',
              weight: 0.7,
              country: nodeA.country
            });
            countryMap.set(`${i}-${j}`, true);
          }
        }
      }
    }
    
    // Combine all links, but filter to avoid too many links
    // Sort links by weight to keep the most important ones
    const sortedIndustryLinks = [...industryLinks].sort((a, b) => b.weight - a.weight);
    const sortedCountryLinks = [...countryLinks].sort((a, b) => b.weight - a.weight);
    
    // Keep only the most significant links to reduce visual clutter
    const maxIndustryLinks = Math.min(nodes.length * 2, sortedIndustryLinks.length);
    const maxCountryLinks = Math.min(nodes.length, sortedCountryLinks.length);
    
    const allLinks = [
      ...sortedIndustryLinks.slice(0, maxIndustryLinks),
      ...sortedCountryLinks.slice(0, maxCountryLinks)
    ];
    
    return allLinks;
  }
  
  // Create the visualization
  function createVisualization() {
    try {
      if (!container || !window.networkData) {
        console.log("Container ou dados não disponíveis");
        return;
      }
      
      // Get data from window storage
      const { nodes, links, industryCategories } = window.networkData;
      
      if (!nodes || !links || !industryCategories) {
        console.error("Dados incompletos");
        hasError = true;
        errorMessage = "Dados incompletos para criar a visualização.";
        return;
      }
      
      // Clear any existing visualization
      d3.select(container).selectAll("*").remove();
      
      // Get container dimensions
      const rect = container.getBoundingClientRect();
      width = rect.width || 1000;
      height = rect.height || 600;
      
      console.log("Dimensões do container:", width, height);
      
      // Create SVG with clipping
      svg = d3.select(container)
        .append("svg")
        .attr("width", width)
        .attr("height", height)
        .attr("viewBox", [0, 0, width, height])
        .attr("style", "max-width: 100%; height: auto;");
      
      // Add a clipping path and background
      svg.append("defs")
        .append("clipPath")
        .attr("id", "chart-area")
        .append("rect")
        .attr("width", width)
        .attr("height", height);
      
      // Main graph container with clip path
      const g = svg.append("g")
        .attr("clip-path", "url(#chart-area)");
      
      // Add zoom behavior
      const zoom = d3.zoom()
        .scaleExtent([0.2, 6])
        .on("zoom", (event) => {
          g.attr("transform", event.transform);
        });
      
      // Set initial zoom level to ensure everything is visible
      svg.call(zoom);
      
      // Define industry-based color palette with semantic colors
      const industryColors = {
        "Technology": "#4287f5", // Blue
        "Finance": "#42f587", // Green
        "Energy": "#f5a742", // Orange
        "Retail": "#f542e0", // Pink
        "Real Estate": "#8a42f5", // Purple
        "Manufacturing": "#7a7a7a", // Gray
        "Healthcare": "#f54242", // Red
        "Media": "#f5e642", // Yellow
        "Food & Beverage": "#42f5e6", // Cyan
        "Telecom": "#a8e65c", // Light green
        "Other": "#a0a0a0" // Gray
      };
      
      // Create color scale based on industry colors
      const communityColor = d3.scaleOrdinal()
        .domain(industryCategories)
        .range(industryCategories.map(category => industryColors[category] || "#a0a0a0"));
                
      const linkColor = d3.scaleOrdinal()
        .domain(['industry', 'country'])
        .range(['#aaa', '#ccc']);
      
      // Prepare node data with positions more evenly distributed initially
      const nodeData = nodes.map((node, i) => {
        // Create initial positions in a circle pattern for better distribution
        const angle = (i / nodes.length) * 2 * Math.PI;
        const radius = Math.min(width, height) * 0.4;
        
        return {
          ...node,
          x: width/2 + radius * Math.cos(angle),
          y: height/2 + radius * Math.sin(angle),
          community: node.primaryIndustry,
          communityLabel: node.primaryIndustry
        };
      });
      
      // Create force simulation first
      simulation = d3.forceSimulation(nodeData)
        .force("link", d3.forceLink(links)
          .id(d => d.id)
          .distance(d => 60 / Math.max(d.weight, 0.3)))
        .force("charge", d3.forceManyBody()
          .strength(d => -60 - d.worth/8000))
        .force("center", d3.forceCenter(width/2, height/2))
        .force("x", d3.forceX(width/2).strength(0.05))
        .force("y", d3.forceY(height/2).strength(0.05))
        .force("collision", d3.forceCollide()
          .radius(d => Math.sqrt(d.worth/2000) + 6))
        .force("cluster", forceCluster(nodeData.map(d => d.primaryIndustry)))
        .alphaDecay(0.02)  // Slower decay for better settling
        .velocityDecay(0.3);  // Smoother movement
      
      // Create link elements with thinner, more subtle styling
      const link = g.append("g")
        .attr("stroke-opacity", 0.4)
        .selectAll("line")
        .data(links)
        .join("line")
        .attr("stroke", d => linkColor(d.type))
        .attr("stroke-width", d => Math.sqrt(d.weight) * 0.7);
      
      // Calculate connection count for each node
      const connectionCounts = {};
      
      // Initialize counts
      nodes.forEach(node => {
        connectionCounts[node.id] = 0;
      });
      
      // Count connections from links
      links.forEach(link => {
        connectionCounts[link.source.id] = (connectionCounts[link.source.id] || 0) + 1;
        connectionCounts[link.target.id] = (connectionCounts[link.target.id] || 0) + 1;
      });
      
      // Store connection count on each node
      nodeData.forEach(node => {
        node.connectionCount = connectionCounts[node.id] || 0;
      });
      
      // Create node elements with cleaner styling
      const node = g.append("g")
        .selectAll("circle")
        .data(nodeData)
        .join("circle")
        .attr("r", d => 4 + Math.sqrt(d.connectionCount) * 1.5) // Size based on connections
        .attr("fill", d => communityColor(d.primaryIndustry))
        .attr("stroke", d => {
          if (d.gender === "M") return "#4682B4";
          if (d.gender === "F") return "#DB7093";
          return "#999";
        })
        .attr("stroke-width", 1.5)
        .attr("stroke-opacity", 0.7);
      
      // Update positions on tick - set this before other interactions
      simulation.on("tick", () => {
        link
          .attr("x1", d => d.source.x)
          .attr("y1", d => d.source.y)
          .attr("x2", d => d.target.x)
          .attr("y2", d => d.target.y);
        
        node
          .attr("cx", d => d.x)
          .attr("cy", d => d.y);
        
        // Update label positions
        if (labelGroup) {
          labelGroup.selectAll("text")
            .attr("x", d => d.x + 10);
          
          labelGroup.selectAll("rect")
            .attr("x", d => d.x + 8);
        }
          
        // Update rank number positions
        g.selectAll("text.rank")
          .attr("x", d => d.x)
          .attr("y", d => d.y + 3);
      });
      
      // Add tooltips to nodes with more industry information
      node.append("title")
        .text(d => {
          const industries = d.industries.join(", ");
          return `${d.name}\nRiqueza: $${d.worth.toLocaleString()} milhões\nConexões: ${d.connectionCount}\nIndústria: ${d.primaryIndustry}\nIndústrias: ${industries}\nPaís: ${d.country}`;
        });
      
      // Label top 15 richest billionaires
      const topRichest = [...nodeData]
        .sort((a, b) => b.worth - a.worth)
        .slice(0, 15);
      
      // Create text labels with background for better readability
      const labelGroup = g.append("g")
        .attr("class", "billionaire-labels");
      
      
      // Add rank number next to each node (inside the circle)
      g.append("g")
        .selectAll("text.rank")
        .data(topRichest)
        .join("text")
        .attr("class", "rank")
        .attr("x", d => d.x)
        .attr("y", d => d.y + 3)
        .text((d, i) => i + 1)
        .attr("font-size", 9)
        .attr("text-anchor", "middle")
        .attr("font-weight", "bold")
        .attr("pointer-events", "none")
        .attr("fill", "white");
      
      // Function to auto-fit the graph
      function autoFitGraph() {
        // Initialize at a better view for visibility
        const initialTransform = d3.zoomIdentity
          .translate(width/2, height/2)
          .scale(0.9)
          .translate(-width/2, -height/2);
        
        svg.call(zoom.transform, initialTransform);
      }
      
      // Add drag behavior
      node.call(d3.drag()
        .on("start", (event, d) => {
          if (!event.active && simulation) simulation.alphaTarget(0.3).restart();
          d.fx = d.x;
          d.fy = d.y;
        })
        .on("drag", (event, d) => {
          d.fx = event.x;
          d.fy = event.y;
        })
        .on("end", (event, d) => {
          if (!event.active && simulation) simulation.alphaTarget(0);
          d.fx = null;
          d.fy = null;
        }));
      
      // Node click to highlight - with cleaner highlight effect
      let selectedNode = null;
      
      node.on("click", (event, d) => {
        event.stopPropagation();
        
        if (selectedNode === d.id) {
          selectedNode = null;
        } else {
          selectedNode = d.id;
        }
        
        // Update node and link visibility
        node.attr("opacity", nd => {
          if (selectedNode === null) return 1;
          
          // Direct connections
          const isConnected = links.some(l => 
            (l.source.id === selectedNode && l.target.id === nd.id) || 
            (l.target.id === selectedNode && l.source.id === nd.id)
          );
          
          // Same industry
          const isSameIndustry = nd.primaryIndustry === nodeData[selectedNode].primaryIndustry;
          
          return nd.id === selectedNode ? 1 : 
                 isConnected ? 0.8 : 
                 isSameIndustry ? 0.5 : 0.15;
        });
        
        link.attr("opacity", l => {
          if (selectedNode === null) return 0.4;
          return l.source.id === selectedNode || l.target.id === selectedNode ? 0.8 : 0.05;
        })
        .attr("stroke-width", l => {
          if (selectedNode === null) return Math.sqrt(l.weight) * 0.7;
          return (l.source.id === selectedNode || l.target.id === selectedNode)
            ? Math.sqrt(l.weight)
            : Math.sqrt(l.weight) * 0.5;
        });
        
        // Also update label opacity
        labelGroup.selectAll("text, rect")
          .attr("opacity", d => {
            if (selectedNode === null) return d.text ? 0.7 : 1;
            return d.id === selectedNode ? 1 : 0.2;
          });
      });
      
      // Click on background to reset selection
      svg.on("click", () => {
        selectedNode = null;
        node.attr("opacity", 1);
        link.attr("opacity", 0.4)
          .attr("stroke-width", l => Math.sqrt(l.weight) * 0.7);
        labelGroup.selectAll("text").attr("opacity", 1);
        labelGroup.selectAll("rect").attr("opacity", 0.7);
      });
      
      // Call autoFit when ready
      autoFitGraph();
      
      // Add minimal, clean legend
      addLegend(svg, communityColor, linkColor, industryCategories);
      
      // Add the richest billionaires table
      addRichestTable(svg, topRichest, communityColor);
      
      vizInitialized = true;
      
    } catch (error) {
      console.error("Erro ao criar visualização:", error);
      hasError = true;
      errorMessage = error.message || "Ocorreu um erro ao criar a visualização.";
    }
  }
  
  // Custom force to separate by industry
  function forceCluster(industries) {
    const strength = 0.18;
    let nodes;
    
    // Calculate industry centers with better distribution
    function getCenters() {
      const centers = {};
      const centerStrength = {};
      
      // Group nodes by industry
      nodes.forEach((node, index) => {
        const industry = industries[index];
        if (!centers[industry]) {
          centers[industry] = { x: 0, y: 0 };
          centerStrength[industry] = 0;
        }
        
        centers[industry].x += node.x;
        centers[industry].y += node.y;
        centerStrength[industry]++;
      });
      
      // Calculate the average position for each industry
      Object.keys(centers).forEach(industry => {
        centers[industry].x /= centerStrength[industry];
        centers[industry].y /= centerStrength[industry];
      });
      
      // Spread centers more evenly if they are too close to each other
      const minDistance = 100; // Minimum distance between centers
      const allCentersList = Object.values(centers);
      
      for (let i = 0; i < allCentersList.length; i++) {
        for (let j = i + 1; j < allCentersList.length; j++) {
          const c1 = allCentersList[i];
          const c2 = allCentersList[j];
          
          const dx = c2.x - c1.x;
          const dy = c2.y - c1.y;
          const distance = Math.sqrt(dx * dx + dy * dy);
          
          if (distance < minDistance) {
            const factor = (minDistance - distance) / distance * 0.5;
            c1.x -= dx * factor;
            c1.y -= dy * factor;
            c2.x += dx * factor;
            c2.y += dy * factor;
          }
        }
      }
      
      return centers;
    }
    
    function force(alpha) {
      const industryCenters = getCenters();
      
      // Apply force toward the center of each node's industry
      nodes.forEach((node, index) => {
        const industry = industries[index];
        const center = industryCenters[industry];
        
        if (center) {
          node.vx += (center.x - node.x) * alpha * strength;
          node.vy += (center.y - node.y) * alpha * strength;
        }
      });
    }
    
    force.initialize = function(_) {
      nodes = _;
    }
    
    return force;
  }
  
  // Add a table showing top 15 billionaires
  function addRichestTable(svg, topBillionaires, colorScale) {
    const tableWidth = 280;
    const rowHeight = 22;
    const tableX = 20;
    const tableY = 20;
    
    // Create a semi-transparent background with border
    const tableBg = svg.append("rect")
      .attr("x", tableX - 10)
      .attr("y", tableY - 10)
      .attr("width", tableWidth + 20)
      .attr("height", (topBillionaires.length + 1) * rowHeight + 30)
      .attr("rx", 8)
      .attr("ry", 8)
      .attr("fill", "white")
      .attr("stroke", "#e3e3e3")
      .attr("stroke-width", 1)
      .attr("opacity", 0.92);
    
    // Add title
    svg.append("text")
      .attr("x", tableX)
      .attr("y", tableY + 6)
      .attr("font-weight", "bold")
      .attr("font-size", "14px")
      .attr("fill", "#333")
      .text("15 Bilionários Mais Ricos");
    
    // Add header separating line
    svg.append("line")
      .attr("x1", tableX - 10)
      .attr("y1", tableY + 16)
      .attr("x2", tableX + tableWidth + 10)
      .attr("y2", tableY + 16)
      .attr("stroke", "#e0e0e0")
      .attr("stroke-width", 1);
    
    // Create the table rows
    const rows = svg.append("g")
      .selectAll("g")
      .data(topBillionaires)
      .join("g")
      .attr("transform", (d, i) => `translate(${tableX}, ${tableY + (i + 1) * rowHeight + 20})`);
    
    // Add row background for odd rows to improve readability
    rows.filter((d, i) => i % 2 === 0)
      .append("rect")
      .attr("x", -10)
      .attr("y", -rowHeight/2)
      .attr("width", tableWidth + 20)
      .attr("height", rowHeight)
      .attr("fill", "#f9f9f9")
      .attr("rx", 3)
      .attr("ry", 3);
    
    // Add rank numbers
    rows.append("text")
      .attr("x", 0)
      .attr("y", 0)
      .attr("font-size", "11px")
      .attr("font-weight", "bold")
      .attr("fill", "#555")
      .text((d, i) => `${i + 1}.`);
    
    // Add industry color indicators
    rows.append("circle")
      .attr("cx", 15)
      .attr("cy", -0)
      .attr("r", 5)
      .attr("fill", d => colorScale(d.primaryIndustry))
      .attr("stroke", "white")
      .attr("stroke-width", 1);
    
    // Add names
    rows.append("text")
      .attr("x", 30)
      .attr("y", 0)
      .attr("font-size", "11px")
      .attr("fill", "#333")
      .attr("font-weight", d => d.worth > 150000 ? "bold" : "normal")
      .text(d => d.name);
    
    // Add worth in billions with better formatting
    rows.append("text")
      .attr("x", tableWidth - 10)
      .attr("y", 0)
      .attr("text-anchor", "end")
      .attr("font-size", "11px")
      .attr("font-weight", "bold")
      .attr("fill", d => {
        // Color-code values based on wealth
        if (d.worth > 200000) return "#1a5fb4";  // >$200B
        if (d.worth > 100000) return "#26a269";  // >$100B
        return "#333";
      })
      .text(d => {
        const billions = d.worth / 1000;
        return `$${billions.toLocaleString('en-US', {minimumFractionDigits: 1, maximumFractionDigits: 1})}B`;
      });
    
    // Add small industry labels
    rows.append("text")
      .attr("x", tableWidth - 80)
      .attr("y", 0)
      .attr("text-anchor", "end")
      .attr("font-size", "9px")
      .attr("fill", "#666")
      .text(d => d.primaryIndustry || "");
    
    // Add toggle button to hide/show the table
    const toggleBtn = svg.append("g")
      .attr("transform", `translate(${tableX + tableWidth}, ${tableY - 5})`)
      .style("cursor", "pointer");
    
    toggleBtn.append("circle")
      .attr("r", 8)
      .attr("fill", "#f0f0f0")
      .attr("stroke", "#ccc")
      .attr("stroke-width", 1);
    
    toggleBtn.append("text")
      .attr("x", 0)
      .attr("y", 3)
      .attr("text-anchor", "middle")
      .attr("font-size", "12px")
      .attr("font-weight", "bold")
      .attr("fill", "#666")
      .text("×");
    
    // Add interaction to toggle
    let isVisible = true;
    toggleBtn.on("click", () => {
      isVisible = !isVisible;
      
      // Update visibility
      tableBg.transition().duration(300).attr("opacity", isVisible ? 0.92 : 0);
      rows.transition().duration(300).attr("opacity", isVisible ? 1 : 0);
      svg.selectAll("text.table-text").transition().duration(300).attr("opacity", isVisible ? 1 : 0);
      svg.selectAll("line.table-line").transition().duration(300).attr("opacity", isVisible ? 1 : 0);
      
      // Update toggle button text
      toggleBtn.select("text").text(isVisible ? "×" : "+");
    });
    
    // Add tooltips to the circles showing industry details
    rows.select("circle").append("title")
      .text(d => {
        return `Indústria: ${d.primaryIndustry}\nIndústrias: ${d.industries.join(", ")}`;
      });
  }
  
  // Add legend to visualization - minimalist version
  function addLegend(svg, colorScale, linkColorScale, industryCategories) {
    const legendWidth = 160;
    const legendX = width - legendWidth - 10;
    
    // Get all unique industry categories
    const uniqueLabels = industryCategories;
    
    // Calculate required height for all labels
    const legendHeight = 70 + (uniqueLabels.length * 18);
    
    // Create a subtle legend background
    const legendBackground = svg.append("rect")
      .attr("x", legendX - 10)
      .attr("y", 10)
      .attr("width", legendWidth)
      .attr("height", legendHeight)
      .attr("rx", 5)
      .attr("ry", 5)
      .attr("fill", "white")
      .attr("opacity", 0.7);
    
    // Create legend group
    const legend = svg.append("g")
      .attr("class", "legend")
      .attr("transform", `translate(${legendX}, 20)`);
    
    // Add title
    legend.append("text")
      .attr("font-weight", "bold")
      .attr("y", 0)
      .attr("font-size", "0.8em")
      .text("Indústrias");
    
    // Add industry color legend
    uniqueLabels.forEach((label, i) => {
      const g = legend.append("g")
        .attr("transform", `translate(0, ${i * 18 + 20})`);
      
      g.append("circle")
        .attr("r", 5)
        .attr("fill", colorScale(label));
      
      g.append("text")
        .attr("x", 12)
        .attr("y", 3)
        .attr("font-size", "0.7em")
        .text(label);
    });
    
    // Add connection types - simplified
    const connectionLegend = svg.append("g")
      .attr("transform", `translate(${legendX}, ${uniqueLabels.length * 18 + 40})`);
    
    connectionLegend.append("text")
      .attr("font-weight", "bold")
      .attr("y", 0)
      .attr("font-size", "0.8em")
      .text("Conexões");
    
    const connectionTypes = [
      { type: 'industry', label: 'Mesma indústria' },
      { type: 'country', label: 'Mesmo país' }
    ];
    
    connectionTypes.forEach((conn, i) => {
      const g = connectionLegend.append("g")
        .attr("transform", `translate(0, ${i * 18 + 20})`);
      
      g.append("line")
        .attr("x1", 0)
        .attr("y1", 0)
        .attr("x2", 10)
        .attr("y2", 0)
        .attr("stroke", linkColorScale(conn.type))
        .attr("stroke-width", 1.5);
      
      g.append("text")
        .attr("x", 15)
        .attr("y", 3)
        .attr("font-size", "0.7em")
        .text(conn.label);
    });
  }
  
  // Handle window resize
  function handleResize() {
    if (container && vizInitialized) {
      createVisualization();
    }
  }
  
  // Cleanup resources
  function cleanup() {
    if (simulation) {
      simulation.stop();
      simulation = null;
    }
  }
  
  // Initialize
  onMount(() => {
    loadData();
    
    return cleanup;
  });
  
  onDestroy(cleanup);
</script>

<svelte:head>
  <title>Rede de Indústrias de Bilionários | Análise Global</title>
  <meta name="description" content="Visualização em rede dos bilionários e suas conexões por indústrias compartilhadas" />
</svelte:head>

<svelte:window on:resize={handleResize} />

<div class="page-container">
  <nav class="main-nav">
    <h1 class="site-title">Análise Global de Bilionários</h1>
    <div class="nav-links">
      <a href="{base}/" class="nav-link">Visão Geral</a>
      <a href="{base}/gender-gap" class="nav-link">Gap de Gênero</a>
      <a href="{base}/map" class="nav-link">Mapa de Migração</a>
      <a href="{base}/network" class="nav-link active">Rede de Indústrias</a>
      <a href="{base}/ascensao" class="nav-link">Trajetórias da Fortuna</a>
    </div>
  </nav>

  <section class="hero">
    <div class="hero-content">
      <h1 in:fly="{{ y: 20, duration: 800, delay: 200 }}">Rede de Bilionários por Indústria</h1>
      <p in:fly="{{ y: 20, duration: 800, delay: 400 }}">Explorando as indústrias e as conexões entre bilionários globais</p>
    </div>
  </section>

  <section class="content-section" in:fade>
    <div class="info-card">
      <div class="card-header">
        <div class="section-icon small">🔍</div>
        <h3>Como Interpretar a Visualização</h3>
      </div>
      <div class="card-content">
        <div class="info-columns">
          <div class="info-column">
            <h4>Cores (Grupos por Indústria)</h4>
            <p>Cada cor representa uma <strong>indústria</strong> ou setor em que os bilionários atuam:</p>
            <ul>
              <li>As cores são atribuídas com base no setor principal de cada bilionário</li>
              <li>Bilionários da mesma indústria tendem a estar mais próximos na visualização</li>
              <li>Alguns bilionários atuam em múltiplos setores, mas são coloridos pelo principal</li>
            </ul>
            <div class="info-note">
              <strong>Nota:</strong> As cores são baseadas na indústria principal identificada para cada bilionário, considerando as diversas categorias em que atua.
            </div>
          </div>
          <div class="info-column">
            <h4>Conexões (Linhas)</h4>
            <p>As linhas indicam relacionamentos entre bilionários:</p>
            <ul>
              <li><strong>Conexões por indústria:</strong> Linhas mais escuras representam bilionários que compartilham setores ou categorias industriais semelhantes</li>
              <li><strong>Conexões por país:</strong> Linhas mais claras conectam bilionários do mesmo país</li>
            </ul>
            <p>A estrutura da rede revela como a riqueza global está interconectada entre países e indústrias.</p>
          </div>
        </div>
      </div>
    </div>
    
    <div class="visualization-card">
      <div class="card-header">
        <div class="section-icon small">🌐</div>
        <h3>Rede de Conexões entre Bilionários</h3>
      </div>
      
      <div class="network-instructions">
        <p><strong>Instruções:</strong> Clique em um nó para destacar suas conexões. Use a roda do mouse para zoom, clique e arraste para mover a visualização.</p>
      </div>
      
      <div class="visualization-container">
        {#if isLoading}
          <div class="loading-container">
            <div class="loading-spinner"></div>
            <p>Carregando visualização de rede...</p>
          </div>
        {:else if hasError}
          <div class="error-container">
            <div class="error-icon">⚠️</div>
            <p>Erro ao carregar os dados: {errorMessage}</p>
            <p>Tente recarregar a página ou verifique se o arquivo de dados está disponível.</p>
            <button class="action-button" on:click={() => {
              // Limpar o estado e forçar recarregamento
              if (simulation) {
                simulation.stop();
                simulation = null;
              }
              window.networkData = null;
              d3.select(container).selectAll("*").remove();
              loadData();
            }}>
              <span class="button-icon">🔄</span> Tentar novamente
            </button>
          </div>
        {/if}
        <div bind:this={container} class="chart-container"></div>
      </div>
      
      <div class="card-content">
        <div class="text-area">
          <p>
            Esta visualização de rede mostra as conexões entre os 200 bilionários mais ricos do mundo com base em fatores compartilhados.
            Cada círculo representa um bilionário, e as conexões mostram relações por:
          </p>
          <ul>
            <li><strong>Cores:</strong> Representam diferentes indústrias em que os bilionários atuam</li>
            <li><strong>Tamanho:</strong> Representa o número de conexões que o bilionário possui</li>
            <li><strong>Conexões:</strong> Bilionários conectados por indústrias compartilhadas ou localização geográfica</li>
          </ul>
          
          <div class="insight-box">
            <div class="insight-icon">💡</div>
            <div class="insight-content">
              <p>Esta visualização revela padrões interessantes: muitos bilionários atuam em múltiplos setores e indústrias, mantendo conexões com outros setores mesmo quando sua principal área de negócios é diferente.</p>
            </div>
          </div>
        </div>
      </div>
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
    padding: 50px 20px;
    background: linear-gradient(135deg, #ebf8ff 0%, #bee3f8 100%);
    border-radius: 0 0 24px 24px;
    margin-bottom: 30px;
    box-shadow: 0 4px 12px rgba(0,0,0,0.05);
  }
  
  .hero-content {
    max-width: 800px;
  }
  
  .hero h1 {
    font-size: 2.8rem;
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
    gap: 30px;
    margin-bottom: 40px;
  }
  
  .info-card, .visualization-card {
    background: white;
    border-radius: 16px;
    overflow: hidden;
    box-shadow: 0 4px 15px rgba(0,0,0,0.06);
    border: 1px solid rgba(0,0,0,0.06);
    transition: box-shadow 0.3s ease;
  }
  
  .info-columns {
    display: flex;
    gap: 30px;
  }
  
  .info-column {
    flex: 1;
  }
  
  .info-column h4 {
    font-size: 1.1rem;
    color: #2d3748;
    margin-top: 0;
    margin-bottom: 12px;
  }
  
  .info-column p {
    margin-top: 0;
    font-size: 0.95rem;
    color: #4a5568;
  }
  
  .info-column ul {
    padding-left: 20px;
    margin-top: 8px;
  }
  
  .info-column li {
    margin-bottom: 6px;
    font-size: 0.95rem;
  }
  
  .info-note {
    background: #f7fafc;
    padding: 10px 15px;
    border-left: 3px solid #4299e1;
    border-radius: 3px;
    margin-top: 15px;
    font-size: 0.9rem;
  }
  
  .card-header {
    display: flex;
    align-items: center;
    padding: 20px 30px;
    border-bottom: 1px solid rgba(0,0,0,0.05);
    gap: 16px;
  }
  
  .section-icon.small {
    width: 36px;
    height: 36px;
    font-size: 1.3rem;
    background: #ebf8ff;
    color: #2b6cb0;
    display: flex;
    align-items: center;
    justify-content: center;
    border-radius: 10px;
    box-shadow: 0 2px 5px rgba(66, 153, 225, 0.15);
  }
  
  .card-header h3 {
    font-size: 1.4em;
    color: #2d3748;
    margin: 0;
    font-weight: 600;
  }
  
  .network-instructions {
    padding: 14px 30px;
    background: #f8fafc;
    border-bottom: 1px solid rgba(0,0,0,0.05);
  }
  
  .network-instructions p {
    margin: 0;
    font-size: 0.9rem;
    color: #4a5568;
  }
  
  .visualization-container {
    width: 100%;
    height: 70vh;
    min-height: 500px;
    max-height: 700px;
    border-bottom: 1px solid rgba(0,0,0,0.05);
    position: relative;
  }
  
  .chart-container {
    width: 100%;
    height: 100%;
    background: #f8fafc;
  }
  
  .loading-container, .error-container {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    background: rgba(255, 255, 255, 0.9);
    z-index: 100;
  }
  
  .loading-spinner {
    width: 40px;
    height: 40px;
    border: 3px solid rgba(66, 153, 225, 0.2);
    border-radius: 50%;
    border-top-color: #4299e1;
    animation: spin 1s linear infinite;
    margin-bottom: 20px;
  }
  
  @keyframes spin {
    to { transform: rotate(360deg); }
  }
  
  .error-icon {
    font-size: 2.5rem;
    margin-bottom: 16px;
    color: #e53e3e;
  }
  
  .error-container p {
    margin-bottom: 20px;
    color: #e53e3e;
    font-size: 1rem;
    text-align: center;
    max-width: 80%;
  }
  
  .action-button {
    display: inline-flex;
    align-items: center;
    background: #4299e1;
    color: white;
    font-weight: 600;
    padding: 8px 16px;
    border-radius: 6px;
    border: none;
    cursor: pointer;
    margin-top: 16px;
    transition: all 0.2s ease;
    box-shadow: 0 2px 5px rgba(66, 153, 225, 0.3);
    font-size: 0.9rem;
  }
  
  .action-button:hover {
    background: #3182ce;
    transform: translateY(-1px);
    box-shadow: 0 3px 8px rgba(66, 153, 225, 0.4);
  }
  
  .button-icon {
    margin-right: 8px;
  }
  
  .card-content {
    padding: 25px 30px;
    display: flex;
    flex-direction: column;
  }
  
  .text-area {
    max-width: 100%;
  }
  
  .text-area p, .text-area li {
    font-size: 1rem;
    color: #4a5568;
    line-height: 1.6;
  }
  
  .text-area ul {
    padding-left: 20px;
    margin-bottom: 20px;
  }
  
  .text-area li {
    margin-bottom: 8px;
  }
  
  .insight-box {
    margin-top: 20px;
    padding: 16px 20px;
    background: linear-gradient(135deg, #f8f9fa 0%, rgba(235, 248, 255, 0.5) 100%);
    border-radius: 10px;
    display: flex;
    gap: 15px;
    border: 1px solid rgba(66, 153, 225, 0.15);
    box-shadow: 0 2px 6px rgba(0,0,0,0.03);
  }
  
  .insight-icon {
    font-size: 1.5rem;
    margin-top: -2px;
  }
  
  .insight-content p {
    margin: 0;
    font-size: 0.9rem;
    color: #4a5568;
  }
  
  .site-footer {
    margin-top: 50px;
    text-align: center;
    padding: 20px 0;
    border-top: 1px solid rgba(0,0,0,0.08);
  }
  
  .footer-content p {
    margin: 0;
    font-size: 0.85rem;
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
    
    .visualization-container {
      height: 50vh;
    }
    
    .insight-box {
      flex-direction: column;
      gap: 10px;
      padding: 16px;
    }
    
    .info-columns {
      flex-direction: column;
      gap: 20px;
    }
  }
  
  @media (max-width: 600px) {
    .page-container {
      padding: 0 12px 20px;
    }
    
    .hero h1 {
      font-size: 1.8rem;
    }
    
    .hero {
      padding: 30px 16px;
    }
    
    .nav-links {
      flex-wrap: wrap;
      justify-content: center;
    }
    
    .card-header h3 {
      font-size: 1.3em;
    }
    
    .section-icon.small {
      width: 32px;
      height: 32px;
      font-size: 1.2rem;
    }
    
    .text-area p, .text-area li {
      font-size: 0.95rem;
    }
  }
</style> 