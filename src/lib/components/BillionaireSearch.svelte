<script>
  export let billionaires = [];
  export let continentMap = {}; // For mapping country to region
  let searchTerm = '';
  let selectedGender = ''; // '' for All, 'M' for Male, 'F' for Female
  let minNetWorthInput = ''; // Input as string, will be parsed to number in Billions
  let selectedRegion = ''; // '' for All regions

  // Prepare data with region and ensure necessary fields exist
  $: preparedBillionaires = billionaires.map((b, index) => ({
    uniqueKey: `${b.rank || 'rankless'}-${b.personName || 'noname'}-${index}`, // Ensure a unique key
    personName: b.personName || 'N/A',
    finalWorth: b.finalWorth !== undefined ? +b.finalWorth : 0,
    industries: b.industries || b.source || 'N/A', // 'source' as fallback for industry
    country: b.country || 'N/A',
    region: continentMap[b.country] || 'Outra', // Default to 'Outra' if country not in map
    gender: b.gender || 'N/A'
  }));

  $: availableRegions = [...new Set(Object.values(continentMap).filter(r => r))].sort();
  $: availableGenders = [
    { value: '', label: 'Todos os Gêneros' },
    { value: 'M', label: 'Masculino' },
    { value: 'F', label: 'Feminino' },
    // Add other gender values if they exist and are relevant, e.g., 'N/A' for Unknown
    // For now, assuming M and F are primary, and N/A is caught by 'Todos' if not specified
  ];

  $: filteredBillionaires = preparedBillionaires
    .filter(b => {
      const nameMatch = b.personName.toLowerCase().includes(searchTerm.toLowerCase());
      
      const genderMatch = selectedGender === '' || b.gender === selectedGender;
      
      const minWorthMillions = minNetWorthInput === '' || minNetWorthInput === null ? 0 : parseFloat(minNetWorthInput) * 1000;
      const netWorthMatch = minWorthMillions === 0 || b.finalWorth >= minWorthMillions;
      
      const regionMatch = selectedRegion === '' || b.region === selectedRegion;
      
      return nameMatch && genderMatch && netWorthMatch && regionMatch;
    })
    .sort((a, b) => b.finalWorth - a.finalWorth); // Sort by net worth descending

  function formatNetWorth(value) {
    // Assuming finalWorth is in millions. 1000 million = 1 billion.
    if (value >= 1000) {
      return (value / 1000).toFixed(1) + ' B';
    }
    return value.toFixed(0) + ' M';
  }
</script>

<div class="billionaire-search-container visualization-card">
  <div class="card-header">
    <div class="section-icon small">🔍</div>
    <h3>Buscar Bilionários</h3>
  </div>
  <div class="filters-wrapper">
    <input type="text" bind:value={searchTerm} placeholder="Buscar por nome..." class="filter-input name-search"/>
    <select bind:value={selectedGender} class="filter-select">
      {#each availableGenders as genderOption (genderOption.value)}
        <option value={genderOption.value}>{genderOption.label}</option>
      {/each}
    </select>
    <input type="number" bind:value={minNetWorthInput} placeholder="Patrimônio Mín. (B)" class="filter-input networth-input" min="0" step="0.1"/>
    <select bind:value={selectedRegion} class="filter-select">
      <option value="">Todas as Regiões</option>
      {#each availableRegions as region (region)}
        <option value={region}>{region}</option>
      {/each}
    </select>
  </div>

  <div class="billionaire-table-wrapper">
    {#if filteredBillionaires.length > 0}
      <table class="billionaire-table">
        <thead>
          <tr>
            <th>Nome</th>
            <th>Patrimônio Líquido</th>
            <th>Indústria Principal</th>
            <th>Região</th>
            <th>País</th>
            <th>Gênero</th>
          </tr>
        </thead>
        <tbody>
          {#each filteredBillionaires as b (b.uniqueKey)}
            <tr>
              <td>{b.personName}</td>
              <td>{formatNetWorth(b.finalWorth)}</td>
              <td>{b.industries}</td>
              <td>{b.region}</td>
              <td>{b.country}</td>
              <td>{b.gender}</td>
            </tr>
          {/each}
        </tbody>
      </table>
    {:else}
      <p class="no-results">Nenhum bilionário encontrado com os critérios atuais.</p>
    {/if}
  </div>
</div>

<style>
  .billionaire-search-container {
    /* Uses .visualization-card from global.css for base styling */
    margin-top: 40px;
    padding-bottom: 24px; /* Add some padding at the bottom */
  }

  /* .card-header and .section-icon.small styles are expected from global.css */

  .filters-wrapper {
    display: flex;
    flex-wrap: wrap; /* Allow filters to wrap on smaller screens */
    gap: 16px;
    padding: 20px 24px;
    align-items: center;
  }

  .filter-input, .filter-select {
    padding: 10px 12px;
    border: 1px solid #cbd5e0;
    border-radius: 8px;
    font-size: 0.95rem;
    box-sizing: border-box;
    height: 40px; /* Consistent height */
  }
  
  .filter-input.name-search {
    flex-grow: 1; /* Allow name search to take more space */
    min-width: 200px;
  }

  .filter-input.networth-input {
    width: 180px; /* Fixed width for net worth */
  }

  .filter-select {
     flex-basis: 180px; /* Base width for select dropdowns */
     flex-grow: 1; /* Allow them to grow if space available */
     min-width: 150px; /* Minimum width before wrapping */
  }

  .filter-input:focus, .filter-select:focus {
    outline: none;
    border-color: #4299e1;
    box-shadow: 0 0 0 2px rgba(66, 153, 225, 0.2);
  }

  .billionaire-table-wrapper {
    max-height: 500px;
    overflow-y: auto;
    overflow-x: auto; /* Allow horizontal scroll for wide content */
    margin: 0 24px; /* Horizontal padding */
    border: 1px solid #e2e8f0; /* Light border for the table area */
    border-radius: 8px;
  }

  .billionaire-table {
    width: 100%;
    border-collapse: collapse;
    min-width: 700px; /* Increased min-width to accommodate columns */
  }

  .billionaire-table th, .billionaire-table td {
    padding: 12px 16px;
    text-align: left;
    border-bottom: 1px solid #e2e8f0;
    font-size: 0.9rem;
    white-space: nowrap; /* Prevent text wrapping */
  }

  .billionaire-table th {
    background-color: #f8fafc; /* Light background for header */
    font-weight: 600;
    color: #4a5568; /* Darker text for header */
    position: sticky;
    top: 0;
    z-index: 1; /* Keep header above scrolling content */
  }
  
  /* Add borders to first and last cells for a contained look if table is wider than wrapper */
  .billionaire-table th:first-child, .billionaire-table td:first-child {
     /* border-left: 1px solid #e2e8f0; */ /* Optional: if border on wrapper is not enough */
  }
  .billionaire-table th:last-child, .billionaire-table td:last-child {
     /* border-right: 1px solid #e2e8f0; */ /* Optional */
  }
   .billionaire-table tr:last-child td {
    border-bottom: none; /* Remove bottom border for the last row in the table */
  }

  .billionaire-table tbody tr:hover {
    background-color: #f1f5f9; /* Hover effect */
  }
  
  .billionaire-table td:nth-child(2) { /* Net worth column */
    text-align: right;
    font-weight: 500;
  }

  .no-results {
    text-align: center;
    padding: 20px;
    color: #718096; /* Muted color for no results message */
  }
</style> 