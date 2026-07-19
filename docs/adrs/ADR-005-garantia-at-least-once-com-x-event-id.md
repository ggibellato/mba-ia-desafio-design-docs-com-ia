# Garantia de entrega at-least-once com X-Event-Id

- Status: Aceito
- Data: 2026-07-19 (data de registro deste ADR; a decisão foi tomada na reunião técnica descrita em `TRANSCRICAO.md`)
- Deciders: Diego (Engenheiro Sênior, Plataforma), Sofia (Engenharia de Segurança), Larissa (Tech Lead)
- Relacionado: `TRANSCRICAO.md` [09:24–09:26]; ADR-001 (Outbox no MySQL), ADR-003 (Retry e DLQ)

## Status

Aceito.

## Contexto

Combinada com o mecanismo de retry (ADR-003), a entrega de webhooks pode, em cenários de falha de rede ou timeout do lado do cliente, resultar no mesmo evento sendo enviado mais de uma vez — por exemplo, se a chamada HTTP for bem-sucedida no cliente mas a confirmação se perder antes de chegar ao worker. Diego trouxe isso explicitamente: a plataforma vai garantir at-least-once, não exactly-once, e o cliente precisa estar preparado para receber o mesmo evento duas vezes (`[09:24]`). Bruno perguntou como o cliente diferenciaria eventos duplicados; Diego propôs enviar um `event_id` (UUID gerado no momento em que o evento entra na outbox) no header `X-Event-Id`, único por evento, para que o cliente possa deduplicar do lado dele (`[09:25]`).

Sofia observou que essa abordagem transfere a responsabilidade de deduplicação para o cliente. Diego reconheceu isso, mas justificou como padrão de mercado — citando Stripe e GitHub como exemplos de provedores que adotam a mesma estratégia — e argumentou que garantir exactly-once exigiria coordenação complexa entre os dois lados, resolvendo apenas uma fração adicional dos casos em troca de complexidade desproporcional (`[09:25]`). Marcos se comprometeu a documentar esse comportamento de forma destacada no portal de desenvolvedor para os clientes (`[09:26]`).

## Decisão

A plataforma garante entrega at-least-once, não exactly-once. Cada evento inserido na `webhook_outbox` (ver ADR-001) recebe um identificador único (`event_id`, UUID) no momento da inserção. Esse identificador é enviado em todo envio/reenvio do evento no header `X-Event-Id`. A responsabilidade de deduplicar eventos recebidos mais de uma vez fica a cargo do cliente, que deve usar o `X-Event-Id` como chave de deduplicação do lado dele. Esse comportamento deve ser documentado de forma destacada no portal de desenvolvedor.

## Alternativas Consideradas

- **Garantia de entrega exactly-once.** Descartada por Diego: exigiria coordenação transacional entre a plataforma e cada cliente (por exemplo, confirmação transacional de recebimento), uma complexidade desproporcional ao ganho, considerando que at-least-once com deduplicação client-side já resolve a vasta maioria dos casos práticos e é o padrão adotado por provedores de referência como Stripe e GitHub (`[09:25]`).

## Consequências

### Positivas

- Simplicidade de implementação no lado da plataforma: não é necessário nenhum protocolo de confirmação transacional com o cliente, apenas reenviar em caso de falha (ADR-003).
- Alinhado a um padrão de mercado já validado por provedores como Stripe e GitHub, reduzindo o risco de a integração surpreender clientes acostumados a esse modelo.
- O `event_id` gerado na inserção na outbox (ADR-001) já é suficiente como chave de deduplicação, sem exigir infraestrutura adicional do lado da plataforma.

### Negativas

- Transfere para o cliente a responsabilidade de implementar deduplicação correta; um cliente que não implementa essa lógica pode processar o mesmo evento (ex.: mudança de status) mais de uma vez.
- Exige comunicação e documentação claras (portal de desenvolvedor) para que os clientes entendam e implementem a deduplicação corretamente — uma falha de comunicação aqui vira bug reportado por terceiros.
- Não elimina duplicidade na origem, apenas a torna detectável; se um cliente ignorar o `X-Event-Id`, efeitos colaterais duplicados no sistema dele são possíveis.
