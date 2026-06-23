# iFood – Case Técnico Data Science

Solução para otimização de distribuição de cupons e ofertas usando dados históricos de transações, perfis de clientes e metadados de ofertas.

## Estrutura do Repositório

```
ifood-case/
├── data/
│   ├── raw/                    # Dados originais (offers.json, profile.json, transactions.json)
│   └── processed/              # Dados processados (gerados pelo notebook 1)
├── notebooks/
│   ├── 1_data_processing.ipynb # Processamento com PySpark
│   └── 2_modeling.ipynb        # EDA, modelagem e impacto de negócio
├── presentation/               # Imagens geradas para os slides
├── requirements.txt
└── README.md
```

## Pré-requisitos

- Python 3.10+
- Java 11+ (necessário para PySpark)

```bash
pip install -r requirements.txt
```

## Como Executar

### 1. Baixar os dados

```bash
mkdir -p data/raw
wget -O data/raw/data.tar.gz \
  "https://data-architect-test-source.s3.sa-east-1.amazonaws.com/ds-technical-evaluation-data.tar.gz"
tar -xzf data/raw/data.tar.gz -C data/raw/ --strip-components=1
```

### 2. Notebook de Processamento (PySpark)

```bash
jupyter notebook notebooks/1_data_processing.ipynb
```

Executa as células em ordem. Ao final, gera:
- `data/processed/interactions_features.csv`
- `data/processed/interactions_features.parquet`

> **Databricks:** Faça upload dos arquivos `.json` para o DBFS e ajuste a variável `RAW` na célula de configuração.

### 3. Notebook de Modelagem

```bash
jupyter notebook notebooks/2_modeling.ipynb
```

Requer o CSV gerado no passo anterior. Ao final, gera:
- `data/processed/all_offer_scores.csv` — scores para todos os pares cliente×oferta
- `data/processed/recommendations.csv` — melhor oferta por cliente
- Imagens em `presentation/` para os slides

## Abordagem

### Problema
Classificação binária: dado um par `(cliente, oferta)`, prever se o cliente converterá.

### Features
| Grupo | Features |
|---|---|
| Perfil do cliente | age, gender, credit_card_limit, account_age_days |
| Comportamento transacional | n_transactions, total_spent, avg_transaction, avg_daily_spend |
| Oferta | offer_type, min_value, duration, discount_value, canais |
| Interação | discount_to_limit_ratio, min_value_vs_avg_spend |

### Modelo
**LightGBM** com validação cruzada estratificada de 5 folds.
- `is_unbalance=True` para lidar com desbalanceamento de classes.
- Early stopping para evitar overfitting.
- Métrica principal: AUC-ROC; secundária: Average Precision.

### Recomendação
Para cada cliente, a oferta recomendada é a que maximiza o **Expected Value**:

```
EV = P(conversão | cliente, oferta) × discount_value
```

### Premissas
- `age = 118` é valor sentinela para perfis incompletos.
- `offer id` (com espaço) nos eventos de transação é o identificador da oferta.
- Para ofertas *informational*, sucesso = visualização (não há evento de completion).
- Matching oferta-cliente feito por agregação (sem controle estrito de janela temporal).
- `time_since_test_start` está em horas.
