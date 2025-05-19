<script>
  import * as d3 from 'd3';
  import { onMount, afterUpdate } from 'svelte';

  export let data;

  let chart;
  let width = 800;
  let height = 500;
  let margin = { top: 50, right: 180, bottom: 60, left: 200 };
  let innerWidth = width - margin.left - margin.right;
  let innerHeight = height - margin.top - margin.bottom;
  let tooltip;
  let hoveredCountry = null;

  $: if (data && data.length > 0 && chart) {
    createChart();
  }

  onMount(() => {
    if (data && data.length > 0) {
      createChart();
    }
    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
  });

  function handleResize() {
    const container = chart.parentNode;
    width = container.clientWidth || 800;
    innerWidth = width - margin.left - margin.right;
    createChart();
  }
  
  // Format billions properly
  function formatBillions(value) {
    return `$${d3.format(".1f")(value / 1000)}B`;
  }

  function createChart() {
    if (!data || !data.length) {
      console.log("No data available for DumbbellChart");
      return;
    }

    console.log("Creating chart with data:", data.length);
    d3.select(chart).selectAll("*").remove();

    const svg = d3.select(chart)
      .attr("width", width)
      .attr("height", height);

    // Create tooltip div if it doesn't exist
    if (!tooltip) {
      tooltip = d3.select("body").append("div")
        .attr("class", "tooltip")
        .style("opacity", 0)
        .style("position", "absolute")
        .style("pointer-events", "none")
        .style("background", "rgba(255, 255, 255, 0.9)")
        .style("border", "1px solid #ddd")
        .style("border-radius", "8px")
        .style("padding", "12px")
        .style("box-shadow", "0 4px 12px rgba(0,0,0,0.1)")
        .style("font-size", "14px")
        .style("max-width", "300px")
        .style("z-index", "100");
    }

    const g = svg.append("g")
      .attr("transform", `translate(${margin.left},${margin.top})`);

    try {
      // Add a background rectangle
      svg.append("rect")
        .attr("width", width)
        .attr("height", height)
        .attr("fill", "transparent");
      
      // Create working copy of data
      const workingData = [...data];
      
      // Sort data by absolute gap size (descending)
      workingData.sort((a, b) => Math.abs(b.maleAvg - b.femaleAvg) - Math.abs(a.maleAvg - a.femaleAvg));
  
      // Only show top 15 countries with largest gaps
      const displayData = workingData.slice(0, 15);
      
      // Create scales
      const y = d3.scaleBand()
        .domain(displayData.map(d => d.country))
        .range([0, innerHeight])
        .padding(0.3);
  
      // Find maximum value for x scale, considering both male and female values
      const maxValue = d3.max(displayData, d => Math.max(d.maleAvg || 0, d.femaleAvg || 0)) * 1.1;
      
      const x = d3.scaleLinear()
        .domain([0, maxValue])
        .range([0, innerWidth]);
      
      // Add subtle grid lines
      g.selectAll("line.grid")
        .data(x.ticks(5))
        .enter()
        .append("line")
        .attr("class", "grid")
        .attr("x1", d => x(d))
        .attr("x2", d => x(d))
        .attr("y1", 0)
        .attr("y2", innerHeight)
        .attr("stroke", "#e2e8f0")
        .attr("stroke-width", 1)
        .attr("stroke-dasharray", "3,3");
  
      // Add axes
      g.append("g")
        .attr("class", "x-axis")
        .attr("transform", `translate(0,${innerHeight})`)
        .call(d3.axisBottom(x)
          .ticks(5)
          .tickFormat(d => formatBillions(d)))
        .call(g => g.select(".domain").attr("stroke", "#cbd5e0"))
        .call(g => g.selectAll(".tick line").attr("stroke", "#cbd5e0"))
        .selectAll("text")
        .attr("font-size", "12px")
        .attr("fill", "#4a5568")
        .attr("font-weight", "500");

      g.append("g")
        .attr("class", "y-axis")
        .call(d3.axisLeft(y))
        .call(g => g.select(".domain").remove())
        .call(g => g.selectAll(".tick line").remove())
        .selectAll("text")
        .attr("font-size", "13px")
        .attr("fill", "#2d3748")
        .attr("font-weight", "600");
  
      // Add title
      svg.append("text")
        .attr("x", width / 2)
        .attr("y", 26)
        .attr("text-anchor", "middle")
        .attr("font-size", "18px")
        .attr("font-weight", "bold")
        .attr("fill", "#2d3748")
        .text("Disparidade de Riqueza entre Gêneros por País");
  
      // Add x-axis label
      svg.append("text")
        .attr("x", width / 2)
        .attr("y", height - 18)
        .attr("text-anchor", "middle")
        .attr("font-size", "14px")
        .attr("fill", "#4a5568")
        .text("Patrimônio Médio (Bilhões USD)");
      
      // Add background for rows
      g.selectAll(".row-background")
        .data(displayData)
        .enter()
        .append("rect")
        .attr("class", "row-background")
        .attr("x", 0)
        .attr("y", d => y(d.country))
        .attr("width", innerWidth)
        .attr("height", y.bandwidth())
        .attr("fill", (d, i) => i % 2 === 0 ? "transparent" : "#f8fafc")
        .attr("rx", 4)
        .attr("ry", 4);
  
      // Create a group for each country
      const groups = g.selectAll(".country-group")
        .data(displayData)
        .enter()
        .append("g")
        .attr("class", "country-group")
        .attr("transform", d => `translate(0,${y(d.country) + y.bandwidth() / 2})`)
        .on("mouseover", function(event, d) {
          // Highlight this row
          d3.select(this).select("line")
            .attr("stroke", "#3182ce")
            .attr("stroke-width", 3);
            
          d3.select(this).selectAll("circle")
            .attr("stroke-width", 2.5)
            .attr("r", 8);
            
          // Show tooltip with more details
          tooltip.transition()
            .duration(200)
            .style("opacity", 0.9);
            
          const maleCount = d.maleCount || 0;
          const femaleCount = d.femaleCount || 0;
          const totalCount = maleCount + femaleCount;
          const malePercentage = totalCount ? Math.round((maleCount / totalCount) * 100) : 0;
          const femalePercentage = totalCount ? Math.round((femaleCount / totalCount) * 100) : 0;
            
          tooltip.html(`
            <div style="font-weight: 600; margin-bottom: 8px; color: #2d3748; font-size: 16px;">${d.country}</div>
            <div style="display: flex; justify-content: space-between; margin-bottom: 6px;">
              <span style="color: #e53e3e; font-weight: 500;">Mulheres:</span> 
              <span>${formatBillions(d.femaleAvg)}</span>
            </div>
            <div style="display: flex; justify-content: space-between; margin-bottom: 6px;">
              <span style="color: #3182ce; font-weight: 500;">Homens:</span> 
              <span>${formatBillions(d.maleAvg)}</span>
            </div>
            <div style="display: flex; justify-content: space-between; margin-bottom: 8px; font-weight: 600;">
              <span>Diferença:</span> 
              <span>${formatBillions(Math.abs(d.maleAvg - d.femaleAvg))}</span>
            </div>
            <div style="margin-top: 8px; border-top: 1px solid #edf2f7; padding-top: 8px; font-size: 13px; color: #4a5568;">
              <div>Total de bilionários: ${totalCount}</div>
              <div>Homens: ${maleCount} (${malePercentage}%)</div>
              <div>Mulheres: ${femaleCount} (${femalePercentage}%)</div>
            </div>
          `)
          .style("left", (event.pageX + 10) + "px")
          .style("top", (event.pageY - 28) + "px");
          
          hoveredCountry = d.country;
        })
        .on("mouseout", function() {
          // Reset style
          d3.select(this).select("line")
            .attr("stroke", "#cbd5e0")
            .attr("stroke-width", 2);
            
          d3.select(this).selectAll("circle")
            .attr("stroke-width", 1.5)
            .attr("r", 6);
            
          // Hide tooltip
          tooltip.transition()
            .duration(500)
            .style("opacity", 0);
            
          hoveredCountry = null;
        });
  
      // Add lines connecting the points (only if both genders have data)
      groups.filter(d => d.maleAvg > 0 && d.femaleAvg > 0)
        .append("line")
        .attr("x1", d => x(d.femaleAvg))
        .attr("x2", d => x(d.maleAvg))
        .attr("y1", 0)
        .attr("y2", 0)
        .attr("stroke", "#cbd5e0")
        .attr("stroke-width", 2)
        .attr("stroke-linecap", "round");
  
      // Add circle for female average (only if data exists)
      groups.filter(d => d.femaleAvg > 0)
        .append("circle")
        .attr("cx", d => x(d.femaleAvg))
        .attr("cy", 0)
        .attr("r", 6)
        .attr("fill", "#fbd5d5")
        .attr("stroke", "#e53e3e")
        .attr("stroke-width", 1.5);
  
      // Add circle for male average (only if data exists)
      groups.filter(d => d.maleAvg > 0)
        .append("circle")
        .attr("cx", d => x(d.maleAvg))
        .attr("cy", 0)
        .attr("r", 6)
        .attr("fill", "#bee3f8")
        .attr("stroke", "#3182ce")
        .attr("stroke-width", 1.5);
  
      // Add gap size labels (only if both genders have data)
      groups.filter(d => d.maleAvg > 0 && d.femaleAvg > 0)
        .append("text")
        .attr("x", d => x(Math.max(d.maleAvg, d.femaleAvg)) + 10)
        .attr("y", 0)
        .attr("dy", "0.35em")
        .attr("font-size", "12px")
        .attr("fill", "#4a5568")
        .text(d => `Gap: ${formatBillions(Math.abs(d.maleAvg - d.femaleAvg))}`);
  
      // Add single gender labels (if only one gender has data)
      groups.filter(d => (d.maleAvg > 0 && d.femaleAvg === 0) || (d.femaleAvg > 0 && d.maleAvg === 0))
        .append("text")
        .attr("x", d => x(Math.max(d.maleAvg, d.femaleAvg)) + 10)
        .attr("y", 0)
        .attr("dy", "0.35em")
        .attr("font-size", "12px")
        .attr("fill", "#4a5568")
        .text(d => {
          const gender = d.maleAvg > 0 ? "Homens" : "Mulheres";
          const value = d.maleAvg > 0 ? d.maleAvg : d.femaleAvg;
          return `Somente ${gender}: ${formatBillions(value)}`;
        });
  
      // Add legend
      const legend = svg.append("g")
        .attr("transform", `translate(${width - margin.right + 25}, ${margin.top + 10})`);
      
      const legendItems = [
        { label: "Mulheres Bilionárias", color: "#e53e3e", fill: "#fbd5d5" },
        { label: "Homens Bilionários", color: "#3182ce", fill: "#bee3f8" }
      ];
      
      legendItems.forEach((item, i) => {
        const g = legend.append("g")
          .attr("transform", `translate(0, ${i * 30})`);
          
        g.append("circle")
          .attr("cx", 0)
          .attr("cy", 0)
          .attr("r", 6)
          .attr("fill", item.fill)
          .attr("stroke", item.color)
          .attr("stroke-width", 1.5);
      
        g.append("text")
          .attr("x", 15)
          .attr("y", 0)
          .attr("dy", "0.35em")
          .attr("font-size", "13px")
          .attr("fill", "#4a5568")
          .text(item.label);
      });
      
      legend.append("text")
        .attr("x", 0)
        .attr("y", -25)
        .attr("font-size", "14px")
        .attr("font-weight", "600")
        .attr("fill", "#2d3748")
        .text("Legenda");
        
      // Add interaction helper text
      legend.append("text")
        .attr("x", 0)
        .attr("y", 75)
        .attr("font-size", "12px")
        .attr("fill", "#718096")
        .text("Mouse sobre um país");
        
      legend.append("text")
        .attr("x", 0)
        .attr("y", 92)
        .attr("font-size", "12px")
        .attr("fill", "#718096")
        .text("para mais detalhes");
        
    } catch (error) {
      console.error("Error rendering chart:", error);
      
      // Show error message on chart
      svg.append("text")
        .attr("x", width / 2)
        .attr("y", height / 2)
        .attr("text-anchor", "middle")
        .attr("font-size", "14px")
        .attr("fill", "#e53e3e")
        .text("Erro ao renderizar gráfico: " + error.message);
    }
  }
</script>

<div class="chart-container">
  <svg bind:this={chart}></svg>
</div>

<style>
  .chart-container {
    width: 100%;
    height: 100%;
    position: relative;
  }

  svg {
    width: 100%;
    height: 100%;
    font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
  }

  :global(.tooltip) {
    transition: opacity 0.3s;
    font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
    line-height: 1.5;
  }

  :global(.x-axis path),
  :global(.y-axis path),
  :global(.x-axis line),
  :global(.y-axis line) {
    stroke: #cbd5e0;
  }

  :global(.x-axis text),
  :global(.y-axis text) {
    fill: #4a5568;
    font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
  }
</style> 