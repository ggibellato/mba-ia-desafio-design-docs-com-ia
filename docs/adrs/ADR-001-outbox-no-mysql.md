# Outbox pattern no MySQL para entrega de eventos de webhook

- Status: Aceito
- Data: 2026-07-19 (data de registro deste ADR; a decisão foi tomada na reunião técnica descrita em `TRANSCRICAO.md`)
- Deciders: Diego (Engenheiro Sênior, Plataforma), Bruno (Engenheiro Pleno, Pedidos), Larissa (Tech Lead)
- Relacionado: `TRANSCRICAO.md` [09:03–09:08], [09:40–09:41]; `RESUMO_TRANSCRICAO.md`

## Status

Aceito.

## Contexto

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) pediram notificação em tempo real (definido pelos clientes como latência abaixo de 10 segundos, `[09:02]`) quando o status de um pedido muda, em vez de continuar fazendo polling manual em `GET /orders`.

A primeira pergunta discutida foi onde disparar essa notificação. Bruno apontou que a transação de mudança de status já é pesada — "atualiza orders, insere na order_status_history, decrementa stock_quantity dos produtos do pedido" — e que inserir uma chamada HTTP síncrona ali travaria a mudança de status de outros pedidos se o cliente estivesse lento, além de não ser possível fazer rollback caso o cliente estivesse fora do ar (`[09:04]`). Diego confirmou que síncrono estava fora de questão e propôs o padrão Outbox: dentro da mesma transação SQL que já atualiza `orders` e `order_status_history`, inserir uma linha em uma tabela `webhook_outbox` com o evento; um worker separado lê essa tabela e dispara as chamadas HTTP de forma assíncrona (`[09:06]`).

No código existente, essa transação já vive em `src/modules/orders/order.service.ts`, no método `OrderService.changeStatus` (`order.service.ts:126-179`), que executa dentro de `this.prisma.$transaction(async (tx) => {...})` e já grava um registro em `OrderStatusHistory` na mesma transação. A inserção na `webhook_outbox` se encaixa nesse mesmo bloco transacional.

## Decisão

Adotar o padrão Outbox implementado como tabela no MySQL já existente (não uma fila externa dedicada). A inserção do evento na tabela `webhook_outbox` ocorre dentro da mesma transação Prisma (`tx`) usada por `OrderService.changeStatus` para atualizar `order`, inserir em `order_status_history` e ajustar `stockQuantity`. Se a transação principal fizer commit, o evento foi registrado; se der rollback, o evento desaparece junto — não há inconsistência possível entre estado do pedido e presença do evento na outbox (`[09:06]`, `[09:40–09:41]`).

A tabela terá índice em `status` (pendente, processando, falhou, entregue) e em `created_at`, para permitir que o worker leia eficientemente os eventos pendentes mais antigos em lotes pequenos. Linhas já entregues são arquivadas após aproximadamente 30 dias — mecanismo fora do escopo desta feature (`[09:08]`).

## Alternativas Consideradas

- **Fila externa dedicada (ex.: Redis Streams).** Diego descartou essa opção por representar overengineering para um time pequeno: subir e operar um cluster Redis adicional apenas para esse fluxo não se justifica quando o MySQL já existente resolve o problema (`[09:07]`).
- **Entrega síncrona dentro do `OrderService.changeStatus`.** Descartada por Bruno e Larissa: acoplaria a mudança de status à disponibilidade/latência de um sistema externo, travando outros pedidos em caso de cliente lento, e sem opção de rollback caso o cliente estivesse fora do ar (`[09:03–09:04]`).

## Consequências

### Positivas

- Garantia forte de atomicidade entre a mudança de status do pedido e o registro do evento a ser notificado, sem exigir um coordenador de transação distribuída — o commit/rollback do MySQL já cobre os dois lados (`[09:06]`, `[09:40–09:41]`).
- Não introduz nova peça de infraestrutura: reaproveita o banco MySQL/Prisma já operado pela equipe (`[09:07]`).
- Mudança de status de um pedido nunca fica bloqueada pela disponibilidade ou latência de um endpoint de cliente externo.

### Negativas

- A entrega deixa de ser síncrona: existe uma janela de tempo entre a mudança de status e o envio do webhook, cuja duração depende do worker (ver ADR-002).
- A tabela `webhook_outbox` cresce continuamente e precisa de rotina de arquivamento (não implementada nesta feature; ver `[09:08]`), sob risco de acúmulo de linhas entregues ao longo do tempo.
- Acopla ainda mais responsabilidade ao método `OrderService.changeStatus`, que já concentra atualização de status, histórico e estoque — qualquer falha na inserção da outbox agora também derruba essa transação.
