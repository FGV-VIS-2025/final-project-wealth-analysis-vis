<script>
  // Props recebidas pelo componente: lista de bilionários e mapa de continentes
  export let billionaires = [];
  export let continentMap = {};

  // Variáveis reativas para os filtros da busca
  let searchTerm = ''; 
  let selectedGender = ''; 
  let minNetWorthInput = '';
  let selectedRegion = ''; 

  // Prepara os dados dos bilionários para exibição e filtragem
  // Adiciona uma chave única e formata alguns campos
  $: preparedBillionaires = billionaires.map((b, index) => ({
    uniqueKey: `${b.rank || 'rankless'}-${b.personName || 'noname'}-${index}`,
    personName: b.personName || 'N/A',
    finalWorth: b.finalWorth !== undefined ? +b.finalWorth : 0,
    industries: b.industries || b.source || 'N/A',
    country: b.country || 'N/A',
    region: continentMap[b.country] || 'Outra', // Mapeia país para região, ou 'Outra' se não mapeado
    gender: b.gender || 'N/A'
  }));

  // Gera a lista de regiões disponíveis para o filtro, a partir do continentMap
  $: availableRegions = [...new Set(Object.values(continentMap).filter(r => r))].sort();
  
  // Define as opções disponíveis para o filtro de gênero
  $: availableGenders = [
    { value: '', label: 'Todos os Gêneros' },
    { value: 'M', label: 'Masculino' },
    { value: 'F', label: 'Feminino' },
  ];

  // Filtra os bilionários com base nos critérios selecionados
  // Ordena os resultados pelo patrimônio líquido em ordem decrescente
  $: filteredBillionaires = preparedBillionaires
    .filter(b => {
      const nameMatch = b.personName.toLowerCase().includes(searchTerm.toLowerCase());
      
      const genderMatch = selectedGender === '' || b.gender === selectedGender;
      
      // Converte o patrimônio mínimo para milhões (a base de dados está em milhões)
      const minWorthMillions = minNetWorthInput === '' || minNetWorthInput === null ? 0 : parseFloat(minNetWorthInput) * 1000;
      const netWorthMatch = minWorthMillions === 0 || b.finalWorth >= minWorthMillions;
      
      const regionMatch = selectedRegion === '' || b.region === selectedRegion;
      
      return nameMatch && genderMatch && netWorthMatch && regionMatch;
    })
    .sort((a, b) => b.finalWorth - a.finalWorth);

  // Função para formatar o patrimônio líquido para exibição (M para milhões, B para bilhões)
  function formatNetWorth(value) {
    if (value >= 1000) {
      return (value / 1000).toFixed(1) + ' B';
    }
    return value.toFixed(0) + ' M';
  }
</script>

<div class="billionaire-search-container visualization-card">
  <div class="card-header">
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
    margin-top: 20px;
    padding-bottom: 24px;
    background-color: #1e1e1e;
    border-radius: 8px;
    border: 1px solid #333;
    color: #e0e0e0;
  }
  
  .card-header {
      display: flex;
      align-items: center;
      padding: 16px 24px;
      border-bottom: 1px solid #333;
      background-color: #2a2a2a;
      border-top-left-radius: 8px;
      border-top-right-radius: 8px;
  }
  .card-header h3 {
      margin: 0;
      font-size: 1.25rem;
      color: #f5f5f5;
  }
  
  .filters-wrapper {
    display: flex;
    flex-wrap: wrap;
    gap: 16px;
    padding: 20px 24px;
    align-items: center;
  }

  .filter-input, .filter-select {
    padding: 10px 12px;
    background-color: #2c2c2c;
    color: #e0e0e0;
    border: 1px solid #444;
    border-radius: 6px;
    font-size: 0.95rem;
    box-sizing: border-box;
    height: 40px; 
  }
  
  .filter-input::placeholder {
      color: #777;
  }
  
  .filter-input.name-search {
    flex-grow: 1;
    min-width: 200px;
  }

  .filter-input.networth-input {
    width: 180px; 
  }

  .filter-select {
     flex-basis: 180px; 
     flex-grow: 1; 
     min-width: 150px; 
  }

  .filter-input:focus, .filter-select:focus {
    outline: none;
    border-color: #ffd700;
    box-shadow: 0 0 0 2px rgba(255, 215, 0, 0.3);
  }

  .billionaire-table-wrapper {
    max-height: 330px;
    overflow-y: auto;
    overflow-x: auto; 
    margin: 0 24px; 
    border: 1px solid #333; 
    border-radius: 8px;
  }

  .billionaire-table {
    width: 100%;
    border-collapse: collapse;
    min-width: 700px; 
  }

  .billionaire-table th, .billionaire-table td {
    padding: 12px 16px;
    text-align: left;
    border-bottom: 1px solid #333;
    font-size: 0.9rem;
    white-space: nowrap;
    color: #c7c7c7;
  }

  .billionaire-table td:first-child {
      color: #e0e0e0;
      font-weight: 500;
  }

  .billionaire-table th {
    background-color: #2a2a2a;
    font-weight: 600;
    color: #f5f5f5; 
    position: sticky;
    top: 0;
    z-index: 1; 
  }
   .billionaire-table tr:last-child td {
    border-bottom: none;
  }

  .billionaire-table tbody tr:hover td {
    background-color: #2c2c2c;
  }
  
  .billionaire-table td:nth-child(2) {
    text-align: right;
    font-weight: 500;
    color: #ffd700;
  }

  .no-results {
    text-align: center;
    padding: 30px 20px;
    color: #888; 
    font-size: 1rem;
  }
</style> 