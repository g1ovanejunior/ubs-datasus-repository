# 🏥 Projeto DataSUS — Engenharia de Dados (ETL) e Base Relacional para Georreferenciamento (CNES - MA 2024)

Este repositório documenta o pipeline completo de **Extração, Transformação e Carga (ETL)**, saneamento de microdados e modelagem relacional em **PostgreSQL** desenvolvidos para o projeto de **Georreferenciamento de Estabelecimentos de Saúde do Maranhão** (ano-base 2024).

## 🎯 Meu Papel no Projeto (Responsável pela Engenharia e Qualidade de Dados)

Dentro da arquitetura do projeto de georreferenciamento, atuei como o **responsável exclusivo pela camada de dados** (*Data Engineering & Data Quality*). Minha missão foi garantir que a equipe e as futuras aplicações espaciais (PostGIS / mapas interativos) consumissem uma base **confiável, íntegra, performática e livre de ruídos operacionais**.

Minhas principais entregas e responsabilidades incluíram:
* **Diagnóstico e Auditoria de Dados Brutos:** Mapeamento estrutural das 216 colunas originais do CNES-ST, identificando campos obsoletos, taxas de nulidade e inconsistências de cadastro.
* **Pipeline de Limpeza e Saneamento (Python/Pandas):** Desenvolvimento dos scripts de deduplicação temporal, filtragem de escopo assistencial, imputação territorial de regiões de saúde e padronização semântica (*snake_case*).
* **Modelagem e Carga no Banco de Dados (SQL/PostgreSQL):** Definição de esquema relacional, tipagem estrita de atributos, garantia de integridade referencial via chaves primárias (`PRIMARY KEY`) e ingestão dos dados tratados no PostgreSQL.

---

## 📊 Resumo da Análise Exploratória (EDA)

A análise exploratória deste projeto diagnosticou a qualidade e a estrutura dos **86.951 registros brutos (216 colunas)** do CNES do Maranhão em 2024, identificando colunas nulas, códigos despadronizados e duplicidades mensais para consolidar uma base limpa de **2.814 unidades de atenção básica** pronta para georreferenciamento e carga no banco de dados PostgreSQL.

| Métrica / Indicador | Dataset Bruto (`CNES-ST`) | Dataset Consolidado Final |
| :--- | :---: | :---: |
| **Total de Registros (Linhas)** | 86.951 (acumulado 12 meses) | **2.814** (estabelecimentos únicos) |
| **Total de Atributos (Colunas)** | 216 | **15** (atributos essenciais) |
| **Colunas 100% Nulas** | 17 colunas mortas | **0** (removidas) |
| **Nulos em `regiao_saude`** | 44,60% (bruto) / 45,64% (deduplicado) | **0,00%** (100% recuperados via IBGE) |
| **Estabelecimentos com `cpf_cnpj = 0`** | 57,20% (gestão pública/mantida) | Identificados e vinculados via `cnpj_mantenedora` |

---

## 🛠️ Pipeline de Limpeza e Tratamento de Dados (Python / Pandas)

Todo o processamento foi estruturado em etapas sequenciais e reprodutíveis utilizando **Python**, **Pandas**, **Pathlib** e **Regex (`re`)**:

### 1. Ingestão Automatizada e Diagnóstico de Nulidade
* Implementação de busca dinâmica de arquivos via `pathlib.Path.rglob()`, eliminando dependências de caminhos absolutos de sistema operacional.
* Levantamento quantitativo e percentual de valores ausentes (`isnull().sum()`) em todas as 216 colunas originais.
* Identificação e descarte de colunas 100% nulas (`COD_IR`, `ESFERA_A`, `NATUREZA`, `NIV_HIER`, `RETENCAO`, `MICR_REG`, `DISTRSAN`, entre outras) e colunas burocráticas de faturamento interno do SUS (`APxxCVyy`, `SERAPxx`, `COMISSxx`, dados bancários e alvarás com >60% de nulidade).

### 2. Seleção de Atributos e Padronização para `snake_case`
* Tradução das siglas legadas do DataSUS para nomes descritivos e padronizados em caixa baixa (`snake_case`), facilitando futuras consultas SQL sem necessidade de delimitadores:
  * `CNES` $\rightarrow$ `cnes`
  * `CODUFMUN` $\rightarrow$ `codigo_municipio_ibge`
  * `COD_CEP` $\rightarrow$ `cep`
  * `TP_UNID` $\rightarrow$ `tipo_unidade`
  * `VINC_SUS` $\rightarrow$ `vinculo_sus`
  * `TPGESTAO` $\rightarrow$ `tipo_gestao`
  * `REGSAUDE` $\rightarrow$ `regiao_saude`

### 3. Deduplicação Temporal (Consolidação Anual)
* Como os microdados do CNES são disponibilizados em recortes mensais (`COMPETEN` de `202401` a `202412`), um mesmo estabelecimento aparecia até 12 vezes na base.
* Os registros foram ordenados cronologicamente pela competência e deduplicados pela chave `cnes` (`drop_duplicates(subset=['cnes'], keep='last')`), reduzindo a base de **86.951** linhas para **7.524** estabelecimentos únicos ativos no estado em 2024.

### 4. Filtragem de Escopo para Georreferenciamento
* Para evitar poluição visual e pontos sem atendimento direto ao cidadão no mapa (como centrais administrativas, ambulâncias móveis e consultórios privados isolados), foi aplicado um filtro focado nas portas de entrada da **Atenção Primária e Especializada**:
  * `Centro de saúde / Unidade básica` (UBS)
  * `Posto de saúde`
  * `Policlínica`
* Resultado: **2.814 unidades físicas** consolidadas em todo o estado do Maranhão.

### 5. Saneamento e Imputação Territorial de `regiao_saude`
* Na base filtrada, **668 unidades (23,7%)** estavam com a coluna `regiao_saude` vazia (`NaN`), além de haver registros corrompidos ou inconsistentes (ex.: `'001'`, `'1'`, `'0001'`, `'008,'`, `'R09'` e `'2111'`).
* **Solução aplicada:**
  1. Criação de expressão regular (`re`) para sanitizar caracteres inválidos, remover prefixos estaduais (`21`) e padronizar os códigos para 2 dígitos (`01` a `18`).
  2. Construção de um dicionário territorial mapeando cada `codigo_municipio_ibge` à sua respectiva Região de Saúde (incluindo mapeamento manual de Davinópolis - IBGE `210375` $\rightarrow$ Região `11 - Imperatriz`).
  3. Criação da coluna descritiva `regiao_saude_nome`, zerando completamente os valores nulos da variável.

---

## 🗄️ Modelagem e Criação da Tabela no PostgreSQL (SQL)

Após a exportação do dataset tratado em formato CSV (`UTF-8`), estruturei o banco de dados relacional no **PostgreSQL** (via **pgAdmin 4**) aplicando tipagem estrita e restrições de integridade para preparar o ambiente para a extensão espacial **PostGIS**.

### Script DDL de Criação da Tabela

```sql
CREATE DATABASE projeto_datasus;

CREATE TABLE estabelecimentos_saude_ma (
    cnes VARCHAR(15) PRIMARY KEY,
    cpf_cnpj VARCHAR(20),
    tipo_pessoa VARCHAR(50),
    cnpj_mantenedora VARCHAR(20),
    tipo_unidade VARCHAR(100) NOT NULL,
    vinculo_sus VARCHAR(10),
    tipo_gestao VARCHAR(30),
    nivel_dependencia VARCHAR(30),
    perfil_clientela NUMERIC,
    turno_atendimento NUMERIC,
    competencia_ano_mes VARCHAR(10),
    codigo_municipio_ibge INT NOT NULL,
    regiao_saude VARCHAR(10) NOT NULL,
    cep VARCHAR(15),
    regiao_saude_nome VARCHAR(100) NOT NULL
);
```

### Decisões Técnicas na Camada SQL
* **Integridade de Entidade (`PRIMARY KEY`):** A coluna `cnes` foi definida como chave primária, garantindo no nível do SGBD que nenhum estabelecimento duplicado possa ser inserido em cargas futuras.
* **Compatibilidade de Tipagem:** Atributos categóricos numéricos que apresentavam valores ausentes na origem (`perfil_clientela` e `turno_atendimento`) foram tipados como `NUMERIC` para evitar falhas de *parsing* durante o processo de `COPY` / importação em lote.
* **Chaves Territoriais (`codigo_municipio_ibge` e `cep`):** Mantidas íntegras (`100%` preenchidas) para viabilizar o cruzamento geográfico (geocodificação por CEP/município) e a criação futura de colunas geométricas (`GEOMETRY(Point, 4326)`) com índice espacial `GIST` no PostGIS.

### Consultas de Validação Pós-Carga

```sql
-- Validação do volume total importado (2.814 registros esperados)
SELECT COUNT(*) AS total_unidades FROM estabelecimentos_saude_ma;

-- Distribuição de unidades por tipo de estabelecimento
SELECT tipo_unidade, COUNT(*) AS quantidade
FROM estabelecimentos_saude_ma
GROUP BY tipo_unidade
ORDER BY quantidade DESC;
```

---

## 💻 Tecnologias Utilizadas

* **Linguagem:** Python 3.12
* **Bibliotecas:** `pandas`, `pathlib`, `re`
* **Banco de Dados:** PostgreSQL / pgAdmin 4 (preparado para extensão **PostGIS**)
* **Fonte de Dados:** Microdados Públicos do DataSUS — CNES Estabelecimentos (`CNES-ST`, Maranhão, 2024)

---

## 🚀 Próximos Passos do Projeto

1. **Geocodificação:** Enriquecer a tabela `estabelecimentos_saude_ma` com coordenadas exatas (`latitude` e `longitude`) a partir da coluna `cep` e endereço oficial.
2. **Habilitação Espacial (PostGIS):** Ativar `CREATE EXTENSION postgis;`, converter as coordenadas em objetos espaciais (`ST_SetSRID(ST_MakePoint(longitude, latitude), 4326)`) e criar índices espaciais.
3. **Visualização e Consumo:** Conectar o banco PostgreSQL/PostGIS à camada de mapa interativo e análises de cobertura territorial.
