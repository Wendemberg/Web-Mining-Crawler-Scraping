# Documentação Técnica - Giro do Mercado ETL

## 📋 Índice

1. [Visão Geral](#visão-geral)
2. [Arquitetura](#arquitetura)
3. [Componentes](#componentes)
4. [Fluxo de Dados](#fluxo-de-dados)
5. [Especificações Técnicas](#especificações-técnicas)
6. [Validações e Qualidade](#validações-e-qualidade)
7. [Performance](#performance)
8. [Troubleshooting](#troubleshooting)

---

## Visão Geral

O projeto **Giro do Mercado ETL** é um pipeline completo de extração, transformação e carga (ETL) que:

1. **Coleta** 110+ notícias do Money Times via web scraping
2. **Valida e normaliza** os dados usando regras rigorosas
3. **Enriquece** com dados históricos de Bitcoin (6 meses)
4. **Armazena** em banco analítico DuckDB com 5 tabelas relacionadas
5. **Audita** todas as execuções para rastreabilidade

### Objetivos Principais

- ✅ Automatizar coleta de notícias financeiras
- ✅ Garantir qualidade de dados (validação + deduplicação)
- ✅ Correlacionar eventos de mercado com preços de criptomoedasmoedas
- ✅ Fornecer base de dados pronta para análise OLAP
- ✅ Rastrear todas as transformações e execuções

---

## Arquitetura

### Diagrama de Fluxo

```
┌─────────────────────────────────────────────────────────────────┐
│                    PIPELINE ETL COMPLETO                        │
└─────────────────────────────────────────────────────────────────┘

    ETAPA 1: EXTRAÇÃO
    ┌──────────────────────────────────────────┐
    │ Selenium (Chrome Headless)               │
    │ + BeautifulSoup4                         │
    │ URL: moneytimes.com.br/tag/giro-mercado │
    │                                          │
    │ Saída: DataFrame com 110 notícias        │
    └──────────────────────────────────────────┘
                       ↓
    ETAPA 2: TRANSFORMAÇÃO
    ┌──────────────────────────────────────────┐
    │ 1. Validação (URL, Título, Data)        │
    │ 2. Normalização (trim, conversão)        │
    │ 3. Deduplicação (SHA256)                 │
    │ 4. Enriquecimento (Bitcoin via yfinance) │
    │                                          │
    │ Saída: DataFrame limpo e validado        │
    └──────────────────────────────────────────┘
                       ↓
    ETAPA 3: CARGA (DuckDB)
    ┌──────────────────────────────────────────┐
    │ Criação de 5 tabelas relacionadas:       │
    │ • tb_noticias                            │
    │ • tb_metricas                            │
    │ • tb_ativo                               │                 │
    │ • tb_auditoria                           │
    │                                          │
    │ Arquivo: giro_mercado.duckdb             │
    └──────────────────────────────────────────┘
                       ↓
    ETAPA 4: ANÁLISE
    ┌──────────────────────────────────────────┐
    │ SQL Queries via DuckDB                   │
    │ • Análise de conteúdo                    │
    │ • Correlação Bitcoin-notícias             │
    │ • Estatísticas de mercado                │
    │                                          │
    │ Saída: Insights e análises               │
    └──────────────────────────────────────────┘
```

### Stack Tecnológico

```
┌─────────────────────┐
│   Jupyter Notebook  │  Interface interativa
└──────────┬──────────┘
           │
    ┌──────┴──────┐
    │              │
    ▼              ▼
┌─────────┐   ┌──────────┐
│ Selenium│   │BeautifulSoup
│(Chrome) │   │(HTML Parse)
└─────────┘   └──────────┘
    │              │
    └──────┬───────┘
           │ Raw Data
           ▼
    ┌─────────────┐
    │  Pandas     │ Transformação
    │  NumPy      │ e Limpeza
    └─────────────┘
           │
           ▼
    ┌─────────────┐
    │  yfinance   │ Dados Bitcoin
    └─────────────┘
           │
    ┌──────┴──────┐
    │ Dados Limpo │
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │  DuckDB     │ Banco Analítico
    └─────────────┘
           │
           ▼
    ┌──────────────────┐
    │giro_mercado.duckdb│ Persistência
    └──────────────────┘
```

---

## Componentes

### 1. Web Scraper (Selenium + BeautifulSoup)

**Responsabilidades:**
- Inicializar Chrome em modo headless
- Navegar por múltiplas páginas
- Extrair notícias via seletores CSS
- Implementar retry automático

**Configurações Chrome:**
```python
--headless           # Sem interface gráfica
--no-sandbox         # Para ambientes restritos
--disable-gpu        # Melhor performance
--disable-dev-shm-usage  # Evitar problemas de memória
--window-size=1920,1080  # Tamanho padrão
```

**Seletores utilizados:**
```css
.news-item              /* Contêiner da notícia */
.news-item__title      /* Título */
.news-item__content    /* Lead/Resumo */
.date                  /* Data/Hora */
```

### 2. Validação de Dados

**Validações implementadas:**

| Tipo | Regra | Descrição |
|------|-------|-----------|
| URL | `urlparse(url)` | Deve ter scheme + netloc válidos |
| Título | `len(titulo) >= 10` | Mínimo 10 caracteres |
| Data | `datetime.strptime()` | Formato ISO 8601 |
| Lead | `len(lead) > 0` | Não pode ser vazio |

**Exemplo de código:**
```python
def validar_url(url):
    if not url:
        return False
    try:
        resultado = urlparse(url)
        return all([resultado.scheme, resultado.netloc])
    except:
        return False

def validar_titulo(titulo):
    return isinstance(titulo, str) and len(titulo.strip()) >= 10
```

### 3. Normalização de Dados

**Operações realizadas:**

```python
# Título: remover espaços extras
titulo = ' '.join(titulo.split())

# URL: remover fragments
url = url.split('#')[0].strip()

# Lead: remover prefixo "Giro do Mercado"
lead = re.sub(r'^Giro\s+do\s+Mercado\s*', '', lead)

# Data: converter formato relativo → absoluto
# "5 dias atrás" → "2025-10-21 14:30:00"
```

### 4. Deduplicação com Hash

**Estratégia de 3 camadas:**

```python
# Camada 1: Hash SHA256 (URL + Título)
def gerar_hash_noticia(url, titulo):
    chave_composta = f"{url}|{titulo}".lower()
    return hashlib.sha256(chave_composta.encode()).hexdigest()

# Camada 2: URL como Primary Key
df_validado = df_validado.drop_duplicates(subset=['url'], keep='first')

# Camada 3: Hash Composto
df_validado = df_validado.drop_duplicates(subset=['hash_noticia'], keep='first')
```

**Resultado:** 0% de duplicação em 110 registros

### 5. Enriquecimento com Bitcoin

**Processo:**

```python
# 1. Determinar período (6 meses)
data_fim = datetime.now()
data_inicio = data_fim - timedelta(days=180)

# 2. Buscar histórico via yfinance
btc_data = yf.download('BTC-USD', start=data_inicio, end=data_fim)

# 3. Mapear preços por data
btc_prices = {data: preco for data, preco in ...}

# 4. Alinha com datas das notícias
for noticia in noticias:
    bitcoin_price = btc_prices.get(noticia['data_publicacao'])
```

**Resultado:**
- 88+ registros com preços Bitcoin alinhados
- ~180 dias de histórico completo
- Preços em USD (BTC-USD)

### 6. DuckDB (Banco Analítico)

**Características:**

```
┌──────────────────────────┐
│      DuckDB Features     │
├──────────────────────────┤
│ • OLAP (Analytical DB)   │
│ • In-Memory + Persistent │
│ • SQL completo           │
│ • Joins otimizados       │
│ • Agregações rápidas     │
│ • Sem servidor           │
└──────────────────────────┘
```

**Vantagens:**

- ✅ Sem configuração de servidor
- ✅ Excelente para análise ad-hoc
- ✅ Suporta múltiplos formatos (Parquet, CSV, JSON)
- ✅ Performance superior para queries analíticas
- ✅ Persistência em arquivo único

---

## Fluxo de Dados

### 1. Extração (Selenium)

```
Input: URL da página
       ↓
  Navegar com Chrome
       ↓
  Parse HTML com BeautifulSoup
       ↓
  Extrair 10 notícias por página
       ↓
  Iterar até 110 notícias
       ↓
Output: DataFrame 110x5
        (titulo, data_hora, url, lead, data_extracao)
```

**Lógica de iteração:**

```python
while len(todas_noticias) < 110:
    # Construir URL com paginação
    url = f'https://moneytimes.com.br/tag/giro-mercado/page/{pagina}/'

    # Extrair notícias da página
    noticias_pagina = extrair_noticias_pagina(driver, url)

    # Adicionar se não forem duplicatas
    for noticia in noticias_pagina:
        if noticia['url'] not in urls_existentes:
            todas_noticias.append(noticia)

    pagina += 1
    time.sleep(2)  # Rate limiting
```

### 2. Transformação (Pandas)

```
Input: df_noticias (110x5)
       ↓
  VALIDAÇÃO
  ├─ URL válida? (110/110 ✓)
  ├─ Título válido? (110/110 ✓)
  └─ Data válida? (110/110 ✓)
       ↓
  NORMALIZAÇÃO
  ├─ Títulos (trim, espaços extras)
  ├─ URLs (remover fragments)
  ├─ Leads (remover prefixos)
  └─ Datas (relativas → absolutas)
       ↓
  DEDUPLICAÇÃO
  ├─ Hash SHA256
  └─ Resultado: 110 registros únicos (0 duplicatas)
       ↓
  ENRIQUECIMENTO
  ├─ Fetch Bitcoin histórico (yfinance)
  ├─ Calcular métricas (comprimento, palavras)
  └─ Adicionar dimensões (ano, mês, dia)
       ↓
Output: df_validado (110x10+)
        Com dados prontos para banco
```

### 3. Carga (DuckDB)

```
Input: DataFrames transformados
       ↓
  CREATE TABLE tb_noticias
  └─ 110 notícias com hash
       ↓
  CREATE TABLE tb_metricas
  └─ 110 métricas de conteúdo
       ↓
  CREATE TABLE tb_ativo
  └─ 110 ativos com preços Bitcoin
       ↓
  CREATE TABLE tb_auditoria
  └─ 1 registro de execução
       ↓
Output: giro_mercado.duckdb
        3 tabelas relacionadas, prontas para análise
```

### 4. Análise (SQL)

```sql
-- Exemplo 1: Notícias com Bitcoin
SELECT
    noticias.titulo,
    ativo.bitcoin,
    bitcoin_hist.preco as preco_anterior
FROM tb_noticias noticias
JOIN tb_ativo ativo ON noticias.id_noticia = ativo.id_noticia
JOIN tb_bitcoin_historico bitcoin_hist
    ON DATE(ativo.data_publicacao) = bitcoin_hist.data
WHERE ativo.bitcoin IS NOT NULL
ORDER BY ativo.data_publicacao DESC;

-- Exemplo 2: Estatísticas Bitcoin
SELECT
    MIN(preco) as minimo,
    MAX(preco) as maximo,
    AVG(preco) as media,
    STDDEV(preco) as desvio_padrao
FROM tb_bitcoin_historico;

-- Exemplo 3: Análise de conteúdo
SELECT
    AVG(comprimento_titulo) as media_titulo,
    AVG(comprimento_lead) as media_lead,
    AVG(num_palavras_titulo) as media_palavras
FROM tb_metricas;
```

---

## Especificações Técnicas

### Ambiente de Execução

```yaml
Python: 3.13.7+
Sistema: Windows, Linux, macOS
Memória: 512MB mínimo, 2GB recomendado
Disco: 50MB livre
Navegador: Chrome/Chromium instalado
```

### Dependências Principais

```
selenium==4.15.2              # WebDriver
beautifulsoup4==4.12.2        # HTML parsing
webdriver-manager==4.0.1      # ChromeDriver automático
pandas==2.1.3                 # Data manipulation
numpy==1.26.2                 # Numerical computing
duckdb==0.9.2                 # Analytic database
yfinance==0.2.32              # Financial data
requests==2.31.0              # HTTP requests
lxml==4.9.3                   # XML/HTML processing
python-dateutil==2.8.2        # Date utilities
```

### Performance

**Tempos de execução esperados:**

| Etapa | Tempo | Descrição |
|-------|-------|-----------|
| Extração | 3-5 min | Scraping 110 notícias (11 páginas) |
| Transformação | 10-15 seg | Validação + normalização |
| Deduplicação | 5-10 seg | Hash + remoção duplicatas |
| Enriquecimento Bitcoin | 10-20 seg | yfinance + merge |
| Carga DuckDB | 5 seg | Criação 5 tabelas |
| **TOTAL** | **~4-6 min** | **Pipeline completo** |

**Tamanho de dados:**

```
giro_mercado.duckdb: ~150-200 KB
Raw notícias: ~2 MB (antes limpeza)
Limpo: ~500 KB (após transformação)
Bitcoin histórico: ~15 KB (180 dias)
```

### Limites e Constraints

```sql
-- tb_noticias
ALTER TABLE tb_noticias ADD CONSTRAINT pk_noticias
    PRIMARY KEY (id_noticia);
ALTER TABLE tb_noticias ADD CONSTRAINT uk_url
    UNIQUE (url);
ALTER TABLE tb_noticias ADD CONSTRAINT uk_hash
    UNIQUE (hash_noticia);

-- tb_ativo
ALTER TABLE tb_ativo ADD CONSTRAINT fk_noticia
    FOREIGN KEY (id_noticia) REFERENCES tb_noticias(id_noticia);

-- tb_metricas
ALTER TABLE tb_metricas ADD CONSTRAINT fk_metrica
    FOREIGN KEY (id_noticia) REFERENCES tb_noticias(id_noticia);
```

---

## Validações e Qualidade

### Matriz de Validação

```
┌────────────────┬─────────┬──────────┐
│ Campo          │ Regra   │ Resultado│
├────────────────┼─────────┼──────────┤
│ URL            │ Valid   │ 110/110  │
│ Título         │ >10 chr │ 110/110  │
│ Data Extra     │ Valid   │ 110/110  │
│ Lead           │ >0 chr  │ 110/110  │
│ Duplicatas     │ 0       │ 0/110    │
│ Bitcoin        │ Algum   │ 88/110   │
└────────────────┴─────────┴──────────┘

Taxa de qualidade: 100%
```

### Logs de Validação

Cada execução produz logs detalhados:

```log
2025-10-26 14:30:45 - INFO - Iniciando validação de 110 registros
[VALIDAÇÃO] Verificando dados...
  ✓ URLs válidas: 110/110 (100.0%)
  ✓ Títulos válidos: 110/110 (100.0%)
  ✓ Datas de extração válidas: 110/110 (100.0%)
[NORMALIZAÇÃO] Normalizando campos...
  ✓ Títulos normalizados
  ✓ URLs normalizadas
  ✓ Leads normalizados
  ✓ Datas convertidas (relativas → absolutas)
[DEDUPLICACAO] Removidas 0 duplicatas
[STATUS] Pipeline: SUCESSO ✓
```

### Rastreabilidade

A tabela `tb_auditoria` registra:

```python
{
    'id_execucao': '5a3f2e8c1b9d4a7f',  # Hash único
    'data_execucao': '2025-10-26 14:30:45',
    'total_noticias': 110,
    'total_metricas': 110,
    'total_ativo': 110,
    'bitcoin_preenchidos': 88,
    'total_bitcoin_historico': 180,
    'periodo_bitcoin': '2025-04-28 a 2025-10-26',
    'origem_dados': 'Money Times - Giro do Mercado',
    'fonte_bitcoin': 'yfinance (BTC-USD) - Últimos 6 meses',
    'status': 'SUCESSO'
}
```

---

## Performance

### Otimizações Implementadas

1. **Selenium:**
   - Chrome headless (sem UI)
   - Desabilitar extensões desnecessárias
   - Rate limiting com `time.sleep(2)`
   - Reutilizar driver entre páginas

2. **Pandas:**
   - Operações vetorizadas com `apply()`
   - Evitar loops innecessários
   - Usar `copy()` seletivamente

3. **DuckDB:**
   - Criar índices em FK
   - Usar tipos de dados apropriados (INT, FLOAT)
   - Preparar statements para queries repetitivas

4. **Memory:**
   - Liberar memoria do driver: `driver.quit()`
   - Usar garbage collection: `gc.collect()`

### Benchmark

```
Extração (110 notícias):      4min 15seg
Transformação (validation):   12seg
Deduplicação (hash):          8seg
Enriquecimento (Bitcoin):     15seg
Carga (DuckDB):               4seg
───────────────────────────
TOTAL:                        ~5min
```

**Throughput:**
- 22 notícias/min (extração)
- 9 mil registros/seg (transformação)

---

## Troubleshooting

### Problema: ChromeDriver não encontrado

**Causa:** webdriver-manager não baixou o driver

**Solução:**
```python
from webdriver_manager.chrome import ChromeDriverManager
from selenium.webdriver.chrome.service import Service

service = Service(ChromeDriverManager().install())
driver = webdriver.Chrome(service=service)
```

### Problema: Timeout na conexão

**Causa:** Página lenta ou bloqueio de IP

**Solução:**
```python
# Aumentar timeout
driver.set_page_load_timeout(30)
WebDriverWait(driver, 15).until(EC.presence_of_all_elements_located(...))

# Implementar retry
for attempt in range(3):
    try:
        driver.get(url)
        break
    except TimeoutException:
        if attempt == 2:
            raise
        time.sleep(5)
```

### Problema: Sem dados de Bitcoin

**Causa:** yfinance retornou vazio ou erro de conexão

**Solução:**
```python
# Testar conexão
import yfinance as yf
btc = yf.download('BTC-USD', period='1d')
print(btc)

# Usar fallback de datas
try:
    btc_data = yf.download(...)
except Exception as e:
    logger.warning(f"Bitcoin fetch falhou: {e}")
    btc_prices = {}  # Continuar sem Bitcoin
```

### Problema: DuckDB não persiste

**Causa:** Conexão não foi fechada corretamente

**Solução:**
```python
import duckdb

try:
    conn = duckdb.connect('giro_mercado.duckdb')
    conn.execute('CREATE TABLE ...')
finally:
    conn.close()  # Sempre fechar para persist

# Verificar arquivo
import os
print(os.path.getsize('giro_mercado.duckdb'))
```

### Problema: Muitas duplicatas removidas

**Causa:** URLs com parâmetros de tracking ou IDs de sessão

**Solução:**
```python
from urllib.parse import urlparse, parse_qs, urlencode

def normalizar_url(url):
    parsed = urlparse(url)
    # Remover parâmetros de tracking
    params = parse_qs(parsed.query)
    params.pop('utm_source', None)
    params.pop('utm_medium', None)
    params.pop('utm_campaign', None)

    nova_query = urlencode(params, doseq=True)
    return urlunparse((parsed.scheme, parsed.netloc, parsed.path,
                       parsed.params, nova_query, ''))
```

## Referências

- [Selenium Documentation](https://www.selenium.dev/documentation/)
- [BeautifulSoup](https://www.crummy.com/software/BeautifulSoup/)
- [DuckDB Docs](https://duckdb.org/docs/)
- [yfinance](https://pypi.org/project/yfinance/)
- [Pandas User Guide](https://pandas.pydata.org/docs/)

---

