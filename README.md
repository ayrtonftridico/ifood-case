# 🚖 iFood Data Architecture Case – NYC Yellow Taxi Data

Este projeto implementa um pipeline de ingestão, normalização e análise de dados abertos da **NYC Taxi & Limousine Commission (TLC)**, utilizando **PySpark** e **Delta Lake** no **Databricks**.  
O objetivo é consolidar os dados de corridas de **Yellow Taxi** e responder às perguntas solicitadas no case para o período de **janeiro a maio de 2023**.

---

## 🎯 Objetivos do Case

1. **Download e Ingestão:** coletar dados históricos da NYC TLC no formato Parquet.
2. **Normalização:** padronizar schema e garantir consistência de tipos de dados.
3. **Particionamento:** criar coluna `anomes` baseada na data real de pickup (`tpep_pickup_datetime`).
4. **Camadas de Dados:** 
   - **Bronze (SOR):** todos os dados históricos (2020–2025).
   - **Silver:** apenas os meses e colunas de interesse do case.
5. **Análises:** responder às perguntas de negócio (média de valores e de passageiros).
6. **Visualização:** gerar gráficos que mostrem tendências de forma intuitiva.

---

## 📦 Dependências

Instale as dependências localmente se não estiver usando Databricks:

```bash
pip install -r requirements.txt
```

Conteúdo do `requirements.txt`:

```txt
pyspark>=3.4.0    # Processamento distribuído com Spark
pandas>=1.5.0     # Conversão de DataFrame Spark para Pandas
matplotlib>=3.7.0 # Visualização de dados (gráficos)
requests>=2.31.0  # Download dos arquivos Parquet
```

> ⚠️ Em ambientes Databricks, `pyspark` e `pandas` já vêm instalados.  
> O pacote `requests` é utilizado apenas se for fazer o download dos arquivos localmente.

---

## 📥 Download dos Arquivos

Como o Databricks Community Edition **não possui acesso à internet**, o download dos arquivos deve ser feito **localmente**.  

Você pode utilizar o seguinte script Python para baixar os Parquet diretamente da NYC TLC:

```python
import os
import requests
from concurrent.futures import ThreadPoolExecutor, as_completed

BASE_URL = "https://d37ci6vzurychx.cloudfront.net/trip-data/{}_tripdata_{}-{}.parquet"
CATEGORIES = ["yellow"]
YEARS = ["2023"]
MONTHS = [f"{i:02d}" for i in range(1, 6)]

OUTPUT_DIR = "nyc_taxi_raw"
os.makedirs(OUTPUT_DIR, exist_ok=True)

def download_file(cat, year, month):
    url = BASE_URL.format(cat, year, month)
    filename = os.path.join(OUTPUT_DIR, f"{cat}_{year}-{month}.parquet")
    if os.path.exists(filename):
        return f"⏩ Já existe: {filename}"
    try:
        r = requests.get(url, timeout=60)
        if r.status_code == 200:
            with open(filename, "wb") as f:
                f.write(r.content)
            return f"✅ Baixado: {url}"
        return f"❌ Não encontrado: {url}"
    except Exception as e:
        return f"⚠️ Erro ao baixar {url}: {e}"

with ThreadPoolExecutor(max_workers=8) as executor:
    futures = [executor.submit(download_file, cat, year, month)
               for cat in CATEGORIES
               for year in YEARS
               for month in MONTHS]
    for future in as_completed(futures):
        print(future.result())
```

Após o download, faça o upload dos arquivos para um **Volume do Databricks** (`/Volumes/.../nyc_taxi/raw/2023/`) ou para uma pasta no **DBFS**.

---

## 🏗️ Pipeline de Transformação

### 1️⃣ Camada Bronze (SOR)
Notebook: `01_ingestao_bronze.py`

- Lê todos os arquivos Parquet (2020–2025).
- Padroniza schema e tipos de dados.
- Cria coluna de partição `anomes` (formato YYYYMM).
- Filtra partições com menos de 10.000 linhas (outliers de volume).
- Salva resultado em **Delta Table**:

```text
workspace.nyc_taxi.yellowtaxi_trips_sor
```

---

### 2️⃣ Camada Silver
Notebook: `02_transformacao_silver.py`

- Seleciona apenas as colunas relevantes:
  - `vendorid`, `passenger_count`, `total_amount`, `tpep_pickup_datetime`, `tpep_dropoff_datetime`, `anomes`
- Filtra dados de **2023-01 a 2023-05**.
- Salva em uma nova Delta Table particionada por `anomes`:

```text
workspace.nyc_taxi.yellowtaxi_trips_2023_silver
```

---

### 3️⃣ Análises
Notebook: `03_analises.py`

- **Média de valor total por mês:**
```sql
SELECT anomes, ROUND(AVG(total_amount), 2) AS media_total_amount
FROM workspace.nyc_taxi.yellowtaxi_trips_2023_silver
GROUP BY anomes
ORDER BY anomes;
```

- **Média de passageiros por hora do dia (Maio/2023):**
```sql
SELECT HOUR(tpep_pickup_datetime) AS hora_do_dia,
       ROUND(AVG(passenger_count), 2) AS media_passageiros
FROM workspace.nyc_taxi.yellowtaxi_trips_2023_silver
WHERE anomes = '202305'
GROUP BY hora_do_dia
ORDER BY hora_do_dia;
```

---

### 4️⃣ Visualização (PLUS)
Notebook: `04_visualizacoes.py`

Cria gráfico de linha mostrando a média de passageiros por hora do dia para maio de 2023:

```python
plt.figure(figsize=(10,5))
plt.plot(pdf["hora_do_dia"], pdf["media_passageiros"], marker="o")
plt.title("Média de Passageiros por Hora - Maio/2023")
plt.xlabel("Hora do Dia")
plt.ylabel("Média de Passageiros")
plt.grid(True)
plt.xticks(range(0, 24))
plt.show()
```

---

## 📊 Resultado Esperado

- **Camada Bronze:** consolidada e limpa, com milhões de registros.
- **Camada Silver:** apenas os 5 meses de interesse, com as colunas necessárias para análise.
- **Consultas SQL:** métricas de média de faturamento e demanda horária.
- **Gráficos:** insights visuais mostrando variação da demanda ao longo do dia.
