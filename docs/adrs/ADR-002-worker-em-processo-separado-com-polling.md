# Worker em processo separado, com polling de 2 segundos

- Status: Aceito
- Data: 2026-07-19 (data de registro deste ADR; a decisão foi tomada na reunião técnica descrita em `TRANSCRICAO.md`)
- Deciders: Diego (Engenheiro Sênior, Plataforma), Larissa (Tech Lead), Bruno (Engenheiro Pleno, Pedidos)
- Relacionado: `TRANSCRICAO.md` [09:08–09:11], [09:29–09:30]; ADR-001 (Outbox no MySQL)

## Status

Aceito.

## Contexto

Com o padrão Outbox definido (ADR-001), a próxima decisão foi como o processo que envia os webhooks lê a tabela `webhook_outbox`. Bruno perguntou se seria possível usar um trigger de banco para ser mais reativo. Diego explicou que o MySQL não tem um mecanismo de listener nativo equivalente a `NOTIFY`/`LISTEN` do Postgres; um trigger de banco só executa SQL, não consegue notificar um processo externo diretamente, e as alternativas para contornar isso (escrever em arquivo, chamar um endpoint a partir do trigger) foram consideradas inadequadas (`[09:09]`). Diego propôs então polling em loop, a cada 2 segundos, buscando os eventos pendentes mais antigos, o que atende com folga o requisito de latência "abaixo de 10 segundos" definido pelos clientes (`[09:09–09:10]`).

Diego também apontou que o worker precisa rodar como processo separado da API — não dentro da mesma instância do servidor HTTP — porque, senão, um restart da API derrubaria o worker junto (`[09:11]`). Larissa observou que já existe um padrão de entry-point único no projeto, `src/server.ts`, que sobe a aplicação Express (`src/server.ts`, `src/app.ts`), e propôs replicar esse padrão criando `src/worker.ts` como um novo ponto de entrada, com um script `npm run worker` (`[09:11]`). Bruno levantou que o worker precisaria se conectar ao mesmo banco usando o mesmo Prisma; Diego confirmou mesmo banco e mesma `DATABASE_URL`, mas reforçou que `PrismaClient` é por processo — o worker precisa de sua própria instância, não pode compartilhar a instância da API porque são processos Node distintos (`[09:29–09:30]`).

## Decisão

Implementar o worker como um processo Node separado da API, com um novo entry-point `src/worker.ts` (análogo ao `src/server.ts` já existente), instanciando seu próprio `PrismaClient` conectado ao mesmo banco/`DATABASE_URL` da API. O worker roda um loop de polling com intervalo fixo de 2 segundos, buscando em cada iteração um lote pequeno dos eventos pendentes mais antigos da `webhook_outbox` (ordenados por `created_at`), processando-os e marcando-os como entregues ou falhos.

## Alternativas Consideradas

- **Trigger de banco de dados para notificação reativa.** Descartada porque o MySQL não oferece um mecanismo nativo de listener para processos externos (diferente do `NOTIFY`/`LISTEN` do Postgres); as soluções alternativas para contornar essa limitação (escrever em arquivo, chamar um endpoint HTTP a partir do trigger) foram consideradas frágeis e fora do padrão (`[09:09]`).
- **Worker rodando dentro do mesmo processo da API.** Rejeitada implicitamente por Diego: acoplar o worker ao processo da API faria com que um restart do servidor HTTP derrubasse também o processamento da outbox, quebrando a garantia de entrega (`[09:11]`).

## Consequências

### Positivas

- Reaproveita um padrão de entry-point já estabelecido no projeto (`src/server.ts`), reduzindo a superfície de decisões novas de infraestrutura.
- Isola a disponibilidade do worker da disponibilidade da API: reiniciar um não derruba o outro.
- Latência de entrega no pior caso (2 segundos de polling) fica bem dentro do requisito de "abaixo de 10 segundos" definido pelos clientes.

### Negativas

- Introduz uma segunda instância de `PrismaClient` e um segundo processo Node a ser operado, monitorado e implantado (deploy, restart, healthcheck próprios).
- Latência mínima de entrega não é zero: mesmo no melhor caso, há até 2 segundos de atraso entre o commit da transação e a primeira tentativa de leitura pelo worker (`[09:10]`, decisão explicitamente aceita por Larissa e Marcos como trade-off).
- Polling consome ciclos de leitura no banco continuamente, mesmo quando não há eventos pendentes — custo aceito como razoável para a escala atual do time, mas não gratuito.
