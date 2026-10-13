# ADR-001 — Bancos de dados de origem

- **Status:** proposta para a Etapa 1
- **Data:** 2026-10-12
- **Decisores:** equipe do projeto RetailPulse

## Contexto

O projeto precisa simular as operações de um comércio eletrônico e fornecer dados coerentes para exercícios de ingestão, CDC, processamento distribuído, qualidade e modelagem analítica. Também existe a intenção de demonstrar ingestão de uma fonte relacional e de uma fonte documental.

## Decisão

Usar PostgreSQL como banco transacional principal e MongoDB, em uma etapa posterior, para documentos de avaliações de produtos.

O PostgreSQL armazenará clientes, catálogo, pedidos, itens, pagamentos, estoque, movimentações, expedições, eventos logísticos, devoluções e reembolsos. O MongoDB armazenará avaliações e comentários que podem ter atributos opcionais e específicos por categoria.

## Alternativas consideradas

1. **Somente PostgreSQL:** é suficiente para iniciar e reduz a complexidade. Não demonstra, porém, integração de fontes relacionais e documentais.
2. **PostgreSQL e MongoDB desde o primeiro dia:** demonstra duas fontes, mas amplia o número de componentes antes de a origem principal estar validada.
3. **Somente MongoDB:** foi descartado para o núcleo transacional, pois as relações entre pedidos, itens, pagamentos e estoque são centrais para o exercício de integridade referencial.

## Consequências

- PostgreSQL será implantado e validado primeiro.
- MongoDB será integrado apenas depois da ingestão relacional básica.
- As referências de `product_id` e `customer_id` entre avaliações no MongoDB e tabelas PostgreSQL não serão chaves estrangeiras físicas entre bancos. A existência das referências será validada na camada Silver.
- Será necessário documentar como cargas iniciais e CDC serão reconciliados, evitando perdas e duplicidades.
- A arquitetura deverá proteger credenciais, usar volumes persistentes e gerar apenas dados sintéticos.

## Fora do escopo desta decisão

Esta ADR não escolhe ainda versões de imagens Docker, estratégia de CDC, configurações de rede ou parâmetros de produção. Essas escolhas serão registradas quando forem necessárias e testadas.
