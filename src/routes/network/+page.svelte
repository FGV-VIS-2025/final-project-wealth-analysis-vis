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
      
      // Load the data
      const csvPath = `${base}/Billionaires Statistics Dataset.csv`;
      const data = await d3.csv(csvPath);
      
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
      
      // Create richer links based on multiple factors but be more selective
      const links = createEnhancedLinks(nodes);
      
      // Calculate communities
      const communities = calculateAdvancedCommunities(nodes, links);
      
      // Store the data in component state
      window.networkData = { nodes, links, communities };
      
      isLoading = false;
      
      // If container is ready, create the visualization
      if (container) {
        createVisualization();
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
    
    // Map industries to more general categories for better clustering
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
    
    // Assign main categories to each node
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
    });
    
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
  
  // Advanced community detection with better industry-based clustering
  function calculateAdvancedCommunities(nodes, links) {
    // First, try to group by main industry categories
    const categoryClusters = {};
    const mainCategories = [
      "Technology", "Finance", "Energy", "Retail", 
      "Real Estate", "Manufacturing", "Healthcare", 
      "Media", "Food & Beverage", "Telecom", "Other"
    ];
    
    // Initialize with category-based communities
    mainCategories.forEach((category, index) => {
      categoryClusters[category] = index;
    });
    
    // First pass: assign nodes to communities based on their primary industry
    const communities = nodes.map(node => {
      // If node has multiple categories, pick the first one as primary
      const primaryCategory = node.mainCategories[0] || "Other";
      return categoryClusters[primaryCategory];
    });
    
    // Second pass: optimize communities using network structure
    const graph = Array(nodes.length).fill().map(() => []);
    const weights = Array(nodes.length).fill().map(() => []);
    
    links.forEach(link => {
      const { source, target, weight = 1, type } = link;
      const sourceId = typeof source === 'object' ? source.id : source;
      const targetId = typeof target === 'object' ? target.id : target;
      
      // Add edge weights based on link type
      const edgeWeight = type === 'industry' ? weight * 2 : weight;
      
      graph[sourceId].push(targetId);
      weights[sourceId].push(edgeWeight);
      
      graph[targetId].push(sourceId);
      weights[targetId].push(edgeWeight);
    });
    
    // Run community optimization iterations
    let changed = true;
    let iterations = 0;
    
    while (changed && iterations < 3) { // Fewer iterations to preserve industry categories
      changed = false;
      iterations++;
      
      for (let nodeId = 0; nodeId < nodes.length; nodeId++) {
        // Count weighted connections to each community
        const communityConnections = {};
        
        for (let i = 0; i < graph[nodeId].length; i++) {
          const neighborId = graph[nodeId][i];
          const communityId = communities[neighborId];
          const weight = weights[nodeId][i];
          
          communityConnections[communityId] = (communityConnections[communityId] || 0) + weight;
        }
        
        // Find best community
        let bestCommunity = communities[nodeId];
        let maxConnection = 0;
        
        for (const [communityId, weight] of Object.entries(communityConnections)) {
          if (weight > maxConnection && communityId !== communities[nodeId].toString()) {
            maxConnection = weight;
            bestCommunity = parseInt(communityId);
          }
        }
        
        // Move to that community if it's better
        if (bestCommunity !== communities[nodeId] && maxConnection > 0) {
          communities[nodeId] = bestCommunity;
          changed = true;
        }
      }
    }
    
    // Get community to category mapping
    const communityCategories = {};
    
    // Count categories in each community
    communities.forEach((communityId, nodeId) => {
      if (!communityCategories[communityId]) {
        communityCategories[communityId] = {};
      }
      
      const node = nodes[nodeId];
      node.mainCategories.forEach(category => {
        communityCategories[communityId][category] = 
          (communityCategories[communityId][category] || 0) + 1;
      });
    });
    
    // Find dominant category for each community
    const communityLabels = {};
    Object.entries(communityCategories).forEach(([communityId, categories]) => {
      const sortedCategories = Object.entries(categories)
        .sort((a, b) => b[1] - a[1]);
      
      if (sortedCategories.length > 0) {
        communityLabels[communityId] = sortedCategories[0][0];
      } else {
        communityLabels[communityId] = "Diversificado";
      }
    });
    
    // Store community labels in global object
    window.communityLabels = communityLabels;
    
    return communities;
  }
  
  // Create the visualization
  function createVisualization() {
    if (!container || !window.networkData) return;
    
    // Get data from window storage
    const { nodes, links, communities } = window.networkData;
    
    try {
      // Clear any existing visualization
      d3.select(container).selectAll("*").remove();
      
      // Get container dimensions
      const rect = container.getBoundingClientRect();
      width = rect.width || 1000;
      height = rect.height || 600;
      
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
        .scaleExtent([0.2, 8])
        .on("zoom", (event) => {
          g.attr("transform", event.transform);
        });
      
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
      
      // Get community labels
      const communityLabels = window.communityLabels || {};
      
      // Create color scale based on industry colors
      const communityColor = d3.scaleOrdinal()
        .domain(Object.keys(communityLabels))
        .range(Object.keys(communityLabels).map(commId => {
          const category = communityLabels[commId];
          return industryColors[category] || "#a0a0a0";
        }));
                
      const linkColor = d3.scaleOrdinal()
        .domain(['industry', 'country'])
        .range(['#aaa', '#ccc']);
      
      // Prepare node data with communities
      const nodeData = nodes.map((node, i) => ({
        ...node,
        community: communities[i],
        communityLabel: communityLabels[communities[i]] || "Diversificado"
      }));
      
      // Create link elements with thinner, more subtle styling
      const link = g.append("g")
        .attr("stroke-opacity", 0.4)
        .selectAll("line")
        .data(links)
        .join("line")
        .attr("stroke", d => linkColor(d.type))
        .attr("stroke-width", d => Math.sqrt(d.weight) * 0.7);
      
      // Create node elements with cleaner styling
      const node = g.append("g")
        .selectAll("circle")
        .data(nodeData)
        .join("circle")
        .attr("r", d => 4 + Math.sqrt(d.worth/2000))
        .attr("fill", d => communityColor(d.community))
        .attr("stroke", d => {
          if (d.gender === "M") return "#4682B4";
          if (d.gender === "F") return "#DB7093";
          return "#999";
        })
        .attr("stroke-width", 1.5)
        .attr("stroke-opacity", 0.7);
      
      // Add tooltips to nodes with more industry information
      node.append("title")
        .text(d => {
          const industries = d.industries.join(", ");
          return `${d.name}\nRiqueza: $${d.worth.toLocaleString()} milhões\nCluster: ${d.communityLabel}\nIndústrias: ${industries}\nPaís: ${d.country}`;
        });
      
      // Label top 15 richest billionaires
      const topRichest = [...nodeData]
        .sort((a, b) => b.worth - a.worth)
        .slice(0, 15);
      
      // Create text labels with background for better readability
      const labelGroup = g.append("g")
        .attr("class", "billionaire-labels");
      
      // Add a background rectangle for better readability
      labelGroup.selectAll("rect")
        .data(topRichest)
        .join("rect")
        .attr("x", d => d.x + 8)
        .attr("y", d => d.y - 7)
        .attr("rx", 3)
        .attr("ry", 3)
        .attr("width", d => Math.max(d.name.length * 5.5, 50))
        .attr("height", 14)
        .attr("fill", "white")
        .attr("opacity", 0.7);
        
      // Add the actual text labels
      labelGroup.selectAll("text")
        .data(topRichest)
        .join("text")
        .attr("x", d => d.x + 10)
        .attr("y", d => d.y + 3)
        .text(d => d.name)
        .attr("font-size", 10)
        .attr("font-weight", "bold")
        .attr("pointer-events", "none")
        .attr("fill", "#333");
      
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
      
      // Add drag behavior
      node.call(d3.drag()
        .on("start", (event, d) => {
          if (!event.active) simulation.alphaTarget(0.3).restart();
          d.fx = d.x;
          d.fy = d.y;
        })
        .on("drag", (event, d) => {
          d.fx = event.x;
          d.fy = event.y;
        })
        .on("end", (event, d) => {
          if (!event.active) simulation.alphaTarget(0);
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
          
          // Same cluster
          const isSameCluster = nd.community === nodeData[selectedNode].community;
          
          return nd.id === selectedNode ? 1 : 
                 isConnected ? 0.8 : 
                 isSameCluster ? 0.5 : 0.15;
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
      
      // Create force simulation with more refined parameters
      simulation = d3.forceSimulation(nodeData)
        .force("link", d3.forceLink(links)
          .id(d => d.id)
          .distance(d => 80 / Math.max(d.weight, 0.3)))
        .force("charge", d3.forceManyBody()
          .strength(d => -50 - d.worth/10000))
        .force("center", d3.forceCenter(width/2, height/2))
        .force("x", d3.forceX(width/2).strength(0.03))
        .force("y", d3.forceY(height/2).strength(0.03))
        .force("collision", d3.forceCollide()
          .radius(d => Math.sqrt(d.worth/2000) + 8))
        // Add cluster-based forces to keep clusters more visually separated
        .force("cluster", forceCluster(communities));
      
      // Add minimal, clean legend
      addLegend(svg, communityColor, linkColor, communityLabels);
      
      // Add the richest billionaires table
      addRichestTable(svg, topRichest, communityColor);
      
      // Update positions on tick
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
        labelGroup.selectAll("text")
          .attr("x", d => d.x + 10);
        
        labelGroup.selectAll("rect")
          .attr("x", d => d.x + 8);
          
        // Update rank number positions
        g.selectAll("text.rank")
          .attr("x", d => d.x)
          .attr("y", d => d.y + 3);
      });
      
      vizInitialized = true;
      
    } catch (error) {
      console.error("Erro ao criar visualização:", error);
      hasError = true;
      errorMessage = error.message || "Ocorreu um erro ao criar a visualização.";
    }
  }
  
  // Custom force to separate clusters
  function forceCluster(communities) {
    const strength = 0.15;
    let nodes;
    
    // Calculate cluster centers
    function getCenters() {
      const centers = {};
      const centerStrength = {};
      
      // Group nodes by community
      nodes.forEach(node => {
        const community = communities[node.index];
        if (!centers[community]) {
          centers[community] = { x: 0, y: 0 };
          centerStrength[community] = 0;
        }
        
        centers[community].x += node.x;
        centers[community].y += node.y;
        centerStrength[community]++;
      });
      
      // Calculate the average position for each community
      Object.keys(centers).forEach(community => {
        centers[community].x /= centerStrength[community];
        centers[community].y /= centerStrength[community];
      });
      
      return centers;
    }
    
    function force(alpha) {
      const clusterCenters = getCenters();
      
      // Apply force toward the center of each node's community
      nodes.forEach(node => {
        const community = communities[node.index];
        const center = clusterCenters[community];
        
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
      .attr("fill", d => colorScale(d.community))
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
    
    // Add small country flags or industry labels
    rows.append("text")
      .attr("x", tableWidth - 80)
      .attr("y", 0)
      .attr("text-anchor", "end")
      .attr("font-size", "9px")
      .attr("fill", "#666")
      .text(d => {
        if (d.mainCategories && d.mainCategories.length > 0) {
          return d.mainCategories[0];
        }
        return "";
      });
    
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
        if (d.communityLabel) {
          return `Cluster: ${d.communityLabel}\nIndústrias: ${d.industries.join(", ")}`;
        }
        return `Indústrias: ${d.industries.join(", ")}`;
      });
  }
  
  // Add legend to visualization - minimalist version
  function addLegend(svg, colorScale, linkColorScale, communityLabels) {
    const legendWidth = 160;
    const legendX = width - legendWidth - 10;
    
    // Get all unique cluster labels
    const clusterLabels = Object.values(communityLabels);
    const uniqueLabels = [...new Set(clusterLabels)];
    
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
      .text("Clusters por Indústria");
    
    // Add industry clusters color legend
    uniqueLabels.forEach((label, i) => {
      const g = legend.append("g")
        .attr("transform", `translate(0, ${i * 18 + 20})`);
      
      g.append("circle")
        .attr("r", 5)
        .attr("fill", () => {
          // Find community ID for this label
          const commId = Object.keys(communityLabels).find(id => communityLabels[id] === label);
          return colorScale(commId);
        });
      
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
    </div>
  </nav>

  <section class="hero">
    <div class="hero-content">
      <h1 in:fly="{{ y: 20, duration: 800, delay: 200 }}">Rede de Bilionários por Indústria</h1>
      <p in:fly="{{ y: 20, duration: 800, delay: 400 }}">Explorando os clusters industriais e as conexões entre bilionários globais</p>
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
            <h4>Clusters (Grupos Coloridos)</h4>
            <p>Cada cor representa um <strong>cluster</strong> ou comunidade de bilionários que:</p>
            <ul>
              <li>Compartilham conexões semelhantes na rede</li>
              <li>Têm maior probabilidade de estar no mesmo ecossistema econômico</li>
              <li>São identificados algoritmicamente com base nos padrões de conexão, não apenas na indústria declarada</li>
            </ul>
            <div class="info-note">
              <strong>Nota:</strong> Um bilionário listado como "Retail" pode aparecer no cluster "Technology" se tiver mais conexões com bilionários da tecnologia.
            </div>
          </div>
          <div class="info-column">
            <h4>Conexões (Linhas)</h4>
            <p>As linhas indicam relacionamentos entre bilionários:</p>
            <ul>
              <li><strong>Conexões por indústria:</strong> Linhas mais escuras representam bilionários que compartilham setores ou categorias industriais semelhantes</li>
              <li><strong>Conexões por país:</strong> Linhas mais claras conectam bilionários do mesmo país</li>
            </ul>
            <p>A estrutura da rede revela como a riqueza global está interconectada e quais bilionários servem como "pontes" entre diferentes indústrias.</p>
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
            <button class="action-button" on:click={() => loadData()}>
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
            <li><strong>Cores:</strong> Representam clusters de bilionários em ecossistemas econômicos similares</li>
            <li><strong>Tamanho:</strong> Representa o patrimônio líquido do bilionário</li>
            <li><strong>Conexões:</strong> Bilionários conectados por indústrias compartilhadas ou localização geográfica</li>
          </ul>
          
          <div class="insight-box">
            <div class="insight-icon">💡</div>
            <div class="insight-content">
              <p>Esta visualização revela padrões inesperados: bilionários de indústrias tradicionalmente diferentes frequentemente se agrupam no mesmo cluster devido a investimentos diversificados e conexões de negócios que transcendem categorias industriais tradicionais.</p>
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