# MVP - ENGENHARIA DE DADOS
Este é um repositório para alocar o MVP de Engenharia de Dados da PUC-RIO. 

O MVP tem como tema o mercado de energia, mais especificamente o Ambiente de Contratação Livre (ACL), e busca analisar se é possível estabelecer uma relação entre o preço da energia, Preço da Liquidação das Diferenças (PLD), e os níveis de geração de usinas em território brasileiro, além de buscar evidenciar se existe alguma usina que possui maior causalidade ou correlação quando o preço da energia se encontra elevado.

Foi utilizada a plataforma DataBricks para manutenção dos Notebooks, desde à ingestão dos dados e desenvolvimento da arquitetura Medallion, até o momento da análise final.

### **1 - Contexto de Negócios e Perguntas**
Para este trabalho, foram feitos os seguintes questionamentos: **Como o intercâmbio entre submercados afeta o preço da energia, qual o tipo de usina que mais impacta o preço do PLD com as suas variações de geração e se é possível determinar uma fonte que mais impacta cada submercado.**

Em um ambiente de mercado competitivo, o preço que se paga pela energia deixa de ser apenas um insumo e se torna algo estratégico; uma empresa que é capaz de identificar isso reduz seus gastos e é capaz de dedicar recursos a outras partes de sua produção. Nesse sentido, ser capaz de identificar padrões e entender a dinâmica do preço da energia e os fatores que o rodeam se torna um diferencial para o planejamento a médio e longo prazo. 

Com esse intuito, agreguei os dados brutos disponibilizados pela **Câmara de Comercialização de Energia Elétrica (CCEE)** e o **Operador Nacional do Sistema Elétrico (ONS)** no que diz respeito ao PLD e à situação de geração de cada usina. No dataset disponibilizado pela CCEE possuímos a divisão das regiões do país de acordo com o Sistema Interligado Nacional (SIN) e o PLD calculado/auferido para cada hora de todos os meses, desde 2001; no dataset disponibilizado pela ONS, por outro lado, temos a geração de cada tipo de usina em cada submercado e se esse submercado estava sendo um exportador ou importador de energia, na granularidade horária. 

### **2 - Carga dos Dados**
Esta etapa foi realizada na camada BRONZE e o passo a passo pode ser conferido em [Estrutura Medallion - Bronze](https://github.com/Th1ago01/MVP_ENGENHARIA-DE-DADOS/blob/main/MVP%20-%20PUC%20RIO%20BRONZE%20LAYER.ipynb). 
O processo completo foi realizado consumindo recursos específicos da CCEE e da ONS para estes tipos de informação. No total, foram ingeridos três datasets nesta etapa:

**2.1 - DADOS DO BALANÇO DE ENERGIA NOS SUBSISTEMAS (2000-2026)** : dados horários de balanço de energia por subsistema da ONS, obtidos via requisição HTTP direta a arquivos Parquets, disponibilizados no Open Data da ONS (https://dados.ons.org.br/dataset/balanco-energia-subsistema);

**2.2 - Dados horários de PLD (2021–2026)**: dados horários obtidos via API pública de dados abertos da CCEE (https://dadosabertos.ccee.org.br/dataset/pld_horario). Para esta etapa, foi necessário utilizar a biblioteca "curl_cffi" do Python, pois a API da CCEE bloqueia requisições HTTP convencionais por medidas de proteção. Essa hipótese foi testada quando um mesmo request que tentei realizar funcionava no **Postman**, mas não em minha máquina local em editores de código como Visual Studio Code (VsCode) ou editores da nuvem (DataBricks);

**2.3 - Dados semanais históricos do PLD (2001-2020)**: dados do PLD no formato semanal, também ingeridos por API. Esse dataset se refere a um período em que o PLD possuía uma metodologia de cálculo diferente da praticada a partir de 2021. Durante a elaboração da camada SILVER e ponderando como seria realizada a integração na camada GOLD, foi decidido não utilizar este dataset visto que além da divergência de granularidade aos outros datasets, a divergência na memória de cálculo dos valores apurados poderia contaminar a análise final do trabalho.

Em todos os casos, os dados foram normalizados para tipo string para evitar conflitos de schema entre lotes que estavam acontecendo. 

### **3 - Modelagem e Catálogo de Dados**
A modelagem dos dados foi estruturada nas camadas seguintes à Bronze, respeitando a arquitetura Medallion, nas camadas Silver e Gold. Nesse processo, fez-se uso da funcionalidade "catalog" dentro do Databricks e criação de schemas representando cada uma das etapas e abrigando cada uma das tabelas pertinentes a cada momento.

Na camada **Silver**, os dados brutos, e que haviam sido convertidos para string, foram limpos e tipados corretamente, dando origem à três tabelas: pld_horario, balanco_energia e balanco_energia_sin. O dicionário completo desta camada, com a descrição de cada coluna, está disponível em [Dicionário de Dados - Silver](https://github.com/Th1ago01/MVP_ENGENHARIA-DE-DADOS/blob/main/Dicion%C3%A1rio_silver.pdf).

Na camada **Gold**, os dados foram reorganizados em um modelo **star schema**, composto por duas tabelas dimensão, dim_subsistema e dim_tempo, e três tabelas de fato no grão original: fact_pld_horario, fact_geracao_carga e fact_geracao_carga_sin.
Adicionalmente, foram criadas duas tabelas agregadas com algumas métricas estipuladas a fim de responder as perguntas deste projeto: agg_intercambio_preco, tabela essa que resume o papel de cada submercado, exportador ou importador, e o spread (diferença) de PLD em relação ao Sudeste; e agg_impacto_fonte, tabela essa que busca medir a associação entre a geração por tipo de fonte e as variações do PLD. O dicionário de dados completo da camada Gold está disponível em [Dicionário de Dados - Gold](https://github.com/Th1ago01/MVP_ENGENHARIA-DE-DADOS/blob/main/Dicion%C3%A1rio_gold.pdf).

Abaixo está uma imagem de como ficou estruturado o catalog "portfolio_energia" criado para abrigar este projeto no Databricks.

<img width="396" height="438" alt="image" src="https://github.com/user-attachments/assets/42b08145-0e83-420d-b59b-fd3922476ab8" />

### **4 - Pipeline de Dados**
O pipeline ETL foi estruturado em três notebooks, uma para cada uma das camadas da arquitetura Medallion. 

Essa tomada de decisão se deve ao fato de que cada camada, e consequentemente notebook, possuem uma responsabilidade distinta: ingestão da fonte (Bronze), limpeza e padronização (Silver), modelagem dimensional e agregação de métricas (Gold). Além disso, em casos de falhas no processo, essa divisão torna mais fácil encontrar a origem do problema. 

Os notebooks estão versionados nesse diretório conforme pontos abaixo:
- [CAMADA BRONZE](https://github.com/Th1ago01/MVP_ENGENHARIA-DE-DADOS/blob/main/MVP%20-%20PUC%20RIO%20BRONZE%20LAYER.ipynb) - ingestão dos dados da ONS e CCEE por meio de API.
- [CAMADA SILVER](https://github.com/Th1ago01/MVP_ENGENHARIA-DE-DADOS/blob/main/MVP%20-%20PUC%20RIO%20SILVER%20LAYER.ipynb) - transformação e tipagem dos dados de forma correta, além de verificações de qualidade após estas transformações, se certificando de que nenhum dado havia sido perdido.
- [CAMADA GOLD](https://github.com/Th1ago01/MVP_ENGENHARIA-DE-DADOS/blob/main/MVP%20-%20PUC%20RIO%20GOLD%20LAYER.ipynb) - separação em tabelas fato ("fact_"), tabelas dimensão ("dim_"), tabelas de métricas ("agg_").

Abaixo estão imagens retiradas diretamente da plataforma Databricks, com as tabelas pertencentes a cada momento do projeto, assim como suas colunas e os tipos de dados aceitos em cada uma delas: 

**BRONZE**

<img width="285" height="649" alt="image" src="https://github.com/user-attachments/assets/b34da385-3fd8-482f-961b-746a5f56cf70" />

**SILVER**

<img width="347" height="876" alt="image" src="https://github.com/user-attachments/assets/fce6f9c0-ef31-4ea1-b7c6-4800c28cca07" />

**GOLD**

<img width="299" height="766" alt="image" src="https://github.com/user-attachments/assets/87e51e6b-bc5d-4b26-b9ef-64753721e2ff" /> <img width="301" height="492" alt="image" src="https://github.com/user-attachments/assets/04782ed7-e917-4a13-a389-1054501ebc20" />

### **5 - Qualidade de Dados**
Para este projeto, devido à disponibilização dos dados em plataformas abertas e a manutenção constante por instituições robustas (CCEE e ONS), aferir a qualidade dos dados não se provou uma tarefa complexa.

Durante o projeto, foram realizadas transformações apenas para garantir o formato correto das colunas após a ingestão na camada Bronze e a checagem de que não existiam duplicatas na ingestão ou a perda de dados no processo de transformação.
Isso se deu, pois, conforme elencado anteriormente, na camada Bronze todas as tabelas possuem seus valores em tipo string para evitar o conflito de salvamento de schemas que estava acontecendo entre lotes de execução; por outro lado, foram realizadas as checagens de duplicatas para garantir que a cada ingestão de dados a camada Bronze estava sendo corretamente atualizada. E, por último, foi feita a checagem de que nenhuma mudança na tipagem das colunas pudesse ter causado a perda de algum dado.

Por mais que tenham sido constatados valores nulos da camada Bronze, especialmente na tabela **ons_balanco_energia_subsistema**, que posteriormente originou as tabelas **balanco_energia** e **balanco_energia_sin** na camada Silver, esses valores nulos significam a realidade da operação e não uma anomalia: valores de geração solar nula ou muito próximas de zero são aceitáveis, visto que tratando-se do contexto é uma fonte de energia que deixa de produzir em períodos noturnos. Abaixo, uma mostra para exemplificar este fato:

| id_subsistema | nom_subsistema | din_instante | val_gerhidraulica | val_gertermica | val_gereolica | val_gersolar | val_carga | val_intercambio |
|---|---|---|---|---|---|---|---|---|
| S | SUL | 2006-09-03 23:00:00 | 2176.15000000 | 1288.61000000 | 45.50000000 | 0E-8 | 5972.70000000 | -2462.44000000 |
| NE | NORDESTE | 2006-09-04 00:00:00 | 6198.74000000 | 153.16999999 | 22.72000000 | 0E-8 | 6364.99000000 | 9.64000000 |
| N | NORTE | 2006-09-04 00:00:00 | 2263.17000000 | 0E-8 | 0E-8 | 0E-8 | 3363.99000000 | -1100.82000000 |
| SIN | SISTEMA INTERLIGADO NACIONAL | 2006-09-04 00:00:00 | 34016.30000000 | 4338.74000000 | 64.19000000 | NULL | 38392.02000000 | 27.21000000 |
| SE | SUDESTE/CENTRO-OESTE | 2006-09-04 00:00:00 | 23832.19999999 | 2895.37000000 | 0E-8 | 0E-8 | 23169.64999999 | 3557.92000000 |

### **6 - Análise de Dados**

Para esta etapa, foram realizadas consultas em SQL no ambiente do DataBricks para tentar responder às perguntas elencadas no início deste trabalho. O notebook com essas consultas pode ser visualizado **aqui**.

- Como o intercâmbio afeta o preço?

O Sudeste/Centro-Oeste (SE) foi adotado como referência, por ser o maior centro de carga do sistema. O spread mede a diferença entre o PLD de cada submercado e o PLD do SE. O percentual de horas descoladas indica com que frequência os preços deixaram de ser iguais.

Os resultados mostram que o papel no intercâmbio está associado à posição do preço em relação ao SE. Quando o Norte exporta energia, seu PLD fica em média R$ 18,45/MWh abaixo do SE, com 19,6% das horas descoladas. Quando importa, o spread é praticamente nulo. O Nordeste segue o mesmo padrão como exportador, com spread médio de −R$ 15,24/MWh e 22,8% das horas descoladas. O Sul se comporta de forma oposta: nos períodos em que importa, seu PLD fica em média R$ 5,88/MWh acima do SE, e o PLD médio chega a R$ 176,95/MWh, o maior da tabela.

| id_subsistema | papel_intercambio | pld_medio | spread_medio_se | pct_horas_descoladas |
|:---:|:---|---:|---:|---:|
| N | EXPORTADOR | 148.55 | -18.45 | 19.63 |
| N | IMPORTADOR | 150.28 | -0.05 | 11.72 |
| NE | EXPORTADOR | 151.05 | -15.24 | 22.78 |
| NE | IMPORTADOR | 117.22 | -11.58 | 23.10 |
| S | EXPORTADOR | 142.82 | -0.11 | 8.39 |
| S | IMPORTADOR | 176.95 | 5.88 | 13.19 |

- Qual fonte mais impacta o PLD e cada submercado?

Para cada fonte, foi calculado o desvio médio do PLD nas horas em que a geração estava alta e nas horas em que estava baixa. O impacto é a diferença entre esses dois desvios e indica quanto o preço muda conforme a geração da fonte varia.

No conjunto do sistema, a fonte térmica apresentou o maior impacto médio, de R$ 16,57/MWh. Nos períodos de geração térmica alta, o PLD ficou em média R$ 7,92/MWh acima do baseline; nos de geração baixa, R$ 7,99/MWh abaixo. A hidráulica fica em segunda lugar, com impacto de R$ 14,49/MWh, depois a solar (R$ 10,66/MWh) e, por último, a eólica (R$ 5,74/MWh). A eólica foi a única fonte em que geração alta coincidiu com PLD ligeiramente abaixo da média, o que indica uma leve tendência de redução de preço quando há mais vento.

Por submercado, a térmica é a mais impactante no Norte (R$ 24,80/MWh) e no Nordeste (R$ 15,77/MWh). No Nordeste, porém, a solar aparece muito próxima (R$ 15,03/MWh). No Sul e no Sudeste, a hidráulica é a fonte de maior impacto (R$ 16,58/MWh e R$ 20,03/MWh), o que é coerente com a forte presença de grandes reservatórios nessas regiões.

Entretanto, é importante entender o impacto das térmicas de forma clara. As usinas térmicas têm custo de operação mais alto e são acionadas justamente quando o preço já está elevado, em geral em períodos de pouca chuva ou de demanda alta. Por isso, a relação observada indica que geração térmica alta e PLD alto andam juntos, mas não que a térmica seja a causa da alta do preço. O mais preciso é dizer que a geração térmica é o melhor indicador de períodos de PLD elevado.

| fonte | desvio_geracao_alta | desvio_geracao_baixa | impacto_medio |
|:---|---:|---:|---:|
| TERMICA | 7.92 | -7.99 | 16.57 |
| HIDRAULICA | 6.55 | -6.52 | 14.49 |
| SOLAR | 2.61 | -2.87 | 10.66 |
| EOLICA | -0.16 | 0.36 | 5.74 |

| id_subsistema | fonte | impacto_medio |
|:---:|:---|---:|
| N | TERMICA | 24.80 |
| N | HIDRAULICA | 9.34 |
| N | EOLICA | 6.71 |
| N | SOLAR | 5.61 |
| NE | TERMICA | 15.77 |
| NE | SOLAR | 15.03 |
| NE | HIDRAULICA | 11.99 |
| NE | EOLICA | 8.08 |
| S | HIDRAULICA | 16.58 |
| S | TERMICA | 12.08 |
| S | SOLAR | 10.76 |
| S | EOLICA | 2.78 |
| SE | HIDRAULICA | 20.03 |
| SE | TERMICA | 13.61 |
| SE | SOLAR | 8.94 |
| SE | EOLICA | 4.53 |



