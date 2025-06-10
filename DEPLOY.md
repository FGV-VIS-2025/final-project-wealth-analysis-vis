# Deploy no GitHub Pages

## Problemas Identificados e Soluções

### 1. Configuração do Base Path
- ✅ **Corrigido**: `svelte.config.js` está usando `BASE_PATH` corretamente
- ✅ **Corrigido**: GeoJSON agora usa o base path correto

### 2. Verificar GitHub Pages Settings

No seu repositório GitHub, vá em:
**Settings → Pages → Source**

Certifique-se de que está configurado como:
- Source: **GitHub Actions**

### 3. Verificar Build Logs

Se ainda não funcionar, verifique os logs do GitHub Actions:
1. Vá na aba **Actions** do seu repositório
2. Clique no último workflow de deploy
3. Verifique se há erros no build ou deploy

### 4. Caminhos de Arquivos

Os arquivos estáticos devem estar em:
- `static/countries.geojson` ✅
- `static/Billionaires Statistics Dataset.csv` ✅

### 5. Comandos para Testar Localmente

```bash
# Instalar dependências
npm install

# Build para produção (simula GitHub Pages)
BASE_PATH=/final-project-wealth-analysis-vis npm run build

# Preview do build
npm run preview
```

### 6. URL Esperada

Seu site deve estar disponível em:
`https://[seu-usuario].github.io/final-project-wealth-analysis-vis/`

### 7. Troubleshooting Comum

**Erro 404 nos arquivos:**
- Verifique se os arquivos estão na pasta `static/`
- Confirme que o base path está correto

**JavaScript não carrega:**
- Verifique console do navegador (F12)
- Confirme que o workflow de deploy executou com sucesso

**Dados não aparecem:**
- Abra Network tab (F12) e veja se os arquivos CSV/GeoJSON estão sendo carregados
- Verifique se o CORS está permitido (GitHub Pages permite por padrão) 