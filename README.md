# forjador_de_dados

## O que é o forjador_de_dados?

O `forjador_de_dados` é um projeto de Engenharia de Dados de ponta a ponta, concebido para simular um fluxo completo de dados — desde a geração das informações até a disponibilização de análises preditivas e insights automatizados — aproximando-se de cenários encontrados em ambientes corporativos reais.

O projeto não tem como objetivo ser um produto comercial, mas sim demonstrar, na prática, como diferentes componentes de um ecossistema analítico se conectam: geração de dados, armazenamento, ETL, cálculo de indicadores, machine learning, exposição via API e consumo por ferramentas externas (BI, dashboards e LLMs).

---

## Visão geral da arquitetura

O pipeline é dividido em seis grandes camadas, cada uma com uma responsabilidade única e desacoplada das demais:

```text
1. Data Seeding        → geração de dados fictícios
2. PostgreSQL          → armazenamento relacional
3. ETL (Pandas)        → extração, limpeza e transformação
4. KPIs & ML           → métricas comerciais e previsões
5. Django REST         → camada de exposição (API)
6. Consumidores        → Power BI, Streamlit, LLM, relatórios
```

Cada camada só conhece a interface da camada anterior — isso significa que o banco de dados pode ser trocado, o motor de ETL pode evoluir, e o modelo de ML pode ser retreinado sem que nada quebre nas camadas de exposição.

### Fluxo detalhado

```text
             Data Seeding
                    |
            Dados Fictícios
                    |
               PostgreSQL
                    |
                 SQL
                    |
               ETL/ELT
                    |
                 Pandas
                    |
             Transformações
                    |
                JSON
                    |
          ---------------------
          |                   |
         KPIs                ML (previsão)
          |                   |
          ---------------------
                    |
              Django REST
                    |
            Aplicações Externas
                    |
       --------------------------------
       |               |              |
    Power BI       Streamlit          LLM
       |               |              |
  Dashboards      Visualização     Insights
       --------------------------------
                    |
                 Relatórios Automáticos
```

---

## 1. Data Seeding

Módulo responsável por gerar automaticamente milhares de registros fictícios, simulando cenários encontrados em ambientes corporativos. Seu principal objetivo é fornecer dados consistentes para análise, treinamento de modelos de Machine Learning e construção de pipelines de Engenharia de Dados.

Os dados são gerados respeitando os relacionamentos entre as tabelas do banco de dados, permitindo simular operações reais de um sistema comercial (por exemplo, um pedido sempre pertence a um cliente e a um produto existentes).

### Dados gerados

* Clientes
* Produtos
* Categorias
* Pedidos
* Vendas
* Estoque
* Métodos de pagamento
* Cupons de desconto
* Regiões comerciais
* Datas das transações
* Avaliações dos clientes

### Estrutura simulada

```text
Clientes:
- Nome
- Idade
- Cidade
- Estado
- Data de cadastro

Produtos:
- Nome
- Categoria
- Preço
- Quantidade em estoque

Pedidos:
- Cliente
- Produto
- Quantidade
- Data da compra
- Valor total

Pagamentos:
- Cartão
- PIX
- Boleto
- Transferência bancária

Vendas:
- Receita
- Região
- Produto vendido
- Horário da venda
- Ticket médio
```

---

## 2. Processo de ETL

Após a geração dos dados, o pipeline realiza a extração das informações armazenadas no PostgreSQL para posterior processamento.

### Extração

* Consultas SQL diretas ao PostgreSQL (via `SQLAlchemy` ou `psycopg2`)
* Conversão do resultado em `DataFrame` (`pandas.read_sql`)
* Organização dos dados em memória para as próximas etapas

### Transformação (Pandas)

O Pandas é o núcleo dessa camada, responsável por:

* Limpeza dos dados (remoção de duplicatas, tratamento de outliers)
* Tratamento de valores nulos (`fillna`, `dropna`)
* Conversão de tipos de dados (datas, valores monetários)
* Remoção de inconsistências (ex: pedidos sem cliente vinculado)
* Criação de novas variáveis (ex: ticket médio, margem de lucro)
* Cálculo de métricas comerciais
* Agrupamentos estatísticos (`groupby`, `resample` para séries temporais)

### Disponibilização

Após o processamento, os dados ficam prontos para serem consumidos por:

* APIs REST
* Dashboards
* Ferramentas de Business Intelligence
* Modelos de Machine Learning
* Sistemas externos

---

## 3. KPIs Comerciais

Indicadores calculados automaticamente pelo sistema a partir dos dados transformados:

* Faturamento total
* Ticket médio
* Crescimento das vendas
* Receita por categoria
* Produtos mais vendidos
* Clientes mais lucrativos
* Vendas por região
* Média diária de vendas
* Taxa de recompra
* Total de pedidos realizados

Esses indicadores são recalculados a cada execução do pipeline e ficam disponíveis tanto para consumo via API quanto para uso interno do módulo de relatórios automáticos.

---

## 4. Machine Learning — Previsão de Vendas

O módulo de Machine Learning utiliza os dados processados pelo pipeline para realizar análises preditivas, com foco em séries temporais de vendas.

### Como funciona

1. Os dados históricos de vendas (já limpos pelo Pandas) são agregados por período (diário, semanal ou mensal).
2. Um modelo de previsão é treinado sobre essa série — as opções mais adequadas para este cenário são `scikit-learn` (para tendências simples) ou `Prophet`/`statsmodels` (para sazonalidade, feriados e variações cíclicas de vendas).
3. O modelo treinado gera previsões futuras (ex: próximos 7, 30 ou 90 dias).
4. O resultado é armazenado (tabela própria no PostgreSQL ou arquivo serializado com `joblib`) para ser consumido pela camada de API sem necessidade de re-treinar o modelo a cada requisição.
5. O retreinamento acontece periodicamente, de forma agendada — não em tempo real — para manter a API leve e rápida.

### Exemplos de análises

* Previsão de vendas
* Previsão de demanda
* Identificação de tendências comerciais
* Detecção de anomalias
* Análise do comportamento das vendas

---

## 5. API REST — Django REST Framework

A camada de exposição do projeto é construída com **Django REST Framework (DRF)**, responsável por disponibilizar todas as informações produzidas pelo pipeline de forma padronizada e segura.

### Por que Django REST

Diferente de frameworks mais minimalistas, o DRF traz de fábrica:

* **ORM nativo**, o que facilita versionar e consultar os dados já processados (KPIs, previsões, relatórios) sem SQL manual na camada de API
* **Serializers**, que traduzem DataFrames/objetos Python para JSON de forma consistente
* **ViewSets + Routers**, que organizam os endpoints de forma padronizada e escalável
* **Painel administrativo (Django Admin)**, útil para inspecionar dados durante o desenvolvimento
* **Sistema de autenticação e permissões** já integrado, importante quando a API passa a ser consumida por sistemas externos

### Princípio arquitetural

A API **não processa dados em tempo real** — ela apenas lê resultados já calculados pelas camadas anteriores (ETL, KPIs, ML). Isso mantém as respostas rápidas e previsíveis, mesmo com grandes volumes de dados históricos.

### Endpoints

```http
GET /api/sales/
GET /api/customers/
GET /api/products/
GET /api/kpis/
GET /api/forecast/
GET /api/reports/
GET /api/insights/
GET /api/analytics/
GET /api/nl-query/      # consultas em linguagem natural via LLM
```

---

## 6. Integrações

A arquitetura foi projetada para ser independente da ferramenta consumidora dos dados — qualquer sistema capaz de consumir uma API REST/JSON pode se conectar.

### Endpoints para Business Intelligence

Os endpoints de KPIs, previsões e relatórios são expostos em JSON simples e paginado, formato compatível nativamente com:

* Power BI
* Streamlit
* Tableau
* Apache Superset
* Metabase
* Aplicações Web
* Aplicações Mobile
* Serviços internos

### Endpoint para consultas em linguagem natural (LLM)

Além do consumo estruturado, o projeto permite integração com Large Language Models (LLMs) para realizar consultas em linguagem natural sobre os dados processados.

**Como funciona:**

1. O usuário envia uma pergunta em texto livre para o endpoint `/api/nl-query/`.
2. O LLM interpreta a intenção da pergunta e a relaciona aos KPIs/dados já calculados (não ao banco bruto, por questões de segurança e controle).
3. A resposta é formulada em linguagem natural e devolvida ao usuário.

**Exemplos de perguntas suportadas:**

```text
Qual foi o produto mais vendido deste mês?

Qual categoria apresentou maior crescimento?

Qual foi o faturamento do último trimestre?

Mostre os KPIs comerciais desta semana.

Existe alguma anomalia identificada nas vendas?
```

---

## 7. Relatórios Automáticos

Um job agendado (ex: Celery + Celery Beat) roda periodicamente e:

1. Coleta os KPIs mais recentes calculados pelo pipeline;
2. Gera insights automáticos (podendo utilizar o próprio LLM para redigir o resumo em linguagem natural);
3. Disponibiliza o relatório via endpoint `/api/reports/` ou envia por outro canal (e-mail, Slack, etc., conforme configuração do ambiente).

---

## Objetivo do projeto

Este projeto tem como objetivo demonstrar a construção de uma arquitetura completa de Engenharia de Dados, abordando todas as etapas do ciclo de vida dos dados: **geração, armazenamento, extração, transformação, análise, disponibilização e consumo por aplicações externas.**

Seu foco principal está na **arquitetura do pipeline de dados** e na **integração entre diferentes componentes do ecossistema analítico**, simulando cenários encontrados em ambientes corporativos modernos — com ênfase em manter a camada de exposição (Django REST) simples, rápida e agnóstica em relação à ferramenta consumidora, seja ela um dashboard de BI ou um LLM realizando consultas em linguagem natural.
