# RiscoRelativo
Projeto criando um Score automatico que identifica clientes com potencial risco de inadimplencia atraves de modelos estatisticos aplicados em base de clientes para identificar padrões de inadimplencia

# Credit Risk Score & Default Analysis

##  Objetivo

Este projeto tem como objetivo identificar o **perfil de clientes com risco de inadimplência** por meio de análise de dados, construir uma **pontuação de crédito** e calcular o **risco relativo**. A intenção é classificar os clientes (e potenciais clientes) em **categorias de risco**, com base na probabilidade de inadimplência.

Essa classificação permite ao banco tomar decisões mais informadas sobre a concessão de crédito, **reduzindo o risco de inadimplência** e fortalecendo a **eficiência operacional** e a **solidez financeira**.

---

##  Perguntas-chave

- Quais variáveis mais influenciam o risco de inadimplência?
- Como essas variáveis se correlacionam entre si e com o comportamento de pagamento do cliente?

---

##  Metodologia

1. Análise exploratória dos dados.
2. Transformação e integração das planilhas em tabelas no BigQuery.
3. Cálculo de **variáveis dummies** e métricas como **risco relativo**, **matriz de confusão**, e avaliação de **modelos de classificação**.
4. Interpretação dos resultados e insights para tomada de decisão.

---

##  Estrutura dos Dados

Os dados estão organizados em diferentes arquivos que serão integrados no BigQuery. Abaixo, uma descrição das variáveis:

### 📁 `user_info.csv`
| Variável | Descrição |
|----------|-----------|
| `user_id` | Identificação única do cliente |
| `age` | Idade do cliente |
| `sex` | Gênero do cliente |
| `last_month_salary` | Último salário informado pelo cliente |
| `number_dependents` | Número de dependentes do cliente |

### 📁 `loans_outstanding.csv`
| Variável | Descrição |
|----------|-----------|
| `loan_id` | Identificação única do empréstimo |
| `user_id` | Identificação do cliente |
| `loan_type` | Tipo de empréstimo (`real state` ou `others`) |

### 📁 `loans_detail.csv`
| Variável | Descrição |
|----------|-----------|
| `user_id` | Identificação do cliente |
| `more_90_days_overdue` | Nº de vezes com atraso > 90 dias |
| `using_lines_not_secured_personal_assets` | Uso de crédito não garantido por bens |
| `number_times_delayed_payment_30_59` | Nº de atrasos entre 30 e 59 dias |
| `debt_ratio` | Taxa de endividamento = Dívidas / Patrimônio |
| `number_times_delayed_payment_60_89` | Nº de atrasos entre 60 e 89 dias |

### 📁 `default.csv`
| Variável | Descrição |
|----------|-----------|
| `user_id` | Identificação do cliente |
| `default_flag` | Flag de inadimplência (`1` = inadimplente, `0` = adimplente) |

---

##  Resultados Esperados

- Descoberta de **variáveis críticas** para o risco de inadimplência
- **Perfil de risco** dos clientes baseado em métricas estatísticas
- Criação de uma **pontuação de crédito interpretável**
- Melhoria nos critérios de aprovação de crédito

---

##  Tecnologias e Ferramentas

- Google Sheets
- Python (Pandas, Scikit-learn, etc.)
- Google BigQuery
- Colab 
- Looker(para dashboards)

## Etapas de Limpeza e Preparação dos Dados

user_info
Problema: Coluna last_month_salary estava como STRING, com valores em notação científica e caracteres especiais.

Solução: Conversão para INT64 com SAFE_CAST. Registros nulos foram mantidos para análise posterior.

Dependentes: 943 valores nulos identificados — sem ação por enquanto.

Duplicidade de IDs: Nenhuma encontrada.

Caracteres especiais: Identificados e tratados.

CREATE OR REPLACE TABLE `Dataset.User_info_cleaned` AS
SELECT
  user_id,
  age,
  gender,
  last_month_salary,
  number_dependents,
  SAFE_CAST(CAST(last_month_salary AS FLOAT64) AS INT64) AS last_month_salary_cleaned
FROM `Dataset.user_info`


Após identificar 425 valores nulos na tabela Loan_total, foi criada a tabela Table_complete, substituindo valores nulos por 0 usando COALESCE():

CREATE OR REPLACE TABLE `projeto-risco-relativo-458712.Dataset.Table_complete` AS
SELECT 
  user.user_id,
  user.age,
  user.gender,
  user.last_month_salary_cleaned,
  user.number_dependents,
  def.default_flag,
  det.number_times_delayed_payment_loan_30_59_days,
  det.number_times_delayed_payment_loan_60_89_days,
  det.more_90_days_overdue,
  det.debt_ratio,
  det.using_lines_not_secured_personal_assets,
  COALESCE(total.real_estate_total, 0) AS real_estate_total,
  COALESCE(total.other_total, 0) AS other_total,
  COALESCE(total.loan_total, 0) AS loan_total
FROM `User_info_cleaned` AS user
LEFT JOIN `Default` AS def ON user.user_id = def.user_id
LEFT JOIN `Loan_detail` AS det ON def.user_id = det.user_id
LEFT JOIN `Loan_total` AS total ON det.user_id = total.user_id;

## Análise Exploratória

Os dados foram visualizados em dashboards interativos:

## default e loan_detail
Nenhum valor nulo ou ID duplicado identificado.

Tabela loan_detail: criação de view com arredondamento de variáveis:

sql

SELECT *,
  ROUND(using_lines_not_secured_personal_assets, 2) AS using_lines_not_secured_personal_assets_round,
  ROUND(debt_ratio, 2) AS debt_ratio_round
FROM `Dataset.Loan_detail`

## Correlação de Variáveis (Tabela loan_detail)
Alta correlação entre variáveis de atraso:

90_dias × 60_89_dias: 0.99

90_dias × 30_59_dias: 0.98

60_89_dias × 30_59_dias: 0.98

Sem correlação com debt_ratio (valores próximos de 0 ou negativos).

## loan_outstanding
IDs duplicados permitidos (um usuário pode ter múltiplos empréstimos).

Padronização de valores categóricos na coluna loan_type: unificação para REAL ESTATE e OTHER.

## Análise de Outliers
Foram avaliadas as variáveis:

age, last_month_salary_cleaned, more_90_days_overdue, number_times_delayed_payment_loan_30_59_days, number_times_delayed_payment_loan_60_89_days, loan_total, debt_ratio, using_lines_not_secured_personal_assets.

Aplicadas estatísticas descritivas (mínimo, máximo, média, desvio padrão) para identificar valores discrepantes.

sql

SELECT 'variavel' AS variavel,
       MIN(campo),
       MAX(campo),
       AVG(campo),
       STDDEV(campo)
FROM `Dataset.tabela`

##  Pré-processamento dos Dados
## Padronização de Variáveis Categóricas
Foi identificado que a coluna loan_type continha variações de escrita como real estate, REAL ESTATE, Real Estate, etc.

Ação: padronização dos valores para REAL ESTATE e OTHER via nova coluna loan_type_cleaned na tabela Loan_outstanding.

##  Identificação de Outliers
Análise de variáveis contínuas como age, salary, debt_ratio, loan_total, entre outras, por meio de:

Mínimo, máximo, média e desvio-padrão

Cálculo da mediana separadamente

Apesar da presença de outliers, os dados foram mantidos para teste do primeiro modelo.

##  Agrupamento de Dados
Usuários com múltiplos empréstimos foram consolidados em nova tabela Loan_total:

Contagem de empréstimos por tipo (REAL ESTATE, OTHER) e total.

Query utilizada: COUNTIF agrupando por user_id.

## União das Tabelas
Criação de consulta temporária unificando os dados de:

Informações do usuário (User_info_cleaned)

Detalhes do empréstimo (Loan_detail)

Inadimplência (Default)

Total de empréstimos (Loan_total)

## Estrutura utilizada: CTEs (WITH) com LEFT JOIN para preservar todos os usuários.

## Cálculo de Quartis
A partir da tabela Table_complete, foram calculados quartis para as variáveis numéricas com a função NTILE(4) e analisada a taxa de inadimplência por quartil.

Metodologia
A análise considera:

Divisão dos clientes em quartis etários

Cálculo da taxa de inadimplência por quartil

Comparação entre o grupo exposto (cada quartil) e os demais

Cálculo do risco relativo por faixa etária

NTILE(4) OVER (ORDER BY age) AS age_quartil
Conclusões

Idade abaixo de 52 anos, menor salário e maior número de atrasos são fatores que aumentam a taxa de inadimplência.

O 4º quartil de atraso (com atrasos reais) apresenta alta inadimplência (7%), enquanto os outros são praticamente nulos.

A inadimplência diminui à medida que a renda aumenta.

Empréstimos pequenos (0–5) representam maior risco que valores médios ou altos, indicando que quantidade pode ser mais relevante que volume.

## Calculo Risco Relativo

Fórmula:
Risco Relativo = (taxa inadimplência grupo exposto) / (taxa inadimplência grupo não exposto)

-- Q1
SELECT SUM(default_flag)
FROM projeto-risco-relativo-458712.Dataset.Table_complete 
WHERE age <= 42; -- Resultado: 334

-- Q2
SELECT SUM(default_flag)
FROM projeto-risco-relativo-458712.Dataset.Table_complete 
WHERE age BETWEEN 43 AND 52; -- Resultado: 186

-- Q3
SELECT SUM(default_flag)
FROM projeto-risco-relativo-458712.Dataset.Table_complete 
WHERE age BETWEEN 53 AND 63; -- Resultado: 123

-- Q4
SELECT SUM(default_flag)
FROM projeto-risco-relativo-458712.Dataset.Table_complete 
WHERE age > 63; -- Resultado: 40

## Analisado Risco Relativo para todas as variaveis e aplicado Dummy a seguir para os seguintes Quartis com Risco Relativo apurado
Criação de Dummies e Score de Risco
Foi criado um score binário baseado nas variáveis com risco relativo acima de 1. Para isso, foi gerada uma nova tabela RR_score com as colunas dummy:

rr_age: 1 se idade ≤ 52

rr_salary: 1 se salário ≤ 4369

rr_debt: 1 se debt_ratio ≥ 0.36576736

rr_loan: 1 se loan_total ≤ 5

rr_credito: 1 se using_lines_not_secured_personal_assets > 0.5485

Score final (rr_score): soma das variáveis acima

## Validação: Matriz de Confusão
Criado um script no Colab para conexão com o BigQuery e validação do modelo binário com base no rr_score.

Pré-processamento:

Instalação de bibliotecas

Autenticação no BigQuery

Consulta da nova tabela RR_score

Critério de classificação (mau pagador):

rr_score ≥ 3 → classificado como inadimplente (mau_pagador = 1)
(Este score foi auto-ajustável manualmente para identificar uma Matriz mais aderente quanto as métricas)

Criação da matriz de confusão usando a classificação manual (mau_pagador) e o rótulo real (default_flag):

confusion_matrix e ConfusionMatrixDisplay foram utilizados corretamente.

Análise de correlação entre default_flag e as variáveis numéricas para entender a aderência:

Variáveis mais correlacionadas com inadimplência:

rr_credito (0.22)


Variáveis com correlação fraca ou irrelevante:

rr_debt foi considerada para remoção (corr ≈ 0.01)

Para esta variável com baixa correlaçao, foi removida do modelo

VAriaveis a serem utilizadas pelo modelo

features = ['rr_credito', 'rr_loan', 'rr_age', 'rr_salary']
target = 'default_flag'

## Criando Modelo manual no bigquery para analise Matriz no Python

CREATE OR REPLACE TABLE `projeto-risco-relativo-458712.Dataset.RR_v19` AS
SELECT 
  user_id,
  IF (age <= 52,1,0) AS rr_age,
  IF(last_month_salary_cleaned <= 4369, 1, 0) AS rr_salary,
  IF(loan_total <= 5, 1, 0) AS rr_loan,
  IF(using_lines_not_secured_personal_assets > 0.548523682, 1, 0) AS rr_credito,
  default_flag,

  -- Score final
  (
    IF(age between 30 and 45, 1, 0) +
    IF(last_month_salary_cleaned <= 4369, 1, 0) +
    IF(loan_total <= 5, 1, 0) +
    IF(using_lines_not_secured_personal_assets > 0.548523682, 1, 0)
  ) AS rr_score,

  -- Coluna para matriz de confusão
  IF(
    (
      IF(age between 30 and 45, 1, 0) +
      IF(last_month_salary_cleaned <= 4369, 1, 0) +
      IF(loan_total <= 5, 1, 0) +
      IF(using_lines_not_secured_personal_assets > 0.548523682, 1, 0)
    ) >= 3,
    1, 0
  ) AS mau_pagador

FROM `projeto-risco-relativo-458712.Dataset.Table_complete`

Matriz de Confusão:
[[31567  3750]  # Verdadeiros negativos / Falsos positivos
 [  367   316]] # Falsos negativos / Verdadeiros positivos

Acurácia: 0.8856

Precisão: 0.0777

Recall: 0.4627

F1-score: 0.1331

📌 Interpretação:

Acurácia alta → modelo acerta bem os bons pagadores.

Recall razoável → conseguiu capturar mais inadimplentes.

Precisão baixa → muitos falsos positivos.

## Foram testados manualmente mais de 20 modelos e optou-se por seguir com a regressao deste modelo


## Modelo de Regressao no Python 

## Criando variaveis para treinar o modelo de Regressao

from sklearn.model_selection import train_test_split

X = rr_df[features]
y = rr_df[target]

X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

## Treinando o modelo

from sklearn.linear_model import LogisticRegression

model = LogisticRegression(max_iter=1000)
model.fit(X_train, y_train)

## Fazendo previsao e gerando a Matriz

from sklearn.metrics import confusion_matrix, ConfusionMatrixDisplay

y_pred = model.predict(X_test)

cm = confusion_matrix(y_test, y_pred)
disp = ConfusionMatrixDisplay(confusion_matrix=cm, display_labels=["Bom Pagador", "Mau Pagador"])
disp.plot(cmap='Blues')
plt.title("Matriz de Confusão - Regressão Logística")
plt.show()

## Avaliando modelo com outras metrica
from sklearn.metrics import accuracy_score, precision_score, recall_score, f1_score

print("Acurácia:", accuracy_score(y_test, y_pred))
print("Precisão:", precision_score(y_test, y_pred))
print("Recall:", recall_score(y_test, y_pred))
print("F1 Score:", f1_score(y_test, y_pred))




## Exportado modelo ao Big Query atribuindo peso as variaveis conforme segue

CREATE OR REPLACE TABLE `projeto-risco-relativo-458712.Dataset.RR_v30_pesos_s` AS
SELECT 
  user_id,
  rr_age,
  rr_salary
  rr_loan,
  rr_credito,
  default_flag,

  -- Novo Score ponderado com os coeficientes da regressão logística
  (
    rr_age * 0.4695 +
    rr_salary * 0.3007 +
    rr_loan * 0.2779 +
    rr_credito * 3.7925
  ) AS rr_score_peso,

  -- Classificação final (ajustável com base em threshold que você quiser testar)
  IF(
    (
      rr_age * 0.4695 +
      rr_salary  * 0.3007 +
      rr_loan * 0.2779 +
      rr_credito * 3.7925
    ) >= 2, -- Ajuste aqui o threshold conforme testes
    1, 0
  ) AS mau_pagador_peso
FROM `projeto-risco-relativo-458712.Dataset.RR_Sensivel_v30`


## Matriz apos ajuste dos coeficientes nas variáveis

Matriz de Confusão:
[[26935  8356]
[   43   640]]
Acurácia: 0.7665
Precisão: 0.0711
Recall: 0.9370
F1-score: 0.1322

## Analise
Acurácia	0.7665	~76,6% das classificações estão corretas (mas cuidado: a base pode ser desbalanceada).
Precisão	0.0711	Das previsões como mau pagador, apenas 7,1% estavam certas. Alta taxa de falsos positivos.
Recall	0.9370	O modelo capturou 93,7% dos maus pagadores — excelente cobertura!
F1-score	0.1322	Equilíbrio entre precisão e recall, mas ainda baixo por conta da precisão ruim.

## Teste de Modelo aplicando Score >=4

Matriz de Confusão:
[[28251  7040]
[   90   593]]
Acurácia: 0.8018
Precisão: 0.0777
Recall: 0.8682
F1-score: 0.1426

Métrica	Valor	Interpretação
Acurácia	0.8018	Modelo acerta 80,2% das classificações
Precisão	0.0777	Entre os classificados como maus pagadores, apenas 7,8% realmente são
Recall	0.8682	O modelo conseguiu detectar 86,8% dos maus pagadores — excelente!
F1-score	0.1426	Melhor equilíbrio entre precisão e recall até agora


## Comparativo dos 3 modelos

##Modelo 1
Manual 	"
[31567 3750] 
[ 367 316]"	"Variaveis: 
Score ≥ 3
Idade entre 30 e 45
Salario <= 4369
Emprestimo  <= 5
Crédito> 0.548523682    "	
0.8856	0.0777	0.4627	0.1331	

Melhor acurácia, mas perde muitos maus pagadores
 --------------------------------------------
##Modelo 2
Regressão Logística (peso)	"
[26935  8356]
[   43   640]"
"Score ≥ 2
Idade <= 52 * 0.4695 
Salario  <= 4369 * 0.3007 
Emprestimo <= 5 * 0.2779 
Credito > 0.548523682 * 3.7925"	0.7665	0.0711	0.9370	0.1322	

Altíssimo recall, mas muitos falsos positivos
---------------------------------------------------

## Modelo 3
Regressão Logística (peso)	
[[28251  7040]
[   90   593]]"
"Score ≥ 4 
Idade <= 52 * 0.4695 
Salario  <= 4369 * 0.3007 
Emprestimo <= 5 * 0.2779 
Credito > 0.548523682 * 3.7925"	0.8018	0.0777	0.8682	0.1426	

Melhor equilíbrio geral — melhor F1-score
-----------------------------------------------

## Conclusao:

Modelo 3  mais equilibrado e menor risco de gerar ruído com muitos falsos positivos

## Link Apresentaçao para a empresa
https://docs.google.com/presentation/d/12mKA0pLryzs8CFV1hDvYjVhz4yhpA6ujnlo-DOn_suI/edit?usp=sharing

## Link BigQuery com os codigos
https://console.cloud.google.com/bigquery?hl=pt&invt=AbyA8A&project=projeto-risco-relativo-458712&ws=!1m18!1m3!8m2!1s129115836155!2sf78870450eab43f1a3d5578fc7855d6b!1m4!4m3!1sprojeto-risco-relativo-458712!2sDataset!3sCorrelacao_inadimplencia!1m4!1m3!1sprojeto-risco-relativo-458712!2sbquxjob_7d7da30c_196f93a2edb!3sUS!1m3!8m2!1s129115836155!2s855779718f2347a1ba5719350e717153&inv=1

## Link dos Dashboards e graficos de analise
https://lookerstudio.google.com/reporting/71d34169-8954-4733-bfec-44caa1674d81



FROM `projeto-risco-relativo-458712.Dataset.RR_v19`
---

##  Contribuições

Fique à vontade para abrir issues ou pull requests com sugestões, melhorias ou correções.

---
Fluxograma do Modelo Gerado
![Captura de Tela 2025-05-23 às 14 44 36](https://github.com/user-attachments/assets/68ec3c57-875d-4d8f-b1d9-60fe35d238c7)



