# Billionaire Wealth Analysis Vis

## Integrantes
- Gabriel Abad
- Tomás Lira
- Leonardo Alexandre

## Pôster
link: https://docs.google.com/presentation/d/1ulw61zjcuxQmU7ULQI_qBHXlpE-da5uO/edit?slide=id.p1#slide=id.p1

## Imagem de Resumo
![Resumo](static/imagem_de_resumo.png)

## Resumo
O projeto foi criado para atender à necessidade de ferramentas que não só apresentem dados sobre bilionários, mas que também permitam a exploração interativa e visual dessas informações. Seu principal objetivo é disponibilizar uma plataforma online que consolide e exiba dados do universo dos bilionários, possibilitando análises sobre como eles se distribuem por gênero, idade, país e residência, além de investigar seus fluxos migratórios e a conexão entre a concentração de bilionários e indicadores macroeconômicos.

## Relatório
link: https://www.overleaf.com/read/zkkrrpctztyn#294711

## Teaser
link: https://youtu.be/JMrvhyD4el8

## Vídeo
link: https://youtu.be/1ICyg6m-28M

## Instruções
Basicamente, foi implementado um scrolltelling. Então para explorar todas as visualizações siga o caminho natural de scrolar para baixo que você passará para próxima visualização/insight.


## Divisão de Tarefas

### Gabriel
- Configurações do github pages, para podermos usar o csv.
- Deploy automático pro github pages ao dar push.
- Estrutura inicial da primeira página(com gráficos estáticos), modularizando o projeto para facilitar o desenvolvimento.
- Scrolltelling, adicionei a lógica que implementa o efeito de passar para uma próxima vis.
- Efeito de transição entre páginas.
- Botões para mudar de página.
- Ferramenta de busca de bilionários.
- Refatorei a página inicial e a página de trajetoria para melhor leitura do código. Dividindo melhor o que cada módulo faz.
- Idealizei e implementei o tema de cores do trabalho (modificando em relação a v1 do trabalho por completo).
- Imagem do Resumo.
- Relatório.
- Introdução e página inicial do poster.
- Fix de algumas sugestões dos colegas/professor e de alguns bugs.

### Tomás
- Criação do mapa do fluxo de imigração
- Reajuste dos gráficos da página principal
- Criação do gráfico de radar
- Correção de bug de animação
- Criação do storytelling

### Leonardo
- Criação do gráfico de gênero interativo
- Construção da distribuição de bilionários interativa
- Elaboração de distribuição por faixa etária interativa
- Efeito de transição em gráficos
- Customização visual do mapa de migração
- Correção de bugs 
- Elaboração do pôster
- Criação da imagem de resumo
- Correções do relatório


**Uso de IA:** 
Utilizamos a ferramenta para gerar grande parte do CSS e HTML do site, com exceção notável da técnica de scrolltelling. Também foi utilizada na refatoração do código, visando melhorar sua legibilidade e manutenibilidade. Também empregamos a IA para ajudar na documentação.

## Instruções de execução

Siga estas etapas para rodar o projeto:

1. **Instale as dependências**
   ```cmd
   npm install
   npm install -D svelte@4
   npm install -D @sveltejs/adapter-static@2
   
2. **Inicie o servidor de desenvolvimento e abra automaticamente no navegador:**
    ```cmd
    npm run dev -- --open
