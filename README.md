# 🚖 iFood Data Architecture Case — NYC Yellow Taxi

Este repositório implementa um pipeline **reprodutível** para ingestão, padronização, modelagem e análise dos dados públicos da **NYC TLC** (Yellow Taxi) com **PySpark** e **Delta Lake**, pensado para rodar no **Databricks Community Edition** (sem acesso à internet).

> **Escopo do case**: ingerir dados **Jan–Mai/2023**, disponibilizar via SQL, e responder perguntas analíticas específicas. Também são exigidas as colunas: `VendorID`, `passenger_count`, `total_amount`, `tpep_pickup_datetime`, `tpep_dropoff_datetime`.


## 🧭 Sumário
- [Arquitetura de Dados](#arquitetura-de-dados)
- [Padrões de Entrega & Conformidade](#padrões-de-entrega--conformidade)
- [Pré-requisitos](#pré-requisitos)
- [Como obter os dados (offline)](#como-obter-os-dados-offline)
- [Execução no Databricks CE (passo a passo)](#execução-no-databricks-ce-passo-a-passo)
- [Modelagem e Particionamento](#modelagem-e-particionamento)
- [Consultas de Resposta (SQL)](#consultas-de-resposta-sql)
- [Validações de Qualidade de Dados](#validações-de-qualidade-de-dados)
- [Dicionário de Dados (Silver)](#dicionário-de-dados-silver)
- [Boas práticas e performance](#boas-práticas-e-performance)
- [Estrutura do Repositório](#estrutura-do-repositório)

## 🏛️ Arquitetura de Dados

Camadas (Data Lake):
- **RAW (landing)**: arquivos originais exatamente como baixados (Parquet). _Somente leitura_.
- **Bronze (SOR)**: base com historico desde 2020 com normalização mínima de tipos/nomes, adição de coluna técnica `anomes` (YYYYMM).
- **Silver (consumo)**: projeção e filtragem para o escopo do case (Jan–Mai/2023) e **apenas as colunas obrigatórias** e um leve filtro de data quality.

Tecnologias:
- **PySpark** para ETL/ELT.
- **Delta Lake** para armazenamento e versionamento.
- **SQL** para consultas analíticas aos consumidores.


## ✅ Padrões de Entrega & Conformidade

Este README incorpora e **mapeia os requisitos do case** às entregas do projeto:

- **Ingestão no Data Lake** dos dados Yellow Taxi: **OK** (seções _Como obter os dados_ e _Execução no Databricks_).  
- **Disponibilização para consumo (SQL)**: **OK** (tabelas Delta, seção _Consultas de Resposta_).  
- **Uso de PySpark**: **OK** (pipelines nas camadas Bronze/Silver).  
- **Colunas obrigatórias presentes**: `VendorID`, `passenger_count`, `total_amount`, `tpep_pickup_datetime`, `tpep_dropoff_datetime`.  
- **Período Jan–Mai/2023**: **OK**, filtrado na Silver.  
- **Análises respondidas**: consultas SQL prontas (média `total_amount` por mês e média `passenger_count` por hora em Maio).  
- **Qualidade, organização e justificativas**: ver _Arquitetura_, _Validações_ e _Boas práticas_.


## 🧩 Pré-requisitos

- Conta no **Databricks Community Edition**.
- **Sem internet no cluster**: o download deve ser feito **fora** do Databricks e os arquivos enviados para um **Volume**.
- Python 3.10+ (apenas se for baixar localmente).  
- O arquivo `requirements.txt` lista dependências para execução local (não necessárias no Databricks).


## 📥 Como obter os dados (offline)

1) **Baixe localmente** os Parquet de **Yellow Taxi** (Jan–Mai/2023).

Script exemplo abaixo:

```python
import os, requests

OUT = "nyc_taxi_yellow_2020_2025"
os.makedirs(OUT, exist_ok=True)

for year in range(2020, 2026):
    for month in range(1, 13):
        url = f"https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_{year}-{month:02d}.parquet"
        path = os.path.join(OUT, f"yellow_{year}-{month:02d}.parquet")

        if os.path.exists(path):
            print("skip", path)
            continue

        r = requests.get(url)
        if r.status_code == 200:
            with open(path, "wb") as f:
                f.write(r.content)
            print("ok", url)
        else:
            print("miss", url, r.status_code)

```
2) **Envie os arquivos** via UI do Databricks para um **Volume** (recomendado) ou pasta gerenciada:

```
/Volumes/<catalog>/<schema>/nyc_taxi/raw/2023/
```


## ▶️ Execução no Databricks CE (passo a passo)

1. **Crie o Volume e Faça upload** dos arquivos Parquet (Jan–Mai/2023) para:
   ```
   /Volumes/workspace/nyc_taxi/raw/2023/
   ```

2. **Bronze — ingestão e normalização mínima**  
   Execute o notebook `01_ingestao_bronze.ipynb`, que:
   - Lê todos os Parquet em `/Volumes/workspace/nyc_taxi/raw/2023/*.parquet`
   - Padroniza schema e nomes
   - Cria `anomes` (YYYYMM) a partir de `tpep_pickup_datetime`
   - Persiste a Delta Table:
     ```
     workspace.nyc_taxi.yellowtaxi_trips_sor
     ```

3. **Silver — projeção para o case**
   Execute `02_transformacao_silver.ipynb`, que:
   - Seleciona **apenas** as colunas obrigatórias
   - Filtra **2023-01** a **2023-05**
   - Particiona por `anomes` e salva como Delta:
     ```
     workspace.nyc_taxi.yellowtaxi_trips_2023_spec
     ```

4. **Análises**
   Execute `analysis.ipynb` (ou as consultas SQL abaixo).  


## 🧱 Modelagem e Particionamento

- **Chave de partição**: `anomes` (YYYYMM) calculado de `tpep_pickup_datetime` para refletir a **data efetiva do evento**.  
- **Extração de metadados do arquivo**: quando necessário, use `_metadata.file_path` (Spark/Delta) — substitui `input_file_name()` em ambientes UC.  
- **Nomes padronizados**: `snake_case`, colunas castadas para tipos consistentes.


## 🧪 Consultas de Resposta (SQL)

**1) Média de `total_amount` por mês (Jan–Mai/2023):**
```sql
SELECT anomes, ROUND(AVG(total_amount), 2) AS media_total_amount
FROM workspace.nyc_taxi.yellowtaxi_trips_2023_spec
GROUP BY anomes
ORDER BY anomes;
```

**2) Média de `passenger_count` por hora do dia em Maio/2023:**
```sql
SELECT HOUR(tpep_pickup_datetime) AS hora_do_dia,
       ROUND(AVG(passenger_count), 2) AS media_passageiros
FROM workspace.nyc_taxi.yellowtaxi_trips_2023_spec
WHERE anomes = '202305'
GROUP BY hora_do_dia
ORDER BY hora_do_dia;
```


## 🔍 Validações de Qualidade de Dados

- **Presença de colunas obrigatórias** (fail-fast): checar existência no DataFrame antes de escrever a Silver.
- **Controles de nulidade**: `VendorID`, `tpep_pickup_datetime`, `tpep_dropoff_datetime` não devem ser nulos na Silver.
- **Regras de domínio** (exemplos):
  - `passenger_count >= 0`
  - `total_amount` pode ser negativo (ajustes), mas monitore outliers extremos.
  - `tpep_dropoff_datetime >= tpep_pickup_datetime`
- **Volumetria por partição**: alerta se partições de `anomes` tiverem volume anômalo.


## 📚 Dicionário de Dados (Silver)

| Campo                   | Tipo      | Descrição                                               |
|------------------------|-----------|---------------------------------------------------------|
| `vendorid`             | LONG      | Identificador do provedor                               |
| `passenger_count`      | INT       | Número de passageiros                                   |
| `total_amount`         | DOUBLE    | Valor total da corrida                                  |
| `tpep_pickup_datetime` | TIMESTAMP | Data/hora do embarque                                   |
| `tpep_dropoff_datetime`| TIMESTAMP | Data/hora do desembarque                                |
| `anomes`               | STRING    | Partição no formato `YYYYMM` (derivada do pickup time)  |


## ⚙️ Boas práticas e performance

- **Delta Lake**: transações ACID e _time travel_ para auditoria e reprocessos.
- **Particionamento por `anomes`**: melhora _pruning_ em filtros mensais.

## 🗂️ Estrutura do Repositório

```
ifood-case/
├─ src/
│  ├─ 01_ingestao_bronze.ipynb
│  ├─ 02_transformacao_silver.ipynb
├─ analysis/
│  └─ analysis.ipynb
├─ README.md
└─ requirements.txt
```



