# 🚖 iFood Data Architecture Case — NYC Taxis (Yellow, Green, FHV, FHVHV)

Este projeto implementa um pipeline **reprodutível** para ingestão, padronização, modelagem e análise dos dados públicos da **NYC TLC** usando **PySpark** e **Delta Lake** no **Databricks Community Edition** (sem acesso à internet).

> - **Yellow**: Foram utilizados todos os arquivos de **2020–2025** para exemplificar práticas de camadas bronze-silver (robustez e volume).
> - **Green, FHV & FHVHV:** apenas **maio/2023** (suficiente para a segunda questão).
---

## 🧭 Sumário
- [Arquitetura de Dados](#%EF%B8%8F-arquitetura-de-dados)
- [Cobertura de Dados (por frota)](#-cobertura-de-dados-por-frota)
- [Pré-requisitos](#-pré-requisitos)
- [Como obter os dados (offline)](#-como-obter-os-dados-offline)
- [Execução no Databricks CE (passo a passo)](#%EF%B8%8F-execução-no-databricks-ce-passo-a-passo)
- [Consultas de Resposta (SQL)](#-consultas-de-resposta-sql)
- [Validações de Qualidade de Dados](#-validações-de-qualidade-de-dados)
- [Boas práticas e performance](#%EF%B8%8F-boas-práticas-e-performance)
- [Estrutura do Repositório](#%EF%B8%8F-estrutura-do-repositório)

---

## 🏛️ Arquitetura de Dados

Camadas (Data Lake):
- **RAW**: arquivos originais exatamente como baixados (Parquet).
- **Bronze**: normalização mínima de tipos/nomes, coluna técnica `anomes` (YYYYMM) derivada da data de pickup de cada schema.
- **Silver**: aplicação de conceitos de Data Quality. Remove linhas com colunas essenciais vazias, linhas com pickupdate < dropoffdate, linhas com passageiro =< 0.
- **Gold**: criação das duas camadas gold utilizadas especificamente para a resposta das questões do case. A primeira é uma tabela com as informações das viagens do yellow taxi de janeiro a maio de 2023. A segunda é uma junção das informações de pickupdate e número de passageiros(apenas yellow e green possuem) de todos os tipos de taxi para maio de 2023.

Tecnologias:
- **PySpark** para ETL/ELT, **Delta Lake** para armazenamento, **SQL** para consumo analítico.

---

## 📊 Cobertura de Dados (por frota)

| Frota   | Período carregado |
|---------|--------------------|
| Yellow  | 2020–2025 (todos os meses disponíveis) | 
| Green   | **somente 2023-05**                     | 
| FHV     | **somente 2023-05**                     | 
| FHVHV   | **somente 2023-05**                     | 

> **Nota metodológica:** quando uma métrica depende de uma coluna ausente numa frota (p.ex. `passenger_count`), a agregação considera **apenas as frotas que fornecem o campo** (Yellow/Green). Isso é explicitado nas consultas SQL.

---

## 🧩 Pré-requisitos

- Conta no **Databricks Community Edition** (cluster sem internet).
- Download **offline** prévio (feito localmente) e upload para os **Volumes**
- Python 3.10+ localmente apenas se for usar o script de download.
- O `requirements.txt` auxilia na execução local (não é necessário no Databricks).

---

## 📥 Como obter os dados (offline)

Como o Databricks CE **não tem internet**, baixe localmente e depois faça **upload** para um **Volume**. Script exemplo (ajuste anos/meses conforme sua estratégia):

```python
import os
import time
import requests

BASE = "https://d37ci6vzurychx.cloudfront.net/trip-data/{}_tripdata_{}-{}.parquet"

# Ajuste aqui os anos que deseja por categoria:
YEARS_BY_CAT = {
    "yellow": [str(y) for y in range(2020, 2026)],  # 2020–2025 (todos os meses)
    "green":  ["2023"],                              # mensal (mude se quiser mais anos)
    "fhv":    ["2023"],                              # mensal
    "fhvhv":  ["2023"],                              # mensal
}

MONTHS = [f"{m:02d}" for m in range(1, 13)]          # meses 01..12
OUT_DIR = "nyc_taxi_raw"                             # pasta de saída local
TIMEOUT = 60                                         # segundos
SLEEP_BETWEEN = 0.25                                 # pausa entre downloads (evita throttling)

os.makedirs(OUT_DIR, exist_ok=True)

def download_one(category: str, year: str, month: str) -> None:
    """
    Baixa um único arquivo mensal para a categoria/ano/mês informados.
    Usa streaming (chunks) para não carregar tudo em memória.
    Pula se o arquivo já existir.
    """
    url = BASE.format(category, year, month)
    out_path = os.path.join(OUT_DIR, f"{category}_{year}-{month}.parquet")

    if os.path.exists(out_path):
        print(f"[skip] {out_path}")
        return

    try:
        with requests.get(url, timeout=TIMEOUT, stream=True) as r:
            if r.status_code != 200:
                print(f"[miss] {url} (HTTP {r.status_code})")
                return

            with open(out_path, "wb") as f:
                for chunk in r.iter_content(chunk_size=1024 * 1024):  # 1MB
                    if chunk:
                        f.write(chunk)

        print(f"[ ok ] {url}")
    except Exception as e:
        print(f"[err ] {url} -> {e}")

def main():
    total_planned = 0
    for cat, years in YEARS_BY_CAT.items():
        for y in years:
            for m in MONTHS:
                total_planned += 1

    print(f"🔎 Planejados {total_planned} arquivos para download em '{OUT_DIR}'.")

    downloaded = 0
    for cat, years in YEARS_BY_CAT.items():
        for y in years:
            for m in MONTHS:
                download_one(cat, y, m)
                downloaded += 1
                time.sleep(SLEEP_BETWEEN)

    print(f"✅ Processo concluído. Arquivos tentados: {downloaded}.")

if __name__ == "__main__":
    main()

```

**Upload sugerido (Volumes):**
```
/Volumes/workspace/nyc_taxi/raw/yellow/*.parquet
/Volumes/workspace/nyc_taxi/raw/green/*.parquet
/Volumes/workspace/nyc_taxi/raw/fhv/*.parquet
/Volumes/workspace/nyc_taxi/raw/fhvhv/*.parquet
```

---

## ▶️ Execução no Databricks CE (passo a passo)

1. **Crie os Volumes e faça o upload** conforme a estrutura indicada (Parquets por frota/pasta).

2. **Bronze — ingestão e normalização mínima**  
   Notebook `01_ingestao_bronze.ipynb`  
   - Leitura **por categoria** (yellow/green/fhv/fhvhv) a partir dos diretórios em `/Volumes/...`.  
   - **Padronização de nomes** (lower/sanitização) e **casts** para o **schema-alvo da frota**.  
   - Derivação de **`anomes` (YYYYMM)** a partir da **coluna de pickup informada** para cada frota.  
   - Escrita em **Delta** particionada por `anomes`:
     ```
     workspace.nyc_taxi.yellow_trips_bronze
     workspace.nyc_taxi.green_trips_bronze
     workspace.nyc_taxi.fhv_trips_bronze
     workspace.nyc_taxi.fhvhv_trips_bronze
     ```

3. **Silver — limpeza simples**  
   Notebook `02_transformacao_silver.ipynb`  
   Para **cada** tabela Bronze, aplicar as mesmas regras genéricas:
   - **Remover linhas** com **colunas essenciais nulas** (ex.: `anomes`, pickup/dropoff; `vendorid` quando existir).  
   - **Remover partições** cujo volume seja **< 3%** da partição mais populosa (da **própria** tabela).  
   - **Remover registros** com **ordem temporal inválida** (`dropoff < pickup`).  
   - **Remover linhas** com `passenger_count ≤ 0` **se** a coluna existir na frota.  
   - **Logar** a quantidade de linhas removidas **por regra**.  
   - Gravar **Delta** particionada por `anomes`:
     ```
     workspace.nyc_taxi.yellow_trips_silver
     workspace.nyc_taxi.green_trips_silver
     workspace.nyc_taxi.fhv_trips_silver
     workspace.nyc_taxi.fhvhv_trips_silver
     ```

4. **Gold — modelagem analítica (último ajuste)**  
   Notebook `03_transformacao_gold.ipynb`  
   - **Gold 1 – Yellow (2023/01–2023/05)**  
     Seleciona **apenas** as colunas do abaixo e filtra o período:  
     `vendorid, passenger_count, total_amount, tpep_pickup_datetime, tpep_dropoff_datetime, anomes`  
     Tabela (Delta, particionada por `anomes`):  
     ```
     workspace.nyc_taxi.yellow_2023_jan_may_gold
     ```
   - **Gold 2 – Maio/2023 unificada (mínimo necessário)**  
     Consolida **todas as frotas** apenas para `anomes = '202305'` para calculo da média de passageiros por hora:  
     `pickup_datetime, passenger_count, anomes`  
     > `passenger_count` **não existe** em FHV/FHVHV — fica **NULL** e não afeta o `AVG`.  
     Tabela (Delta, particionada por `anomes`):  
     ```
     workspace.nyc_taxi.may_2023_min_gold
     ```

---

## 🧪 Consultas de Resposta (SQL)

**1) Média de `total_amount` por mês (Jan–Mai/2023, apenas frotas que possuem `total_amount`):**
```sql
SELECT anomes,
       ROUND(AVG(total_amount), 2) AS media_total_amount
FROM workspace.nyc_taxi.trips_2023_silver
WHERE anomes BETWEEN '202301' AND '202305'
  AND total_amount IS NOT NULL
GROUP BY anomes
ORDER BY anomes;
```

**2) Média de `passenger_count` por hora do dia em Maio/2023 (frotas que possuem `passenger_count`: Yellow/Green):**
```sql
SELECT HOUR(pickup_datetime) AS hora_do_dia,
       ROUND(AVG(passenger_count), 2) AS media_passageiros
FROM workspace.nyc_taxi.trips_2023_silver
WHERE anomes = '202305'
  AND passenger_count IS NOT NULL
GROUP BY hora_do_dia
ORDER BY hora_do_dia;
```

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

- **Delta Lake**.
- **Leitura em lote** por diretório e `unionByName` com `allowMissingColumns` para esquemas diferentes.
- **`repartition("anomes")`** antes de escrever, melhorando o layout físico das partições.

---

## 🗂️ Estrutura do Repositório

```
ifood-case/
├─ src/
│  ├─ 01_ingestao_bronze.ipynb
│  ├─ 02_transformacao_silver.ipynb
│  └─ 03_transformacao_gold.ipynb
├─ analysis/
│  └─ analysis.ipynb
├─ README.md
└─ requirements.txt
```

---

## 📌 Notas finais

- As escolhas de cobertura por frota (**2020–2025** para Yellow; **maio/2023** para Green/FHV/FHVHV) foram feitas para **equilibrar escala** e **limitações** da versão teste do Databricks
