# 🚖 iFood Data Architecture Case — NYC Taxis (Yellow, Green, FHV, FHVHV)

Este projeto implementa um pipeline **reprodutível** para ingestão, padronização, modelagem e análise dos dados públicos da **NYC TLC** usando **PySpark** e **Delta Lake** no **Databricks Community Edition** (sem acesso à internet).
O foco do case é responder às perguntas para **Jan–Mai/2023**, com ênfase nas colunas exigidas no enunciado: `VendorID`, `passenger_count`, `total_amount`, `tpep_pickup_datetime`, `tpep_dropoff_datetime`.


> - **Yellow**: Foram utilizados todos os arquivos de **2020–2025** para exemplificar práticas de camadas bronze-silver (robustez e volume).
> - **Green, FHV & FHVHV:** apenas **maio/2023** (suficiente para a segunda questão).
> - Todas as análises que exigem “todas as frotas” usam **todos os dados disponíveis por frota** acima; onde uma coluna não existe em determinada frota (ex.: `passenger_count` em FHV/HV), a métrica é computada sobre as frotas que fornecem o campo (documentado nas consultas).

---

## 🧭 Sumário
- [Arquitetura de Dados](#arquitetura-de-dados)
- [Cobertura de Dados (por frota)](#cobertura-de-dados-por-frota)
- [Pré-requisitos](#pré-requisitos)
- [Como obter os dados (offline)](#como-obter-os-dados-offline)
- [Execução no Databricks CE (passo a passo)](#execução-no-databricks-ce-passo-a-passo)
- [Modelagem e Particionamento](#modelagem-e-particionamento)
- [Consultas de Resposta (SQL)](#consultas-de-resposta-sql)
- [Validações de Qualidade de Dados](#validações-de-qualidade-de-dados)
- [Boas práticas e performance](#boas-práticas-e-performance)
- [Estrutura do Repositório](#estrutura-do-repositório)

---

## 🏛️ Arquitetura de Dados

Camadas (Data Lake):
- **RAW**: arquivos originais exatamente como baixados (Parquet). _Somente leitura_.
- **Bronze**: normalização mínima de tipos/nomes, coluna técnica `anomes` (YYYYMM) derivada de `*_pickup_datetime`, e **coluna `category`** identificando a frota: `yellow|green|fhv|fhvhv`.
- **Silver**: projeção para o escopo do case (Jan–Mai/2023), colunas solicitadas e filtros de qualidade leves.
- **Gold**: 

Tecnologias:
- **PySpark** para ETL/ELT, **Delta Lake** para armazenamento, **SQL** para consumo analítico.

---

## 📊 Cobertura de Dados (por frota)

| Frota   | Período carregado | Observações de schema |
|---------|--------------------|-----------------------|
| Yellow  | 2020–2025 (todos os meses disponíveis) | Campos `tpep_*`, `passenger_count`, `total_amount` presentes. |
| Green   | **somente 2023-05**                     | Similar a Yellow; colunas `lpep_*` em anos antigos são normalizadas. |
| FHV     | **somente 2023-05**                     | Geralmente **não** possui `passenger_count` e `total_amount`. |
| FHVHV   | **somente 2023-05**                     | Geralmente **não** possui `passenger_count` nem `total_amount`. |

> **Nota metodológica:** quando uma métrica depende de uma coluna ausente numa frota (p.ex. `passenger_count`), a agregação considera **apenas as frotas que fornecem o campo** (Yellow/Green). Isso é explicitado nas consultas SQL.

---

## 🧩 Pré-requisitos

- Conta no **Databricks Community Edition** (cluster sem internet).
- Download **offline** prévio (feito localmente) e upload para **Volumes** do UC.
- Python 3.10+ localmente apenas se for usar o script de download.
- O `requirements.txt` auxilia na execução local (não é necessário no Databricks).

---

## 📥 Como obter os dados (offline)

Como o Databricks CE **não tem internet**, baixe localmente e depois faça **upload** para um **Volume**. Script exemplo (ajuste anos/meses conforme sua estratégia):

```python
import os, requests
from concurrent.futures import ThreadPoolExecutor, as_completed

BASE = "https://d37ci6vzurychx.cloudfront.net/trip-data/{}_tripdata_{}-{}.parquet"
CATEGORIES = ["yellow", "green", "fhv", "fhvhv"]
YEARS_FULL = [str(y) for y in range(2020, 2026)]  # uso em yellow/green
MONTHS_FULL = [f"{m:02d}" for m in range(1, 13)]
YEARS_MAY = ["2023"]                               # uso em fhv/fhvhv (maio)
MONTHS_MAY = ["05"]

def plan():
    for cat in CATEGORIES:
        if cat in ("yellow", "green"):
            years, months = YEARS_FULL, MONTHS_FULL
        else:
            years, months = YEARS_MAY, MONTHS_MAY
        for y in years:
            for m in months:
                yield cat, y, m

OUT = "nyc_taxi_raw"
os.makedirs(OUT, exist_ok=True)

def fetch(cat, y, m):
    url = BASE.format(cat, y, m)
    path = os.path.join(OUT, f"{cat}_{y}-{m}.parquet")
    if os.path.exists(path):
        return f"skip {path}"
    try:
        r = requests.get(url, timeout=60)
        if r.status_code == 200:
            with open(path, "wb") as f:
                f.write(r.content)
            return f"ok   {url}"
        return f"miss {url} ({r.status_code})"
    except Exception as e:
        return f"err  {url} ({e})"

with ThreadPoolExecutor(max_workers=8) as ex:
    futures = [ex.submit(fetch, *args) for args in plan()]
    for fut in as_completed(futures):
        print(fut.result())
```

**Upload sugerido (Volumes UC):**
```
/Volumes/workspace/nyc_taxi/raw/yellow/2020-2025/*.parquet
/Volumes/workspace/nyc_taxi/raw/green/2020-2025/*.parquet
/Volumes/workspace/nyc_taxi/raw/fhv/2023-05/*.parquet
/Volumes/workspace/nyc_taxi/raw/fhvhv/2023-05/*.parquet
```

---

## ▶️ Execução no Databricks CE (passo a passo)

1. **Crie os Volumes e faça o upload** conforme a estrutura acima.

2. **Bronze — ingestão e normalização mínima**  
   Notebook `01_ingestao_bronze.ipynb` (exemplo de lógica):
   - Leitura por categoria (diretórios diferentes), padronização de nomes (`lower`), cast para schema alvo **por frota** e criação da coluna `category`.
   - Derivação de `anomes` a partir de `*_pickup_datetime` (ex.: `tpep_pickup_datetime`, `lpep_pickup_datetime`, `pickup_datetime`).
   - União com `unionByName(allowMissingColumns=True)` entre frotas.
   - **Filtro de partições esparsas** (ex.: descartar `anomes` com `< 10k` linhas).
   - Persistência da Delta Table particionada por `anomes`:
     ```
     workspace.nyc_taxi.trips_sor
     ```

3. **Silver — projeção para o case**  
   Notebook `02_transformacao_silver.ipynb`:
   - Seleção das colunas para o case. Para **compatibilizar frotas**, normalizamos nomes de data/hora para:
     - `pickup_datetime` e `dropoff_datetime` (derivadas de `tpep_*`, `lpep_*` ou `*_datetime`).
   - Filtro **2023-01** a **2023-05**.
   - Particionamento por `anomes` e gravação:
     ```
     workspace.nyc_taxi.trips_2023_silver
     ```

4. **Análises**  
   Notebook `analysis.ipynb` com consultas SQL (abaixo).

---

## 🧱 Modelagem e Particionamento

- **Chaves técnicas:** `category` (`yellow|green|fhv|fhvhv`) e `anomes` (`YYYYMM`).
- **Datas unificadas na Silver:** `pickup_datetime`, `dropoff_datetime`.
- **Particionamento por `anomes`** (reflete o **evento real** e otimiza _partition pruning_).

---

## 🧪 Consultas de Resposta (SQL)

**1) Média de `total_amount` por mês (Jan–Mai/2023, apenas frotas que possuem `total_amount`):**
```sql
SELECT anomes,
       ROUND(AVG(total_amount), 2) AS media_total_amount
FROM workspace.nyc_taxi.trips_2023_silver
WHERE anomes BETWEEN '202301' AND '202305'
  AND total_amount IS NOT NULL            -- frotas sem esse campo não entram
GROUP BY anomes
ORDER BY anomes;
```

**2) Média de `passenger_count` por hora do dia em Maio/2023 (frotas que possuem `passenger_count`: Yellow/Green):**
```sql
SELECT HOUR(pickup_datetime) AS hora_do_dia,
       ROUND(AVG(passenger_count), 2) AS media_passageiros
FROM workspace.nyc_taxi.trips_2023_silver
WHERE anomes = '202305'
  AND passenger_count IS NOT NULL         -- FHV e FHVHV normalmente não possuem esse campo
GROUP BY hora_do_dia
ORDER BY hora_do_dia;
```

> Se desejar, você pode adicionar um `GROUP BY category` para comparar o comportamento entre frotas quando a coluna existir.

---

## 🔍 Validações de Qualidade de Dados

- **Presença de colunas obrigatórias** na Silver.
- **Nulidade e domínios:**
  - `pickup_datetime`/`dropoff_datetime` não nulos.
  - `passenger_count >= 0` quando existir.
  - `total_amount >= 0` (admitindo eventuais negativos por ajustes, mas monitorando outliers).
  - `dropoff_datetime >= pickup_datetime`.
- **Volumetria por partição** e alerta para `anomes` com baixíssimo volume.

---

## ⚙️ Boas práticas e performance

- **Delta Lake** (ACID, _time travel_, vacuum).
- **Leitura em lote** por diretório e `unionByName` com `allowMissingColumns` para esquemas diferentes.
- **`repartition("anomes")`** antes de escrever, melhorando o layout físico das partições.

---

## 🗂️ Estrutura do Repositório

```
ifood-case/
├─ src/
│  ├─ 01_ingestao_bronze.ipynb
│  └─ 02_transformacao_silver.ipynb
├─ analysis/
│  └─ analysis.ipynb
├─ README.md
└─ requirements.txt
```

---

## 📌 Notas finais

- As escolhas de cobertura por frota (**2020–2025** para Yellow/Green; **maio/2023** para FHV/FHVHV) foram feitas para **equilibrar escala** e **limitações** do Databricks CE, mantendo fidelidade ao enunciado da segunda questão.
- Onde uma coluna não existe na fonte, a métrica é calculada sobre as frotas que **de fato fornecem** o campo — isso fica explícito nos `WHERE ... IS NOT NULL` das consultas.
