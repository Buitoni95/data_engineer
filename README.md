# RetailPulse

**Status do projeto:** planejamento e modelagem inicial (Etapa 1)  
**Versão da documentação:** 0.1  
**Objetivo:** construir uma plataforma local de Engenharia de Dados ponta a ponta para um cenário simulado de varejo, evoluindo de forma incremental e documentada.

## Problema de negócio

A RetailPulse é uma empresa fictícia de comércio eletrônico. Ela precisa acompanhar vendas, pagamentos, estoque, entregas e a experiência dos clientes. O projeto demonstrará como dados operacionais podem ser integrados, tratados, testados e disponibilizados para análises e uma aplicação de IA generativa.

## Escopo planejado

- PostgreSQL como banco transacional principal.
- MongoDB como origem complementar para avaliações de produtos em documentos flexíveis.
- Python para simular operações e gerar eventos.
- Airbyte para ingestão em lote/incremental.
- Kafka e Debezium para eventos e CDC.
- MinIO como armazenamento de objetos local.
- Apache Spark e Delta Lake para processamento e tabelas do lakehouse.
- Camadas ING, Silver e Gold.
- dbt-spark para modelos analíticos e testes.
- Great Expectations e testes dbt para qualidade dos dados.
- JupyterLab com PySpark para análise exploratória.
- Hadoop YARN para uma etapa posterior de execução distribuída em cluster.
- Qdrant, modelo de linguagem local, FastAPI e Streamlit para busca semântica, IA e interface.
- Docker Compose para administrar o ambiente local; automação e CI serão adicionadas em etapas posteriores.

## Situação atual

Somente o escopo e a modelagem inicial estão sendo preparados. Os componentes listados não devem ser considerados implantados até que sejam implementados, testados e documentados.

## Primeira etapa

1. Definir entidades, relacionamentos e regras transacionais.
2. Revisar o diagrama ER em [`docs/data-model.drawio`](docs/data-model.drawio).
3. Revisar a especificação em [`docs/data-model.md`](docs/data-model.md).
4. Registrar a decisão de banco de dados em [`docs/decisions/ADR-001-database-source.md`](docs/decisions/ADR-001-database-source.md).
5. Só então implementar a criação do PostgreSQL e validar persistência e integridade.

## Princípios do projeto

- Começar pequeno e acrescentar uma tecnologia por vez.
- Usar apenas dados sintéticos; não inserir dados pessoais reais.
- Registrar decisões, premissas, testes e limitações.
- Separar ambiente exploratório de pipelines produtivos.
- Tratar métricas do Data Warehouse como resultados reproduzíveis e testáveis.
- Não adicionar uma tecnologia sem uma responsabilidade clara no cenário.

## Documentação

Consulte a pasta [`docs/`](docs/) para o modelo de dados e as decisões arquiteturais. Conforme o projeto avançar, serão adicionados diagramas de arquitetura implantada, instruções de execução, resultados dos testes e guias de operação.
