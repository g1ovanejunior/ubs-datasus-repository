
# 🏥 Projeto DataSUS — Pipeline ETL e Tratamento de Dados do CNES (Maranhão - 2024)

Este repositório contém o pipeline completo de **Engenharia e Tratamento de Dados (ETL — *Extract, Transform, Load*)** aplicado aos microdados públicos do **Cadastro Nacional de Estabelecimentos de Saúde (CNES)**, disponibilizados pelo **DataSUS** (Ministério da Saúde).

O objetivo deste projeto é extrair, higienizar, enriquecer e estruturar os dados cadastrais e geográficos das unidades de saúde do estado do **Maranhão (MA)** referentes ao ano de **2024**, preparando uma base confiável e otimizada para ingestão em banco de dados relacional/espacial (**PostgreSQL / PostGIS**) e posterior consumo em aplicações de georreferenciamento (como um mapa interativo de Unidades Básicas de Saúde).

---

## 🛠️ Tecnologias e Ferramentas Utilizadas

* **Extração (R):** Linguagem **R** com o pacote [`microdatasus`](https://github.com/rfsaldanha/microdatasus) (consumo direto dos arquivos `.DBC` via FTP do DataSUS) e `readr`.
* **Transformação e Análise (Python):** Linguagem **Python** utilizando `pandas`, `numpy`, `pathlib` e `re` (Expressões Regulares / Regex) em ambiente **Jupyter Notebook** (`.ipynb`) e scripts `.py`.
* **Armazenamento Alvo:** **PostgreSQL** com extensão espacial **PostGIS**.
* **Versionamento:** **Git** e **GitHub**.

---

## 📊 Panorama Geral dos Dados (Antes vs. Depois)

| Etapa do Pipeline | Quantidade de Registros (Linhas) | Quantidade de Atributos (Colunas) | Descrição do Estado da Base |
| :--- | :---: | :---: | :--- |
| **1. Base Bruta Extraída (12 meses)** | `86.951` | `216` | Dados acumulados de Jan/2024 a Dez/2024 (`CNES-ST`) com siglas legadas e colunas vazias. |
| **2. Seleção e Renomeação** | `86.951` | `24` | Remoção de 17 colunas 100% nulas e 175 colunas burocráticas/operacionais sem uso geográfico. |
| **3. Deduplicação Anual** | `7.524` | `24` | Consolidação de estabelecimentos únicos no Maranhão, mantendo o cadastro mais recente. |
| **4. Filtragem de Escopo (UBS/Policlínicas)** | `2.814` | `24` | Retenção exclusiva de unidades de Atenção Básica e Policlínicas. |
| **5. Imputação Territorial (`regiao_saude`)** | `2.814` | `25` | Padronização de códigos e eliminação de **100% dos valores nulos** via cruzamento IBGE. |

---

## 🔄 Passo a Passo do Pipeline de Tratamento de Dados (ETL)

### Etapa 1: Extração dos Microdados (`Extract`)
Os dados brutos do módulo **CNES-ST (Estabelecimentos)** foram extraídos diretamente dos servidores do DataSUS utilizando o pacote `microdatasus` em R:
* **Escopo:** Estado do Maranhão (`uf = "MA"`), abrangendo todas as 12 competências mensais de 2024 (`month_start = 1` a `month_end = 12`).
* **Pré-processamento nativo:** A função `process_cnes()` realizou a descompressão dos arquivos `.DBC`, decodificação de tabelas auxiliares e enriquecimento inicial com metadados municipais (latitude, longitude, altitude e área do município).
* **Saída gerada:** `cnes_estabelecimentos_MA_2024_completo.csv` (86.951 linhas × 216 colunas).

---

### Etapa 2: Diagnóstico e *Data Profiling* (Análise Exploratória)
Antes de alterar qualquer dado, foi executado um diagnóstico estrutural (*Data Profiling*) no Python para mapear tipos de dados (`dtypes`), cardinalidade e taxas de valores ausentes (*missing values* / `NaN`):
1. **Colunas Totalmente Mortas (100% Nulas):** Foram identificadas **17 colunas** sem nenhum valor preenchido em todo o dataset (ex.: `COD_IR`, `MICR_REG`, `DISTRSAN`, `ESFERA_A`, `RETENCAO`, `NATUREZA`, `NIV_HIER`, `AV_ACRED`, `AV_PNASS`, entre outras).
2. **Colunas de Ruído Operacional / Faturamento:** Cerca de 175 colunas continham matrizes internas de faturamento e convênios (`AP01CV01` a `AP07CV07`), comissões hospitalares (`COMISS01` a `COMISS12`) e contagens granulares de instalações (`QTINSTxx`), dispensáveis para o mapeamento voltado ao cidadão.
3. **Anomalias de Codificação e Preenchimento:** Identificou-se que a coluna `REGSAUDE` apresentava **45,64% de valores nulos** e inconsistências de formatação (ex.: `'001'`, `'0001'`, `'1'`, `'2111'`).

---

### Etapa 3: *Feature Selection* e Padronização (`snake_case`)
Para otimizar o armazenamento e facilitar consultas SQL no PostgreSQL, realizou-se a seleção de atributos (*Feature Selection*) e o mapeamento semântico das colunas:
* **Tradução de Siglas Legadas:** Colunas originais do DataSUS foram renomeadas para termos autoexplicativos (ex.: `CODUFMUN` $\rightarrow$ `codigo_municipio_ibge`, `TP_UNID` $\rightarrow$ `tipo_unidade`, `munResLat` $\rightarrow$ `municipio_latitude`).
* **Padrão `snake_case`:** Todos os identificadores foram convertidos para letras minúsculas com palavras separadas por sublinhado (`_`), evitando problemas de *case-sensitivity* e necessidade de aspas duplas em queries SQL.

---

### Etapa 4: Deduplicação Temporal (`Deduplication`)
Como o DataSUS publica arquivos mensais contendo a fotografia completa da rede de saúde a cada mês, cada estabelecimento ativo aparecia até 12 vezes no dataset acumulado.
* **Ordenação Cronológica:** Os registros foram ordenados pela coluna `competencia_ano_mes` (`COMPETEN`, formato `YYYYMM` de `202401` a `202412`).
* **Consolidação:** Aplicou-se `drop_duplicates(subset=['cnes'], keep='last')`, reduzindo a base de **86.951 registros brutos** para **7.524 estabelecimentos únicos** no Maranhão, preservando sempre a atualização cadastral mais recente do ano.

---

### Etapa 5: Filtragem de Domínio (Escopo da Aplicação)
Com foco na Atenção Primária e atendimento ambulatorial público, aplicou-se um filtro categórico na coluna `tipo_unidade`, selecionando exclusivamente:
* `Centro de saúde / Unidade básica`
* `Posto de saúde`
* `Policlínica`

Essa etapa filtrou a base de 7.524 estabelecimentos gerais para **2.814 unidades** diretamente alinhadas ao objetivo da aplicação.

---

### Etapa 6: Higienização via Regex e Imputação Territorial (`regiao_saude`)
A coluna `regiao_saude` (`REGSAUDE`) possuía **45,64% de registros nulos** e códigos despadronizados. Para recuperar 100% dessa informação sem inventar dados:
1. **Normalização com Expressões Regulares (`re`):** Extraíram-se apenas os dígitos numéricos, removendo caracteres inválidos e o prefixo estadual `21` quando cadastrado incorretamente (ex.: `2111` $\rightarrow$ `11`), padronizando todas as **18 Regiões de Saúde do Maranhão** com dois dígitos (`'01'` a `'18'`).
2. **Imputação Determinística por Município (Moda):** Como a divisão regional de saúde é territorialmente fixa por município, criou-se um dicionário mapeando cada `codigo_municipio_ibge` à sua respectiva `regiao_saude` predominante na base. O único município sem registro preenchido (`210375` — Davinópolis) foi mapeado manualmente para a Região `11` (Imperatriz).
3. **Enriquecimento Semântico:** Criou-se a coluna `regiao_saude_nome` traduzindo cada código numérico para sua nomenclatura oficial (ex.: `01 - São Luís`, `11 - Imperatriz`), atingindo **0% de valores nulos**.

---

## 📁 Estrutura do Repositório

```text
projeto-datasus/
├── main/
│   ├── extração_cnes.R      # Script em R para download e extração via microdatasus
│   └── main.ipynb           # Jupyter Notebook com todo o pipeline de limpeza e análise
├── .gitignore               # Ignora arquivos CSV pesados (>50MB) e ambientes virtuais (.venv)
└── README.md                # Documentação técnica do projeto
```

> **Nota sobre os dados:** Os arquivos `.csv` brutos (~80 MB) são omitidos do versionamento via `.gitignore` para respeitar as boas práticas e limites de tamanho de repositório do GitHub, podendo ser reproduzidos integralmente executando os scripts contidos neste projeto.
