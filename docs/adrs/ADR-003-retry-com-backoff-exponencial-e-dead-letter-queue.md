# Retry com backoff exponencial e Dead Letter Queue

- Status: Aceito
- Data: 2026-07-19 (data de registro deste ADR; a decisão foi tomada na reunião técnica descrita em `TRANSCRICAO.md`)
- Deciders: Diego (Engenheiro Sênior, Plataforma), Larissa (Tech Lead), Bruno (Engenheiro Pleno, Pedidos), Sofia (Engenharia de Segurança)
- Relacionado: `TRANSCRICAO.md` [09:14–09:19], [09:35–09:36]; ADR-001 (Outbox no MySQL)

## Status

Aceito.

## Contexto

Como o cliente do webhook pode estar temporariamente indisponível, a reunião discutiu o que fazer quando uma tentativa de entrega falha. Diego propôs backoff exponencial: reter e tentar novamente após um intervalo crescente e, após um teto de tentativas, considerar falha permanente (`[09:15]`). O número de tentativas foi debatido: Bruno sugeriu 3, mais agressivo; Diego argumentou que 3 é pouco, citando um caso real em que um cliente teve indisponibilidade de duas horas por manutenção planejada e teria sido descartado prematuramente com apenas três tentativas em 30 minutos (`[09:16]`). A equipe fechou em 5 tentativas, com progressão de backoff de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas — um total de aproximadamente 15 horas entre a primeira falha e a última tentativa, considerado aceitável por Marcos mesmo para uma indisponibilidade prolongada do lado do cliente (`[09:16–09:17]`).

Para eventos que esgotam as tentativas, Diego propôs uma tabela separada, `webhook_dead_letter`, guardando payload, motivo da falha e timestamp, em vez de apenas marcar o evento como "failed" na própria outbox — mantendo a outbox principal limpa e a DLQ como evidência para debug e reprocessamento (`[09:18]`). Bruno perguntou quem reprocessa esses eventos; Diego respondeu que o reprocessamento é manual, via endpoint administrativo `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento como pendente na outbox (`[09:18]`). Sofia exigiu que esse endpoint fosse restrito à role `ADMIN`, já que mexer na fila de entrega de notificações não é atribuição de operador, e que o replay fosse logado para fins de auditoria (`[09:35–09:36]`). O projeto já possui o mecanismo `requireRole` (`src/middlewares/auth.middleware.ts:49-61`), usado hoje para restringir a criação de usuários à role `ADMIN` (`src/modules/users/user.routes.ts:15`), e que será reaproveitado para proteger esse endpoint.

## Decisão

Implementar retry com backoff exponencial de 5 tentativas, nos intervalos 1min / 5min / 30min / 2h / 12h. Após a quinta tentativa falhar, o evento é considerado falha permanente e movido para uma tabela separada `webhook_dead_letter`, contendo o payload original, o motivo da última falha e o timestamp. O reprocessamento de eventos na DLQ é manual, através do endpoint `POST /admin/webhooks/dead-letter/:id/replay`, protegido por `requireRole('ADMIN')` (reaproveitando o middleware já existente em `src/middlewares/auth.middleware.ts`), com o autor do replay registrado para auditoria.

## Alternativas Consideradas

- **3 tentativas de retry.** Descartada por ser insuficiente diante de indisponibilidades reais já observadas em clientes (até duas horas de manutenção planejada), o que faria a plataforma desistir de eventos legítimos cedo demais (`[09:16]`).
- **Retry indefinido com backoff, sem teto.** Descartada por Diego: deixaria eventos "pendurados" indefinidamente caso o cliente desapareça de vez, sem um ponto claro de intervenção manual (`[09:15]`).
- **Marcar falha permanente diretamente na tabela `webhook_outbox` (campo `status = failed`), sem tabela dedicada.** Rejeitada implicitamente em favor de uma tabela separada, por manter a leitura da outbox principal mais limpa e fornecer um registro dedicado (payload, motivo, timestamp) para debug e reprocessamento (`[09:18]`).

## Consequências

### Positivas

- Tolerância a indisponibilidades reais e prolongadas do lado do cliente (até ~15 horas), reduzindo falsos negativos de entrega.
- Separação clara entre eventos em processamento normal (`webhook_outbox`) e eventos que exigem intervenção manual (`webhook_dead_letter`), facilitando operação e auditoria.
- Reaproveita `requireRole`, mecanismo de autorização já existente e testado no projeto, para proteger o endpoint sensível de replay.

### Negativas

- Um evento legitimamente perdido só é definitivamente classificado como falha após quase 15 horas, atrasando qualquer alerta ou ação corretiva automática sobre aquele cliente.
- O reprocessamento de eventos na DLQ é manual — não há retry automático após o teto, exigindo intervenção de um operador com role `ADMIN` para cada caso.
- Introduz mais uma tabela (`webhook_dead_letter`) e mais um endpoint administrativo a manter, versionar e proteger.
