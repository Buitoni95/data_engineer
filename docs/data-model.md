# RetailPulse — especificação inicial do modelo de dados

- **Versão:** 0.1
- **Status:** proposta para revisão antes da implementação
- **Data:** 12/10/2026
- **Banco principal:** PostgreSQL
- **Fonte documental complementar planejada:** MongoDB
- **Diagrama editável:** [`data-model.drawio`](data-model.drawio)

## 1. Objetivo e escopo

Este documento especifica o modelo operacional inicial de uma loja virtual fictícia. O modelo foi desenhado para ser simples o suficiente para executar em ambiente local, mas rico o bastante para produzir dados úteis em exercícios posteriores de ingestão, CDC, streaming, qualidade, Lakehouse, Data Warehouse e IA generativa.

O PostgreSQL será o sistema transacional de referência para o núcleo comercial e operacional. O MongoDB será introduzido em uma etapa posterior para avaliações de produtos com atributos flexíveis. Os dados gerados serão inteiramente sintéticos.

### Incluído no MVP

- Clientes e catálogo de produtos.
- Pedidos compostos por um ou mais itens.
- Tentativas de pagamento por pedido.
- Estoque atual e histórico de movimentações.
- Uma expedição por pedido, com histórico de eventos logísticos.
- Solicitações de devolução com um ou mais itens e reembolsos.
- Avaliações documentais de produtos no MongoDB, adicionadas após validar a fonte relacional.

### Fora do MVP

- Múltiplos centros de distribuição ou depósitos.
- Mais de uma expedição por pedido.
- Pagamento dividido entre vários meios de pagamento.
- Múltiplas moedas, tributação fiscal detalhada, cupons e programas de fidelidade.
- Cadastro real de pessoas ou uso de dados pessoais reais.
- Escrita direta do assistente de IA no banco operacional.

Essas exclusões são limites de escopo, não afirmações de que tais capacidades sejam desnecessárias em sistemas comerciais reais.

## 2. Convenções

- Nomes de tabelas, colunas e valores de status serão mantidos em inglês e em `snake_case`; a documentação e as regras serão explicadas em português.
- Chaves internas serão identificadores numéricos gerados pelo banco. Identificadores de negócio, como `order_number` e `sku`, serão mantidos separadamente.
- Valores monetários serão armazenados em `NUMERIC(12,2)`, na moeda BRL. Não usar ponto flutuante binário para valores financeiros.
- Quantidades de estoque e de itens serão inteiros positivos ou zero, conforme a semântica da coluna.
- Instantes serão armazenados com fuso (`TIMESTAMPTZ`) e normalizados em UTC. A interface poderá exibir o horário local de São Paulo.
- Datas e status devem ser validados na origem; transformações downstream também terão testes independentes.
- `created_at` representa a criação do registro. `updated_at` representa a última alteração da entidade quando a tabela não for append-only.
- Tabelas de histórico de eventos e movimentações serão tratadas como append-only no fluxo normal. Correções deverão gerar eventos/movimentos compensatórios, em vez de apagar a trilha histórica.

## 3. Visão das entidades e relacionamentos

| Entidade | Finalidade | Relacionamento principal |
|---|---|---|
| `customers` | Cadastro sintético dos clientes | Um cliente pode realizar vários pedidos |
| `categories` | Categorias de catálogo | Uma categoria pode conter vários produtos |
| `products` | Catálogo e preço atual | Um produto pode aparecer em vários itens de pedido e movimentos de estoque |
| `orders` | Cabeçalho do pedido e totais do pedido | Um pedido tem itens, tentativas de pagamento e pode ter expedição e devoluções |
| `order_items` | Produtos e quantidades compradas em cada pedido | Cada item aponta para um pedido e produto |
| `payments` | Tentativas de pagamento | Um pedido pode ter várias tentativas sequenciais |
| `inventory` | Posição atual do estoque por produto | Uma posição por produto no MVP |
| `inventory_movements` | Livro de movimentos de estoque | Cada movimento aponta para um produto e pode apontar para um item do pedido |
| `shipments` | Situação logística atual | No MVP, no máximo uma expedição por pedido |
| `shipment_events` | Histórico dos eventos de uma expedição | Uma expedição pode ter muitos eventos cronológicos |
| `returns` | Solicitação de devolução | Um pedido pode ter várias solicitações de devolução |
| `return_items` | Itens e quantidades incluídos em uma devolução | Cada linha aponta para uma solicitação e para um item comprado |
| `refunds` | Reembolsos associados a pagamentos e, quando aplicável, a devoluções | Um pagamento pode ter mais de um reembolso parcial |
| `product_reviews` (MongoDB) | Avaliações e comentários em documentos flexíveis | Referências lógicas a produto e cliente; sem FK entre bancos |

Cardinalidades detalhadas:

- `customers 1:N orders`
- `categories 1:N products`
- `orders 1:N order_items`
- `products 1:N order_items`
- `orders 1:N payments`
- `products 1:1 inventory` no MVP
- `products 1:N inventory_movements`
- `order_items 1:N inventory_movements` (a referência ao item pode ser nula para movimentos que não derivam de um pedido)
- `orders 0:1 shipments` no MVP
- `shipments 1:N shipment_events`
- `orders 1:N returns`
- `returns 1:N return_items`
- `order_items 1:N return_items`
- `payments 1:N refunds`
- `returns 0:N refunds` (o vínculo é opcional para reembolsos não associados diretamente a uma devolução)
- Os campos `product_id` e `customer_id` de `product_reviews` são referências lógicas para as entidades PostgreSQL, validadas na Silver; não são relacionamentos físicos entre bancos.

## 4. Especificação das tabelas PostgreSQL

A lista abaixo é a especificação lógica para orientar o DDL. Ela não é o script de criação das tabelas; a implementação será feita na próxima subetapa.

### 4.1 `customers`

| Campo | Tipo sugerido | Obrigatório | Regra |
|---|---|---:|---|
| `customer_id` | `BIGINT` identity | Sim | PK |
| `full_name` | `VARCHAR(150)` | Sim | Não pode ser vazio após normalização |
| `email` | `VARCHAR(254)` | Sim | Único; armazenar em formato normalizado |
| `phone` | `VARCHAR(30)` | Não | Formato controlado pelo simulador |
| `status` | `VARCHAR(20)` | Sim | `active`, `inactive` ou `blocked` |
| `created_at` | `TIMESTAMPTZ` | Sim | Data/hora de criação |
| `updated_at` | `TIMESTAMPTZ` | Sim | Data/hora da última atualização |

### 4.2 `categories`

| Campo | Tipo sugerido | Obrigatório | Regra |
|---|---|---:|---|
| `category_id` | `BIGINT` identity | Sim | PK |
| `name` | `VARCHAR(100)` | Sim | Único e não vazio |
| `description` | `TEXT` | Não | Descrição da categoria |
| `is_active` | `BOOLEAN` | Sim | Indica disponibilidade no catálogo |
| `created_at` | `TIMESTAMPTZ` | Sim | Data/hora de criação |

### 4.3 `products`

| Campo | Tipo sugerido | Obrigatório | Regra |
|---|---|---:|---|
| `product_id` | `BIGINT` identity | Sim | PK |
| `category_id` | `BIGINT` | Sim | FK para `categories` |
| `sku` | `VARCHAR(50)` | Sim | Único e estável |
| `name` | `VARCHAR(180)` | Sim | Não vazio |
| `description` | `TEXT` | Não | Descrição do produto |
| `current_price` | `NUMERIC(12,2)` | Sim | Maior ou igual a zero; preço atual de catálogo |
| `is_active` | `BOOLEAN` | Sim | Indica se pode ser vendido |
| `created_at` | `TIMESTAMPTZ` | Sim | Data/hora de criação |
| `updated_at` | `TIMESTAMPTZ` | Sim | Data/hora da última atualização |

O `current_price` não será usado para recalcular vendas antigas. O preço praticado na venda ficará registrado em `order_items.unit_price`.

### 4.4 `orders`

| Campo | Tipo sugerido | Obrigatório | Regra |
|---|---|---:|---|
| `order_id` | `BIGINT` identity | Sim | PK |
| `order_number` | `VARCHAR(30)` | Sim | Identificador de negócio único |
| `customer_id` | `BIGINT` | Sim | FK para `customers` |
| `status` | `VARCHAR(25)` | Sim | `pending_payment`, `confirmed`, `processing`, `shipped`, `delivered` ou `cancelled` |
| `subtotal_amount` | `NUMERIC(12,2)` | Sim | Soma dos valores dos itens após descontos por item, antes do desconto geral |
| `discount_amount` | `NUMERIC(12,2)` | Sim | Desconto adicional aplicado ao pedido; entre zero e o subtotal |
| `shipping_amount` | `NUMERIC(12,2)` | Sim | Frete, maior ou igual a zero |
| `total_amount` | `NUMERIC(12,2)` | Sim | `subtotal_amount - discount_amount + shipping_amount`; não negativo |
| `shipping_recipient` | `VARCHAR(150)` | Sim | Cópia do destinatário no momento do pedido |
| `shipping_address_line1` | `VARCHAR(200)` | Sim | Cópia do endereço no momento do pedido |
| `shipping_address_line2` | `VARCHAR(200)` | Não | Complemento do endereço |
| `shipping_city` | `VARCHAR(100)` | Sim | Cidade do destino |
| `shipping_state` | `VARCHAR(50)` | Sim | Estado/UF do destino |
| `shipping_postal_code` | `VARCHAR(20)` | Sim | CEP fictício válido no formato definido pelo simulador |
| `shipping_country` | `CHAR(2)` | Sim | Código de país, inicialmente `BR` |
| `created_at` | `TIMESTAMPTZ` | Sim | Criação do pedido |
| `updated_at` | `TIMESTAMPTZ` | Sim | Última atualização |
| `cancelled_at` | `TIMESTAMPTZ` | Não | Preenchido quando o pedido for cancelado |

A cópia do endereço no pedido é intencional: alterações futuras no cadastro de um cliente não podem modificar o histórico do destino de uma compra anterior. Todos os dados pessoais deste projeto serão sintéticos.

### 4.5 `order_items`

| Campo | Tipo sugerido | Obrigatório | Regra |
|---|---|---:|---|
| `order_item_id` | `BIGINT` identity | Sim | PK |
| `order_id` | `BIGINT` | Sim | FK para `orders` |
| `line_number` | `INTEGER` | Sim | Número da linha dentro do pedido; único por pedido |
| `product_id` | `BIGINT` | Sim | FK para `products` |
| `quantity` | `INTEGER` | Sim | Maior que zero |
| `unit_price` | `NUMERIC(12,2)` | Sim | Preço unitário congelado no momento da compra; não negativo |
| `item_discount_amount` | `NUMERIC(12,2)` | Sim | Desconto total da linha; entre zero e `quantity * unit_price` |

O valor líquido de uma linha é `quantity * unit_price - item_discount_amount`. O subtotal do pedido é a soma desses valores. O preço atual do catálogo não altera o valor histórico do item.

### 4.6 `payments`

| Campo | Tipo sugerido | Obrigatório | Regra |
|---|---|---:|---|
| `payment_id` | `BIGINT` identity | Sim | PK |
| `order_id` | `BIGINT` | Sim | FK para `orders` |
| `attempt_number` | `INTEGER` | Sim | Maior que zero; único por pedido em conjunto com `order_id` |
| `method` | `VARCHAR(20)` | Sim | `pix`, `credit_card`, `debit_card` ou `boleto` |
| `status` | `VARCHAR(20)` | Sim | `pending`, `approved`, `declined` ou `cancelled` |
| `amount` | `NUMERIC(12,2)` | Sim | Maior que zero; no MVP deve corresponder ao total do pedido |
| `provider_reference` | `VARCHAR(100)` | Não | Referência fictícia do provedor; única quando preenchida |
| `created_at` | `TIMESTAMPTZ` | Sim | Criação da tentativa |
| `updated_at` | `TIMESTAMPTZ` | Sim | Última atualização |
| `approved_at` | `TIMESTAMPTZ` | Não | Preenchido se aprovada |

Uma recusa não será sobrescrita como se nunca tivesse ocorrido: cada tentativa de pagamento terá sua própria linha. No MVP, não haverá divisão de um pedido entre vários pagamentos aprovados; no máximo uma tentativa poderá ficar aprovada para cada pedido.

### 4.7 `inventory`

| Campo | Tipo sugerido | Obrigatório | Regra |
|---|---|---:|---|
| `product_id` | `BIGINT` | Sim | PK e FK para `products`; uma posição por produto no MVP |
| `on_hand_quantity` | `INTEGER` | Sim | Quantidade física existente; maior ou igual a zero |
| `reserved_quantity` | `INTEGER` | Sim | Quantidade reservada; maior ou igual a zero e menor ou igual à quantidade física |
| `updated_at` | `TIMESTAMPTZ` | Sim | Última atualização da posição |

A quantidade disponível será calculada como `on_hand_quantity - reserved_quantity`. Um único depósito será suficiente na primeira versão.

### 4.8 `inventory_movements`

| Campo | Tipo sugerido | Obrigatório | Regra |
|---|---|---:|---|
| `movement_id` | `BIGINT` identity | Sim | PK |
| `product_id` | `BIGINT` | Sim | FK para `products` |
| `order_item_id` | `BIGINT` | Não | FK para `order_items` quando o movimento vier de um item do pedido |
| `movement_type` | `VARCHAR(25)` | Sim | `receipt`, `reservation`, `release_reservation`, `shipment`, `return`, `adjustment_in` ou `adjustment_out` |
| `quantity` | `INTEGER` | Sim | Maior que zero; o tipo define o efeito |
| `idempotency_key` | `VARCHAR(120)` | Sim | Único; evita aplicar a mesma operação duas vezes |
| `occurred_at` | `TIMESTAMPTZ` | Sim | Instante do movimento |
| `notes` | `TEXT` | Não | Justificativa ou referência operacional |

Semântica dos movimentos: `receipt`, `return` e `adjustment_in` aumentam o estoque físico; `reservation` aumenta o reservado; `release_reservation` diminui o reservado; `shipment` diminui o estoque físico e a reserva correspondente; `adjustment_out` diminui o físico. Cada alteração da posição atual e seu movimento correspondente devem ser gravados atomicamente.

### 4.9 `shipments`

| Campo | Tipo sugerido | Obrigatório | Regra |
|---|---|---:|---|
| `shipment_id` | `BIGINT` identity | Sim | PK |
| `order_id` | `BIGINT` | Sim | FK para `orders` e único no MVP (uma expedição por pedido) |
| `carrier` | `VARCHAR(80)` | Sim | Transportadora fictícia |
| `tracking_code` | `VARCHAR(100)` | Não | Único quando preenchido |
| `status` | `VARCHAR(25)` | Sim | `pending`, `shipped`, `in_transit`, `out_for_delivery`, `delivered` ou `exception` |
| `estimated_delivery_at` | `TIMESTAMPTZ` | Não | Previsão de entrega |
| `shipped_at` | `TIMESTAMPTZ` | Não | Data/hora da expedição |
| `delivered_at` | `TIMESTAMPTZ` | Não | Data/hora da entrega efetiva |
| `created_at` | `TIMESTAMPTZ` | Sim | Criação do registro |
| `updated_at` | `TIMESTAMPTZ` | Sim | Última atualização |

### 4.10 `shipment_events`

| Campo | Tipo sugerido | Obrigatório | Regra |
|---|---|---:|---|
| `shipment_event_id` | `BIGINT` identity | Sim | PK |
| `shipment_id` | `BIGINT` | Sim | FK para `shipments` |
| `event_type` | `VARCHAR(30)` | Sim | `created`, `shipped`, `in_transit`, `out_for_delivery`, `delivered`, `failed_attempt` ou `exception` |
| `event_at` | `TIMESTAMPTZ` | Sim | Instante do evento |
| `location` | `VARCHAR(150)` | Não | Local fictício associado ao evento |
| `description` | `TEXT` | Não | Descrição do evento |

Eventos logísticos são históricos e não devem ser apagados na operação normal. A data de cada evento deve ser cronologicamente coerente; o status atual da expedição deve corresponder ao último evento aplicável.

### 4.11 `returns`

| Campo | Tipo sugerido | Obrigatório | Regra |
|---|---|---:|---|
| `return_id` | `BIGINT` identity | Sim | PK |
| `order_id` | `BIGINT` | Sim | FK para `orders` |
| `status` | `VARCHAR(20)` | Sim | `requested`, `approved`, `rejected`, `received` ou `completed` |
| `requested_at` | `TIMESTAMPTZ` | Sim | Data/hora da solicitação |
| `reviewed_at` | `TIMESTAMPTZ` | Não | Data/hora da aprovação ou rejeição |
| `received_at` | `TIMESTAMPTZ` | Não | Data/hora do recebimento da devolução |
| `completed_at` | `TIMESTAMPTZ` | Não | Data/hora da conclusão |

No MVP, a devolução somente pode ser solicitada após a entrega do pedido. Mudanças de estado respeitam a sequência de negócio documentada na seção 5.

### 4.12 `return_items`

| Campo | Tipo sugerido | Obrigatório | Regra |
|---|---|---:|---|
| `return_item_id` | `BIGINT` identity | Sim | PK |
| `return_id` | `BIGINT` | Sim | FK para `returns` |
| `order_item_id` | `BIGINT` | Sim | FK para `order_items` |
| `quantity` | `INTEGER` | Sim | Maior que zero |
| `reason` | `VARCHAR(40)` | Sim | `damaged`, `not_as_described`, `wrong_item`, `customer_changed_mind` ou `other` |
| `disposition` | `VARCHAR(20)` | Não | `restock`, `discard` ou `inspect`; definida após avaliação física |

A soma de quantidades solicitadas e aceitas para um item não pode exceder a quantidade comprada. Itens de devolução precisam pertencer ao mesmo pedido referenciado em `returns`; a implementação deve validar essa consistência, pois duas FKs simples, isoladamente, não garantem que ambos os registros pertencem ao mesmo pedido.

### 4.13 `refunds`

| Campo | Tipo sugerido | Obrigatório | Regra |
|---|---|---:|---|
| `refund_id` | `BIGINT` identity | Sim | PK |
| `payment_id` | `BIGINT` | Sim | FK para `payments` |
| `return_id` | `BIGINT` | Não | FK para `returns`; nulo quando não associado diretamente a uma devolução |
| `amount` | `NUMERIC(12,2)` | Sim | Maior que zero |
| `status` | `VARCHAR(20)` | Sim | `pending`, `completed`, `failed` ou `cancelled` |
| `provider_reference` | `VARCHAR(100)` | Não | Referência fictícia do provedor |
| `requested_at` | `TIMESTAMPTZ` | Sim | Solicitação do reembolso |
| `completed_at` | `TIMESTAMPTZ` | Não | Preenchido somente quando concluído |

Reembolsos parciais são permitidos. A soma dos reembolsos concluídos não pode superar o valor efetivamente aprovado no pagamento associado. Se um reembolso estiver associado a uma devolução, pedido, devolução e pagamento devem pertencer à mesma cadeia transacional.

## 5. Fluxos e regras transacionais

### 5.1 Criação do pedido e reserva de estoque

1. O simulador seleciona um cliente ativo e produtos ativos.
2. O pedido é criado com estado `pending_payment`; os itens congelam preço e desconto praticados na compra.
3. O sistema calcula e valida o subtotal e o total do pedido.
4. Deve verificar a quantidade disponível para todos os produtos.
5. A criação dos itens, as reservas de estoque, os movimentos de reserva e a posição atual do estoque devem ser consistentes. Se não for possível reservar todos os itens, a operação deve falhar integralmente ou seguir uma política explícita; no MVP, usaremos a falha integral.
6. As gravações relacionadas devem ser feitas em uma transação de banco de dados, para não deixar pedido parcial ou reserva sem trilha de movimento.

### 5.2 Pagamento

- Uma tentativa é inicialmente `pending` e depois evolui para `approved`, `declined` ou `cancelled`.
- Uma recusa pode ser seguida de nova tentativa com `attempt_number` incrementado.
- Um pedido pode ter no máximo uma tentativa aprovada no MVP.
- Quando o pagamento for aprovado, o pedido poderá passar para `confirmed` e depois `processing`.
- Se o pedido for cancelado antes do envio, a reserva deverá ser liberada uma única vez, gerando um `release_reservation` identificável por chave de idempotência.

### 5.3 Expedição e entrega

- A expedição só é criada para um pedido elegível e pago segundo a regra de negócio simulada.
- O envio gera um evento logístico e um movimento de estoque correspondente.
- O estoque físico e a reserva são atualizados na mesma transação da expedição.
- Eventos são acrescidos à trilha histórica; timestamps não devem regredir.
- Um pedido só pode estar `delivered` se houver expedição com evento de entrega coerente.

### 5.4 Devolução e reembolso

- A devolução é solicitada depois da entrega.
- Quantidade devolvida por item não pode exceder o total comprado, considerando outras devoluções válidas.
- Apenas itens recebidos e considerados aptos para revenda geram movimento de retorno ao estoque.
- Reembolsos podem ser parciais, mas o total concluído fica limitado ao valor do pagamento aprovado.
- O histórico de devoluções, reembolsos e estoque será preservado para auditoria e reconciliação.

## 6. Estados e transições esperadas

### Pedido

- `pending_payment` → `confirmed` após pagamento aprovado.
- `pending_payment` → `cancelled` após cancelamento ou esgotamento das tentativas de pagamento.
- `confirmed` → `processing` → `shipped` → `delivered`.
- `cancelled` e `delivered` são estados finais para o pedido no MVP; uma devolução é registrada em entidades próprias e não altera retroativamente o status histórico de entrega.
- Transições fora do fluxo definido devem ser recusadas ou registradas como exceção controlada.

### Pagamento

- `pending` → `approved`, `declined` ou `cancelled`.
- `declined` e `cancelled` encerram aquela tentativa; uma nova tentativa é um novo registro.
- `approved` não deve voltar a `pending` ou `declined`; devoluções financeiras são registradas em `refunds`.

### Devolução

- `requested` → `approved` ou `rejected`.
- `approved` → `received` → `completed`.
- `rejected` é final no MVP.

As transições serão executadas pelo simulador/serviço transacional, e não apenas pela escolha livre de valores de status. Os estados anteriores e as datas associadas deverão permitir reconstruir a evolução operacional.

## 7. Qualidade de dados: critérios iniciais

Estes critérios orientarão tanto os testes na origem quanto as validações posteriores em Great Expectations e dbt.

| ID | Regra | Severidade sugerida | Onde validar |
|---|---|---|---|
| DQ-001 | Chaves primárias e identificadores de negócio únicos | Crítica | PostgreSQL e Silver |
| DQ-002 | FKs relacionais válidas; nenhum item deve referenciar pedido ou produto inexistente | Crítica | PostgreSQL e Silver |
| DQ-003 | Quantidade de item maior que zero; preços e descontos dentro dos limites | Crítica | PostgreSQL e Silver |
| DQ-004 | Total do pedido consistente com itens, descontos e frete, com precisão de centavos | Crítica | Serviço transacional e Gold |
| DQ-005 | `reserved_quantity` entre zero e `on_hand_quantity` | Crítica | PostgreSQL e Silver |
| DQ-006 | Alterações de estoque têm movimento correspondente e chave de idempotência única | Crítica | PostgreSQL e Silver |
| DQ-007 | No máximo um pagamento aprovado por pedido no MVP | Crítica | PostgreSQL e Silver |
| DQ-008 | Valor de pagamento consistente com o total do pedido no MVP | Alta | PostgreSQL e Silver |
| DQ-009 | Pedido entregue tem expedição e evento de entrega coerentes | Alta | Silver e Gold |
| DQ-010 | Eventos logísticos seguem ordem temporal válida | Alta | Silver |
| DQ-011 | Quantidades de devolução aceitas não excedem as compradas | Crítica | Serviço transacional e Silver |
| DQ-012 | Reembolsos concluídos não excedem o pagamento aprovado | Crítica | Serviço transacional e Silver |
| DQ-013 | Avaliações MongoDB referenciam produtos existentes e, quando preenchido, clientes existentes | Alta | Integração entre fontes na Silver |
| DQ-014 | Nota de avaliação está no intervalo de 1 a 5 | Alta | MongoDB e Silver |
| DQ-015 | Dados usados em Gold respeitam a granularidade documentada de cada fato | Crítica | dbt |

Para regras que dependem de somas ou da relação entre várias linhas, uma simples restrição de coluna pode não ser suficiente. Elas exigirão transações controladas, validações na aplicação e/ou testes de qualidade.

## 8. Extensão MongoDB: `product_reviews`

A coleção será introduzida depois de a fonte PostgreSQL e a primeira ingestão estarem validadas. Cada documento representará uma avaliação e poderá conter:

- `_id`: identificador do documento.
- `product_id`: referência lógica obrigatória ao `products.product_id`.
- `customer_id`: referência lógica opcional ou obrigatória conforme decidirmos a política de avaliações; para o primeiro simulador, recomenda-se obrigatória para facilitar testes entre fontes.
- `rating`: número inteiro de 1 a 5.
- `title` e `comment`: texto da avaliação.
- `pros` e `cons`: listas opcionais de pontos positivos e negativos.
- `attributes`: objeto flexível para características próprias da categoria, por exemplo autonomia da bateria ou caimento de roupa.
- `moderation.status`: por exemplo `pending`, `approved` ou `rejected`.
- `created_at`: instante da avaliação.

O valor `verified_purchase` não será tratado como verdade apenas por existir em um documento. Se for necessário, será derivado/validado comparando a avaliação com pedidos e itens na Silver. Os identificadores entre bancos serão verificados durante a integração, pois não há FK física cruzando PostgreSQL e MongoDB.

## 9. Decisões e questões abertas

Decisões assumidas na versão 0.1:

1. Um único depósito e uma posição de estoque por produto.
2. Uma expedição por pedido.
3. Um pedido pode ter várias tentativas, mas não pagamentos divididos.
4. O preço de venda é congelado em `order_items`.
5. Os eventos de movimentação de estoque e de logística preservam histórico.
6. As avaliações ficam no MongoDB, e não em uma tabela relacional duplicada.
7. Todos os dados de demonstração serão sintéticos.

Antes do DDL, revisar principalmente: (a) se `customer_id` em avaliações será obrigatório; (b) o comportamento do estoque após cancelamento; (c) se a expedição só será criada após pagamento aprovado; e (d) se haverá uma política explícita de validade dos produtos inativos em pedidos históricos. A proposta atual assume que produtos inativos podem continuar aparecendo nos registros históricos, mas não podem ser escolhidos em novos pedidos.

## 10. Critérios de aceite do modelo

- O diagrama e este documento descrevem as mesmas entidades e cardinalidades.
- Todas as relações têm PK/FK ou uma justificativa explícita quando forem referências entre sistemas.
- As regras de negócio têm um local planejado de validação.
- Totais financeiros, estoque e histórico de eventos têm semântica sem ambiguidade.
- O modelo pode ser implementado localmente sem exigir dados em grande escala.
- A especificação é revisada antes da criação das tabelas e antes de se considerar a Etapa 1 concluída.
