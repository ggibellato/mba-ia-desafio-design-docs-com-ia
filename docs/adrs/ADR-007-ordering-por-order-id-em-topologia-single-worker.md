# Garantia de ordering apenas por order_id, em topologia single-worker

- Status: Aceito
- Data: 2026-07-19 (data de registro deste ADR; a decisão foi tomada na reunião técnica descrita em `TRANSCRICAO.md`)
- Deciders: Diego (Engenheiro Sênior, Plataforma), Larissa (Tech Lead), Marcos (Product Manager)
- Relacionado: `TRANSCRICAO.md` [09:12–09:14]; ADR-001 (Outbox no MySQL), ADR-002 (Worker em processo separado com polling)

## Status

Aceito.

## Contexto

Larissa levantou o cenário em que um mesmo pedido muda de status rapidamente em sequência (por exemplo, `PAID` → `PROCESSING` → `SHIPPED`) e perguntou se o cliente receberia os eventos na ordem correta (`[09:12]`). Diego explicou que a resposta depende da topologia do worker: com um único worker processando a `webhook_outbox` em ordem de `created_at`, o cliente recebe os eventos na ordem em que foram inseridos — ou seja, ordering implícita por `order_id`, garantida enquanto houver apenas um worker ativo. Se o sistema escalar para múltiplos workers processando em paralelo no futuro, essa garantia se perde (`[09:12]`).

Bruno perguntou o que aconteceria se um dia fosse necessário escalar; Diego respondeu que, nesse caso, seria possível particionar o processamento por `order_id` ou usar lock pessimista, mas que isso é "problema do futuro", não desta feature (`[09:13]`). Larissa fechou o ponto determinando que essa limitação seja documentada explicitamente: não há garantia de ordering global, apenas ordering por `order_id`, e apenas enquanto a topologia for single-worker (`[09:13]`). Marcos confirmou que os clientes nunca pediram garantia de ordering global — eles só precisam saber quando cada pedido deles muda de status, individualmente (`[09:14]`).

No modelo de dados existente, os eventos relacionados a um pedido já são naturalmente agrupáveis por `order_id`: `OrderItem` e `OrderStatusHistory` têm índice em `orderId` (`prisma/schema.prisma`, modelos `OrderItem` e `OrderStatusHistory`), e `Order` tem índice em `createdAt` — o mesmo tipo de campo (`created_at`) que a `webhook_outbox` usará para ordenar a leitura do worker (ADR-001).

## Decisão

O sistema garante ordering de entrega de eventos apenas por `order_id`, e apenas enquanto o processamento da `webhook_outbox` for feito por um único worker (topologia single-worker, ADR-002), lendo os eventos pendentes em ordem de `created_at`. Não há garantia de ordering global entre pedidos diferentes, nem garantia de ordering caso a topologia evolua para múltiplos workers processando em paralelo. Essa limitação é documentada explicitamente como conhecida e aceita para o escopo atual da feature.

## Alternativas Consideradas

- **Particionamento do processamento por `order_id` (ex.: hashing consistente de `order_id` para workers) ou uso de lock pessimista por pedido, para permitir múltiplos workers em paralelo mantendo ordering.** Considerada plausível por Diego para uma fase futura de escala, mas explicitamente adiada: não há necessidade identificada agora, e adicionar essa complexidade sem um problema real de throughput seria antecipação desnecessária (`[09:13]`).

## Consequências

### Positivas

- Solução simples: nenhuma lógica adicional de particionamento, coordenação entre workers ou locks é necessária para o escopo atual.
- Atende integralmente à necessidade real dos clientes, que é acompanhar a evolução de status de cada pedido individualmente, não uma ordenação global entre pedidos distintos (`[09:14]`).
- Mantém abertura arquitetural: a limitação é conhecida e documentada, e o caminho de evolução (particionamento por `order_id` ou lock pessimista) já está identificado caso o sistema precise escalar para múltiplos workers.

### Negativas

- É uma garantia frágil a mudanças futuras: se alguém subir um segundo worker sem revisitar esta decisão, a ordering por `order_id` deixa de valer silenciosamente, sem nenhum mecanismo de proteção no código.
- Limita a escala do processamento de eventos a um único worker enquanto esta decisão não for revisitada — um gargalo de throughput conhecido, ainda que aceitável para o volume atual.
- Não há garantia nenhuma de ordering entre pedidos diferentes, o que pode surpreender um cliente que dependa (incorretamente) de causalidade entre eventos de pedidos distintos.
