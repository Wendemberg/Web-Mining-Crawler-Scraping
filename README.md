# Web Scraping - Giro do Mercado 📊

Pipeline ETL completo para coleta, validação e análise de notícias financeiras com integração de dados históricos de Bitcoin.

## 📌 Visão Geral

Este projeto coleta **110 notícias** da seção **Giro do Mercado** do Money Times, realiza transformações robustas e armazena os dados em um banco analítico (DuckDB) enriquecido com **6 meses de histórico de preços do Bitcoin**.

**Stack Tecnológico:**
- 🐍 **Linguagem**: Python 3.13.7
- 🕷️ **Web Scraping**: Selenium + BeautifulSoup
- 📊 **Data Processing**: Pandas + NumPy
- 💰 **Financial Data**: yfinance (BTC-USD)
- 🗄️ **Database**: DuckDB (OLAP Analytics)
- 📓 **Environment**: Jupyter Notebook

## 🎯 Funcionalidades

### ✅ Extração de Dados
- Coleta de 110+ notícias via Selenium (headless Chrome)
- Extração de: título, URL, data/hora, lead (primeiro parágrafo)
- Suporte a paginação automática
- User-agent realista para evitar bloqueios

### ✅ Validação e Limpeza
- Validação de URLs (esquema + netloc)
- Validação de títulos (comprimento mínimo 10 caracteres)
- Validação de datas de extração
- Conversão de datas relativas para absolutas
- Remoção de prefixos desnecessários

### ✅ Deduplicação
- Hash SHA256 para identificação única
- Estratégia de chave composta (URL + título)
- Remoção automática de duplicatas
- Rastreamento de índices únicos

### ✅ Enriquecimento de Dados
- **Preços históricos do Bitcoin**: Últimos 6 meses
- **Cálculo de métricas**: Comprimento de títulos/leads, contagem de palavras
- **Dimensões temporais**: Ano, mês, dia
- **Rastreabilidade**: Hash único por notícia

### ✅ Banco Analítico
- 3 tabelas relacionadas no DuckDB
- Foreign keys para integridade referencial
- Otimizado para análise OLAP
- Exportação em memória e persistência em arquivo

## 📊 Estrutura do Banco de Dados

### TB_NOTICIAS (Tabela Principal)
```
- id_noticia: INT (PK)
- titulo: VARCHAR
- url: VARCHAR
- lead: VARCHAR
- data_publicacao: TIMESTAMP
- data_extracao: TIMESTAMP
- hash_noticia: VARCHAR
- ano, mes, dia: INT
```

### TB_METRICAS (Análise de Conteúdo)
```
- id_noticia: INT (FK)
- comprimento_titulo: INT
- comprimento_lead: INT
- num_palavras_titulo: INT
- num_palavras_lead: INT
- tem_url: BOOLEAN
```

### TB_ATIVO (Ativos + Bitcoin)
```
- id_ativo: INT (PK)
- id_noticia: INT (FK)
- titulo: VARCHAR
- url: VARCHAR
- data_publicacao: TIMESTAMP
- bitcoin: FLOAT (Preço em USD)
- ano, mes, dia: INT
```


### TB_AUDITORIA (Rastreabilidade)
```
- id_execucao: VARCHAR (PK)
- data_execucao: TIMESTAMP
- total_noticias: INT
- total_metricas: INT
- total_ativo: INT
- bitcoin_preenchidos: INT
- total_bitcoin_historico: INT
- periodo_bitcoin: VARCHAR
- origem_dados: VARCHAR
- fonte_bitcoin: VARCHAR
- status: VARCHAR
```

## 🚀 Instalação

### 1.Pré-requisitos
- Python 3.13.7+
- Google Chrome ou Chromium instalado
- Jupyter Notebook ou JupyterLab



### 2. Criar Ambiente Virtual (Recomendado)
```bash
python -m venv venv

# Windows
venv\Scripts\activate

# Linux/macOS
source venv/bin/activate
```

### 3. Instalar Dependências
```bash
pip install -r requirements.txt
```

### 4. Executar o Notebook
```bash
jupyter notebook scraping_giro_mercado.ipynb
```

## 📋 Requirements

```
python-dateutil==2.8.2
requests==2.31.0
duckdb==0.9.2
yfinance==0.2.32
numpy==1.26.2
pandas==2.1.3
lxml==4.9.3
webdriver-manager==4.0.1
beautifulsoup4==4.12.2
selenium==4.15.2
```

## 🔄 Pipeline ETL

### ETAPA 1: EXTRAÇÃO
- Acessa a URL do Money Times
- Encontra elementos HTML com class `news-item`
- Extrai título, data/hora, URL, lead
- Itera por múltiplas páginas até coletar 110 notícias

### ETAPA 2: TRANSFORMAÇÃO
- **Validação**: URL, título, data
- **Normalização**: Remove espaços extras, converte datas relativas
- **Deduplicação**: Remove duplicatas por hash
- **Enriquecimento**: Busca preços Bitcoin via yfinance

### ETAPA 3: CARGA
- Cria tabelas no DuckDB
- Estabelece relacionamentos via foreign keys
- Persiste dados em arquivo `giro_mercado.duckdb`

### ETAPA 4: QUALIDADE
- Logs em tempo real
- Estatísticas de transformação
- Rastreabilidade em tb_auditoria

## 📊 Exemplo de Uso

### Consultar Notícias com Preços Bitcoin
```python
import duckdb

conn = duckdb.connect('giro_mercado.duckdb')

# Notícias com preços Bitcoin
resultado = conn.execute('''
    SELECT
        ativo.titulo,
        ativo.data_publicacao,
        ativo.bitcoin,
        noticias.url
    FROM tb_ativo ativo
    JOIN tb_noticias noticias ON ativo.id_noticia = noticias.id_noticia
    WHERE ativo.bitcoin IS NOT NULL
    ORDER BY ativo.data_publicacao DESC
    LIMIT 10
''').fetchall()

for row in resultado:
    print(f"{row[1]}: BTC ${row[2]:,.2f}")
```

### Estatísticas de Bitcoin
```python
stats = conn.execute('''
    SELECT
        MIN(preco) as preco_minimo,
        MAX(preco) as preco_maximo,
        AVG(preco) as preco_medio,
        COUNT(*) as total_dias
    FROM tb_bitcoin_historico
''').fetchall()

print(f"Bitcoin Min: ${stats[0][0]:,.2f}")
print(f"Bitcoin Max: ${stats[0][1]:,.2f}")
print(f"Bitcoin Média: ${stats[0][2]:,.2f}")
```

### Análise de Conteúdo
```python
conteudo = conn.execute('''
    SELECT
        AVG(comprimento_titulo) as media_titulo,
        AVG(comprimento_lead) as media_lead,
        AVG(num_palavras_titulo) as media_palavras_titulo
    FROM tb_metricas
''').fetchall()
```

## 📈 Resultados Esperados

- ✅ **110 notícias** coletadas e validadas
- ✅ **0% taxa de duplicação** (deduplicação robusta)
- ✅ **88+ registros** com preços Bitcoin alinhados
- ✅ **~180 dias** de histórico de Bitcoin
- ✅ **3 tabelas** relacionadas no DuckDB
- ✅ **Rastreabilidade** completa em auditoria

## 🔍 Monitoramento

O notebook fornece logs detalhados em cada etapa:

```
[LOG] Total de registros antes da validação: 110
[VALIDAÇÃO] Verificando dados...
  ✓ URLs válidas: 110/110
  ✓ Títulos válidos: 110/110
[DEDUPLICACAO] Removidas 0 duplicatas
[BANCO] 5 tabelas criadas com sucesso
```

## 🛠️ Troubleshooting

### ChromeDriver não encontrado
```python
# O webdriver-manager baixa automaticamente
# Se houver problemas, reinstale:
pip install --upgrade webdriver-manager
```

### Timeout de conexão
```python
# Aumentar timeout em setup_driver()
WebDriverWait(driver, 10)  # 10 segundos
```

### Sem dados de Bitcoin
```python
# Verificar conexão com yfinance
import yfinance as yf
btc = yf.download('BTC-USD', period='1d')
print(btc)
```

## 📝 Documentação Adicional

- [PROJECT.md](PROJECT.md) - Documentação técnica completa
- [Documentação DuckDB](https://duckdb.org/docs/)
- [Documentação yfinance](https://pypi.org/project/yfinance/)



---

**Última atualização**: 2025-10-26
**Status**: ✅ Pipeline ETL Operacional
