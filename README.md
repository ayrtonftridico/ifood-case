# 🚖 iFood Data Architecture Case — NYC Yellow Taxi (README aprimorado)

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
- [Erros comuns & Troubleshooting](#erros-comuns--troubleshooting)
- [Estrutura do Repositório](#estrutura-do-repositório)
- [Licença](#licença)


## 🏛️ Arquitetura de Dados

Camadas (Data Lake):
- **RAW (landing)**: arquivos originais exatamente como baixados (Parquet). _Somente leitura_.
- **Bronze (SOR)**: normalização mínima de tipos/nomes, adição de coluna técnica `anomes` (YYYYMM), controle de ingestão.
- **Silver (consumo)**: projeção e filtragem para o escopo do case (Jan–Mai/2023) e **apenas as colunas obrigatórias**.
- **(Opcional) Gold**: agregações e métricas derivadas (não requerido, mas útil para apresentações).

Tecnologias:
- **PySpark** para ETL/ELT.
- **Delta Lake** para armazenamento, versionamento e _time travel_.
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

1) **Baixe localmente** os Parquet de **Yellow Taxi** (Jan–Mai/2023). Um script de exemplo (fora do Databricks) está em `scripts/download_tlc.py` (ou use o snippet deste README).  
2) **Envie os arquivos** via UI do Databricks para um **Volume** (recomendado) ou pasta gerenciada:

```
/Volumes/<catalog>/<schema>/nyc_taxi/raw/2023/
```

> **Por que Volume?** O Databricks CE costuma restringir DBFS público e o cluster não tem internet. Volumes funcionam bem com Unity Catalog.


## ▶️ Execução no Databricks CE (passo a passo)

> **Observação**: Se seu ambiente **não** tiver Unity Catalog, substitua nomes `catalog.schema.tabela` por `hive_metastore.default.tabela` e paths de `/Volumes/...` por um caminho suportado (ex.: `/mnt/...`).

1. **Crie catálogo/esquema/volumes** (Unity Catalog):
   ```sql
   CREATE CATALOG IF NOT EXISTS workspace;
   CREATE SCHEMA  IF NOT EXISTS workspace.nyc_taxi;
   CREATE VOLUME  IF NOT EXISTS workspace.nyc_taxi.raw;
   CREATE VOLUME  IF NOT EXISTS workspace.nyc_taxi.silver;
   ```

2. **Faça upload** dos arquivos Parquet (Jan–Mai/2023) para:
   ```
   /Volumes/workspace/nyc_taxi/raw/2023/
   ```

3. **Bronze — ingestão e normalização mínima**  
   Execute o notebook `01_ingestao_bronze.py`, que:
   - Lê todos os Parquet em `/Volumes/workspace/nyc_taxi/raw/2023/*.parquet`
   - Padroniza schema e nomes
   - Cria `anomes` (YYYYMM) a partir de `tpep_pickup_datetime`
   - Persiste a Delta Table:
     ```
     workspace.nyc_taxi.yellowtaxi_trips_sor
     ```

4. **Silver — projeção para o case**
   Execute `02_transformacao_silver.py`, que:
   - Seleciona **apenas** as colunas obrigatórias
   - Filtra **2023-01** a **2023-05**
   - Particiona por `anomes` e salva como Delta:
     ```
     workspace.nyc_taxi.yellowtaxi_trips_2023_silver
     ```

5. **Análises**
   Execute `03_analises.py` (ou as consultas SQL abaixo).  
   (Opcional) `04_visualizacoes.py` gera gráficos de apoio.


## 🧱 Modelagem e Particionamento

- **Chave de partição**: `anomes` (YYYYMM) calculado de `tpep_pickup_datetime` para refletir a **data efetiva do evento**.  
- **Extração de metadados do arquivo**: quando necessário, use `_metadata.file_path` (Spark/Delta) — substitui `input_file_name()` em ambientes UC.  
- **Nomes padronizados**: `snake_case`, colunas castadas para tipos consistentes.


## 🧪 Consultas de Resposta (SQL)

**1) Média de `total_amount` por mês (Jan–Mai/2023):**
```sql
SELECT anomes, ROUND(AVG(total_amount), 2) AS media_total_amount
FROM workspace.nyc_taxi.yellowtaxi_trips_2023_silver
GROUP BY anomes
ORDER BY anomes;
```

**2) Média de `passenger_count` por hora do dia em Maio/2023:**
```sql
SELECT HOUR(tpep_pickup_datetime) AS hora_do_dia,
       ROUND(AVG(passenger_count), 2) AS media_passageiros
FROM workspace.nyc_taxi.yellowtaxi_trips_2023_silver
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
| `vendorid`             | INT       | Identificador do provedor                               |
| `passenger_count`      | INT       | Número de passageiros                                   |
| `total_amount`         | DOUBLE    | Valor total da corrida                                  |
| `tpep_pickup_datetime` | TIMESTAMP | Data/hora do embarque                                   |
| `tpep_dropoff_datetime`| TIMESTAMP | Data/hora do desembarque                                |
| `anomes`               | STRING    | Partição no formato `YYYYMM` (derivada do pickup time)  |


## ⚙️ Boas práticas e performance

- **Delta Lake**: transações ACID e _time travel_ para auditoria e reprocessos.
- **Particionamento por `anomes`**: melhora _pruning_ em filtros mensais.
- **Auto-Optimize** (se disponível) e **OPTIMIZE + ZORDER** (opcional) em colunas de tempo.
- **Idempotência**: pipelines projetados para reprocesso seguro de partições.


## 🧯 Erros comuns & Troubleshooting

- **Sem internet no Databricks CE / `UnknownHostException`**: baixe os Parquet **fora** do cluster e faça upload para **Volumes**.
- **`Public DBFS root is disabled`**: prefira `/Volumes/<catalog>/<schema>/<volume>/...`.
- **Unity Catalog — `input_file_name()` não suportado**: use **`_metadata.file_path`** para extrair ano/mês do caminho.
- **`DATATYPE_MISMATCH` ao concatenar strings** no Spark SQL: utilize `concat()` ou `format_string()` em vez de `+` para strings.


## 🗂️ Estrutura do Repositório

```
ifood-case/
├─ src/
│  ├─ 01_ingestao_bronze.py
│  ├─ 02_transformacao_silver.py
│  ├─ 03_analises.py
│  └─ 04_visualizacoes.py              # opcional
├─ scripts/
│  └─ download_tlc.py                  # download offline (local)
├─ notebooks/                          # (opcional) versões em notebook
├─ README.md
└─ requirements.txt
```


## 📄 Licença

Uso educacional para execução do case técnico. Dados originais pertencem à NYC TLC (uso público).


---

### 📎 Snippet opcional — Download local (fora do Databricks)

```python
import os, requests
from concurrent.futures import ThreadPoolExecutor, as_completed

BASE_URL = "https://d37ci6vzurychx.cloudfront.net/trip-data/{}_tripdata_{}-{}.parquet"
CATEGORIES = ["yellow"]
YEARS = ["2023"]
MONTHS = [f"{i:02d}" for i in range(1, 6)]  # Jan–Mai

OUTPUT_DIR = "nyc_taxi_raw"
os.makedirs(OUTPUT_DIR, exist_ok=True)

def download_file(cat, year, month):
    url = BASE_URL.format(cat, year, month)
    filename = os.path.join(OUTPUT_DIR, f"{cat}_{year}-{month}.parquet")
    if os.path.exists(filename):
        return f"⏩ Já existe: {filename}"
    try:
        r = requests.get(url, timeout=60)
        r.raise_for_status()
        with open(filename, "wb") as f:
            f.write(r.content)
        return f"✅ Baixado: {url}"
    except Exception as e:
        return f"⚠️ Erro ao baixar {url}: {e}"

with ThreadPoolExecutor(max_workers=8) as ex:
    futures = [ex.submit(download_file, c, y, m) for c in CATEGORIES for y in YEARS for m in MONTHS]
    for f in as_completed(futures):
        print(f.result())
```
