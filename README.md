# MVP_ENGENHARIA-DE-DADOS
Este é um repositório para alocar o MVP de Engenharia de Dados da PUC-RIO. 

O MVP tem como tema o mercado de energia, mais especificamente o Ambiente de Contratação Livre (ACL), e busca analisar se é possível estabelecer uma relação entre o preço da energia, Preço da Liquidação das Diferenças (PLD), e os níveis de geração de usinas em território brasileiro, além de buscar evidenciar se existe alguma usina que possui maior causalidade ou correlação quando o preço da energia se encontra elevado.

Foi utilizada a plataforma DataBricks para manutenção dos Notebooks, desde à ingestão dos dados e desenvolvimento da arquitetura Medallion, até o momento da análise final e plotagem dos gráficos.

### **1 - Contexto de Negócios e Perguntas**
Para este trabalho, foram feitos os seguintes questionamentos: **Como o intercâmbio entre submercados afeta o preço da energia e qual o tipo de usina que mais impacta o preço do PLD com as suas variações de geração, sejam altas ou baixas?**

Em um ambiente de mercado competitivo, o preço que se paga pela energia deixa de ser apenas um insumo e se torna algo estratégico; uma empresa que é capaz de identificar isso reduz seus gastos e é capaz de dedicar recursos a outras partes de sua produção. Nesse sentido, ser capaz de identificar padrões e entender a dinâmica do preço da energia e os fatores que o rodeam se torna um diferencial para o planejamento a médio e longo prazo. 

Com esse intuito, agreguei os dados brutos disponibilizados pela **Câmara de Comercialização de Energia Elétrica (CCEE)** e o **Operador Nacional do Sistema Elétrico (ONS)** no que diz respeito ao PLD e à situação de geração de cada usina. No dataset disponibilizado pela CCEE possuímos a divisão das regiões do país de acordo com o Sistema Interligado Nacional (SIN) e o PLD calculado/auferido para cada hora de todos os meses, desde 2001; no dataset disponibilizado pela ONS, por outro lado, temos a geração de cada tipo de usina em cada submercado e se esse submercado estava sendo um exportador ou importador de energia, na granularidade horária. 

### **2 - Carga dos Dados**
Esta etapa foi realizada na camada BRONZE e o passo a passo pode ser conferido em [Estrutura Medallion - Bronze](https://github.com/Th1ago01/MVP_ENGENHARIA-DE-DADOS/blob/main/MVP%20-%20PUC%20RIO%20BRONZE%20LAYER.ipynb). 
O processo completo foi realizado consumindo recursos específicos da CCEE e da ONS para estes tipos de informação. No total, foram ingeridos três datasets nesta etapa:

**2.1 - DADOS DO BALANÇO DE ENERGIA NOS SUBSISTEMAS (2000-2026)** : dados horários de balanço de energia por subsistema da ONS, obtidos via requisição HTTP direta a arquivos Parquets, disponibilizados no Open Data da ONS (https://dados.ons.org.br/dataset/balanco-energia-subsistema);

**2.2 - Dados horários de PLD (2021–2026)**: dados horários obtidos via API pública de dados abertos da CCEE (https://dadosabertos.ccee.org.br/dataset/pld_horario). Para esta etapa, foi necessário utilizar a biblioteca "curl_cffi" do Python, pois a API da CCEE bloqueia requisições HTTP convencionais por medidas de proteção. Essa hipótese foi testada quando um mesmo request que tentei realizar funcionava no **Postman**, mas não em minha máquina local em editores de código como Visual Studio Code (VsCode) ou editores da nuvem (DataBricks);

**2.3 - Dados semanais históricos do PLD (2001-2020)**: dados do PLD no formato semanal, também ingeridos por API. Esse dataset se refere a um período em que o PLD possuía uma metodologia de cálculo diferente da praticada a partir de 2021.

Em todos os casos, os dados foram normalizados para tipo string para evitar conflitos de schema entre lotes que estavam acontecendo. Abaixo, está a forma como foi organizado o catálogo, schema e as respectivas tabelas e suas colunas:

<img width="265" height="734" alt="image" src="https://github.com/user-attachments/assets/a7a56c9f-6351-4eb1-ab13-d356d3565b1f" />

### **3 - Modelagem e Catálogo de Dados**
A modelagem dos dados foi estruturada nas camadas seguintes à Bronze, respeitando a arquitetura Medallion, nas camadas Silver e Gold. Nesse processo, fez-se uso da função "catalog" dentro do Databricks e criação de schemas representando cada uma das etapas e abrigando cada uma das tabelas pertinentes a cada momento.

Na camada **Silver**, os dados brutos, e que haviam sido convertidos para string, foram limpos e tipados corretamente, dando origem à três tabelas: pld_horario, balanco_energia e balanco_energia_sin. O dicionário completo desta camada, com a descrição de cada coluna, está disponível em [Dicionário de Dados - Silver](https://github.com/Th1ago01/MVP_ENGENHARIA-DE-DADOS/blob/main/Dicion%C3%A1rio_silver.pdf).

Na camada **Gold**, os dados foram reorganizados em um modelo **star schema**, composto por duas tabelas dimensão, dim_subsistema e dim_tempo, e três tabelas de fato no grão original: fact_pld_horario, fact_geracao_carga e fact_geracao_carga_sin.
Adicionalmente, foram criadas duas tabelas agregadas com algumas métricas estipuladas a fim de responder as perguntas deste projeto: agg_intercambio_preco, tabela essa que resume o papel de cada submercado, exportador ou importador, e o spread (diferença) de PLD em relação ao Sudeste; e agg_impacto_fonte, tabela essa que busca medir a associação entre a geração por tipo de fonte e as variações do PLD. O dicionário de dados completo da camada Gold está disponível em [Dicionário de Dados - Gold](https://github.com/Th1ago01/MVP_ENGENHARIA-DE-DADOS/blob/main/Dicion%C3%A1rio_gold.pdf).

Abaixo está uma imagem de como ficou estruturado o catalog "portfolio_energia" criado para abrigar este projeto no Databricks.

<img width="396" height="438" alt="image" src="https://github.com/user-attachments/assets/42b08145-0e83-420d-b59b-fd3922476ab8" />


### **4 - Pipeline de Dados**
O pipeline ETL foi estruturado em três notebooks, uma para cada uma das camadas da arquitetura Medallion. 

Essa tomada de decisão se deve ao fato de que cada camada, e consequentemente notebook, possuem uma responsabilidade distinta: ingestão da fonte (Bronze), limpeza e padronização (Silver), modelagem dimensional e agregação de métricas (Gold). Além disso, em casos de falhas no processo, essa divisão torna mais fácil achar a origem do problema. 

Os notebooks estão versionados nese diretório conforme pontos abaixo:
- [CAMADA BRONZE](https://github.com/Th1ago01/MVP_ENGENHARIA-DE-DADOS/blob/main/MVP%20-%20PUC%20RIO%20BRONZE%20LAYER.ipynb) - ingestão dos dados da ONS e CCEE por meio de API.
- [CAMADA SILVER](https://github.com/Th1ago01/MVP_ENGENHARIA-DE-DADOS/blob/main/MVP%20-%20PUC%20RIO%20SILVER%20LAYER.ipynb) - transformação e tipagem dos dados de forma correta, além de verificações de qualidade após estas transformações, se certificando que nenhum dado havia sido perdido.
- [CAMADA GOLD](https://github.com/Th1ago01/MVP_ENGENHARIA-DE-DADOS/blob/main/MVP%20-%20PUC%20RIO%20GOLD%20LAYER.ipynb) - separação em tabelas fato ("fact_"), tabelas dimensão ("dim_"), tabelas de métricas ("agg_"). 


