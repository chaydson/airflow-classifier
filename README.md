# airflow-classifier

Pipeline de aprendizado de máquina orquestrado pelo Apache Airflow para classificação automática de propostas da plataforma [Brasil Participativo](https://brasilparticipativo.presidencia.gov.br). O DAG roda diariamente, baixa os dados abertos mais recentes, treina um classificador e salva o modelo atualizado no formato ONNX.

---

## Sumário

- [Visão Geral](#visão-geral)
- [Arquitetura](#arquitetura)
- [Pré-requisitos](#pré-requisitos)
- [Instalação](#instalação)
- [Executando o Airflow](#executando-o-airflow)
- [Estrutura do Projeto](#estrutura-do-projeto)
- [Pipeline de Dados](#pipeline-de-dados)
- [Modelo](#modelo)

---

## Visão Geral

O **airflow-classifier** automatiza o ciclo de vida de um modelo de classificação de texto em português. A cada dia ele:

1. Baixa o arquivo de dados abertos do Brasil Participativo.
2. Combina esses dados com propostas simuladas locais.
3. Executa um pipeline de pré-processamento de NLP (limpeza, remoção de stopwords, desacentuação, TF-IDF).
4. Treina um classificador **LinearSVC** para categorizar propostas por ministério/órgão responsável.
5. Exporta o modelo treinado para o formato **ONNX** (`model/proposal_classifier_v1_1_0.onnx`).

---

## Arquitetura

```
Brasil Participativo Open Data (CSV)
            │
            ▼
   Limpeza e pré-processamento
  (spaCy · NLTK · Unidecode · regex)
            │
            ▼
    Vetorização TF-IDF
   (bigramas, 50 000 features)
            │
            ▼
    Treinamento LinearSVC
            │
            ▼
  Exportação para ONNX
  model/proposal_classifier_v1_1_0.onnx
```

O DAG `model_dag` é agendado com `@daily` e gerenciado pelo Apache Airflow 2.7.

---

## Pré-requisitos

- Python 3.8+
- Apache Airflow 2.7.3
- pip

---

## Instalação

```bash
# Clone o repositório
git clone https://github.com/chaydson/airflow-classifier.git
cd airflow-classifier

# Instale as dependências
pip install -r requirements.txt
```

> **Nota:** O modelo de linguagem `pt_core_news_lg` do spaCy é instalado automaticamente via `requirements.txt`, pois está listado como dependência direta.

---

## Executando o Airflow

```bash
# Inicializa o banco de dados do Airflow (apenas na primeira vez)
airflow db init

# Inicia o scheduler em segundo plano
airflow scheduler &

# Inicia a interface web
airflow webserver --port 8080
```

Acesse `http://localhost:8080` e ative o DAG **model_dag** na interface. Ele também pode ser acionado manualmente:

```bash
airflow dags trigger model_dag
```

---

## Estrutura do Projeto

```
airflow-classifier/
├── dags/
│   └── model_dag.py            # Definição do DAG e lógica de treinamento
├── model/
│   └── proposal_classifier_v1_1_0.onnx  # Modelo treinado (gerado pelo DAG)
├── production/
│   └── data_extraction/
│       └── propostas_simuladas.csv       # Propostas sintéticas para enriquecer o treino
├── requirements.txt            # Dependências Python
├── airflow.cfg                 # Configurações do Airflow
└── webserver_config.py         # Configurações do webserver do Airflow
```

---

## Pipeline de Dados

| Etapa | Descrição |
|---|---|
| **Download** | Baixa o ZIP de dados abertos do Brasil Participativo e extrai o CSV de propostas |
| **Limpeza** | Remove tags HTML e trechos desnecessários com regex |
| **Enriquecimento** | Concatena propostas reais com `propostas_simuladas.csv` |
| **Stopwords** | Remove stopwords em português usando NLTK |
| **Desacentuação** | Normaliza texto retirando acentos com Unidecode |
| **TF-IDF** | Vetoriza com bigramas e máximo de 50 000 features |
| **Split** | Divide em treino (80%) e teste (20%) com estratificação |

---

## Modelo

- **Algoritmo:** `LinearSVC` com `class_weight='balanced'`
- **Formato de saída:** ONNX 1.15 (`model/proposal_classifier_v1_1_0.onnx`)
- **Entrada:** vetor TF-IDF `float32` com até 50 000 dimensões
- **Saída:** categoria do órgão/ministério responsável pela proposta
