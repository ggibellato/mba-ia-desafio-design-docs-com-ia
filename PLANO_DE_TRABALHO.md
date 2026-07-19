# Plano de Trabalho — Desafio "Da Reunião ao Documento"

> Guia passo a passo pessoal para completar o desafio. Não é um entregável do exercício (não está listado na estrutura obrigatória) — é um roteiro de execução. Pode ser apagado ou movido para fora do repo antes da entrega final, se preferir.

Referência completa dos requisitos: [`EXERCICIO.md`](./EXERCICIO.md). Este plano só resume/ordena as tarefas; em caso de dúvida, o `EXERCICIO.md` é a fonte da verdade.

## Legenda de status
- [ ] a fazer
- [~] em andamento
- [x] concluído

---

## Fase 0 — Setup [x] concluído

- [x] Fork do repositório base no GitHub (se ainda não feito) e confirmar que este diretório local aponta pro fork. (`origin` = `ggibellato/mba-ia-desafio-design-docs-com-ia`)
- [x] Confirmar que `TRANSCRICAO.md` e `src/`, `prisma/`, `tests/` estão intactos (não serão alterados durante o desafio). (sem alterações locais nesses caminhos)
- [x] Ler `TRANSCRICAO.md` uma vez, na íntegra, sem IA — só para ter contexto próprio antes de delegar leitura à IA.

## Fase 1 — Contextualização com IA

- [x] Pedir à IA uma exploração do código (`src/`, `prisma/schema.prisma`) para mapear: módulos existentes (auth, users, customers, products, orders), máquina de estados do pedido, controle de estoque, auditoria de mudanças de status, padrões de erro/exceções usados.
- [x] Pedir à IA um resumo estruturado da transcrição, separando explicitamente:
  - decisões fechadas
  - requisitos funcionais explícitos
  - restrições
  - pontos de integração com código existente
  - itens descartados
  - itens adiados para fases futuras
  - detalhes técnicos secundários
- [x] Revisar esse resumo manualmente contra a transcrição — é a base de tudo que vem depois. Erros aqui se propagam para todos os documentos.

## Fase 2 — ADRs primeiro (esqueleto das decisões)

As 6 decisões principais que a reunião discute (cobrir pelo menos 5 delas):

- [x] ADR — Padrão Outbox no MySQL (`ADR-001-outbox-no-mysql.md`)
- [x] ADR — Política de retry com backoff e DLQ (`ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md`)
- [x] ADR — Autenticação HMAC-SHA256 com secret por endpoint (`ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md`)
- [x] ADR — Garantia at-least-once com `X-Event-Id` (`ADR-005-garantia-at-least-once-com-x-event-id.md`)
- [x] ADR — Worker em processo separado em polling (`ADR-002-worker-em-processo-separado-com-polling.md`)
- [x] ADR — Reuso dos padrões existentes do projeto (`ADR-006-reuso-dos-padroes-existentes-do-projeto.md`)

Para cada ADR criado em `docs/adrs/ADR-NNN-titulo-em-kebab-case.md`:
- [x] Seção Status
- [x] Seção Contexto (rastreável à transcrição)
- [x] Seção Decisão
- [x] Seção Alternativas Consideradas (≥1 alternativa real ou plausível)
- [x] Seção Consequências (positivas e negativas, trade-off explícito)

Checklist do conjunto:
- [x] Entre 5 e 8 arquivos ADR no total. (7 arquivos: os 6 acima + `ADR-007-ordering-por-order-id-em-topologia-single-worker.md`, cobrindo a garantia de ordering discutida na reunião)
- [x] Pelo menos 1 ADR referencia arquivos/módulos/classes reais do código (`order.service.ts`, `server.ts`, `app-error.ts`, `error.middleware.ts`, `auth.middleware.ts` — referenciados em ADR-001, 002, 003 e 006, todos confirmados existentes).
- [x] Decisões técnicas secundárias (formato de payload, timeouts, headers) — decidido: ficam só no FDD, não viram ADR extra (TLS obrigatório e limite de 64KB foram classificados por Larissa na própria reunião `[09:24]` como requisito não funcional, não decisão arquitetural).

## Fase 3 — RFC (`docs/RFC.md`)

- [x] Metadados: autor, status, data, revisores (Larissa como autora; os outros 4 participantes da reunião como revisores).
- [x] Resumo executivo (TL;DR).
- [x] Contexto e problema.
- [x] Proposta técnica (visão geral, sem repetir nível de detalhe do FDD).
- [x] Alternativas consideradas: **≥2** alternativas reais descartadas na reunião, cada uma com o trade-off que motivou o descarte. (4 alternativas)
- [x] Questões em aberto: **≥2** pontos levantados e não decididos/adiados na reunião. (4 questões)
- [x] Impacto e riscos.
- [x] Decisões relacionadas: links para os ADRs já escritos na Fase 2. (7 ADRs linkados)
- [x] Conferir tamanho: 2 a 4 páginas — se estiver mais longo, provavelmente há detalhe demais de implementação (isso pertence ao FDD). (~1.640 palavras, ~3 páginas)

## Fase 4 — FDD (`docs/FDD.md`)

- [x] Contexto e motivação técnica.
- [x] Objetivos técnicos.
- [x] Escopo e exclusões.
- [x] Fluxos detalhados: criação do evento na outbox, processamento pelo worker, retry, DLQ.
- [x] Contratos públicos: **≥4** endpoints HTTP, cada um com payload de exemplo (request + response), headers, status codes, semântica. (7 endpoints documentados)
- [x] Matriz de erros previstos com códigos `WEBHOOK_*`. (8 códigos)
- [x] Estratégias de resiliência (timeouts, retries, backoff, fallback).
- [x] Observabilidade: métricas, logs e tracing (as 3 têm que aparecer).
- [x] Dependências e compatibilidade.
- [x] Critérios de aceite técnicos.
- [x] Riscos e mitigação.
- [x] **Seção obrigatória "Integração com o sistema existente"**: nomear ≥4 caminhos de arquivo reais do código base e descrever a integração de cada um (ex: como `changeStatus` é estendido, como classes de erro existentes são reaproveitadas). (9 caminhos reais, todos conferidos com `find`)

## Fase 5 — PRD (`docs/PRD.md`)

Produzido por último entre os grandes docs — deve ser consolidação do que já foi decidido em ADRs/RFC/FDD.

- [x] Resumo e contexto da feature.
- [x] Problema e motivação.
- [x] Público-alvo e cenários de uso.
- [x] Objetivos e métricas de sucesso — **≥1 objetivo com métrica e meta quantitativa**. (2 objetivos; Objetivo 1 com meta quantitativa ≤10s, marcada como "proposta")
- [x] Escopo (incluso e fora de escopo) — seção "Fora de escopo" com **≥2 itens** explicitamente descartados/adiados na reunião. (5 itens)
- [x] Requisitos funcionais — **≥8 requisitos** discutidos na reunião. (11 requisitos)
- [x] Requisitos não funcionais. (7)
- [x] Decisões e trade-offs principais.
- [x] Dependências.
- [x] Riscos e mitigação — **≥2 riscos**, cada um com probabilidade, impacto e mitigação. (5 riscos, todos com os 3 campos)
- [x] Critérios de aceitação.
- [x] Estratégia de testes e validação.

## Fase 6 — Tracker (`docs/TRACKER.md`)

- [x] Montar a tabela no formato obrigatório: `ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização`.
- [x] Varrer PRD, RFC, FDD e cada ADR extraindo os itens rastreáveis. (pacote completo coberto — 102 linhas)
- [x] Cobertura ≥80% dos itens identificáveis nos documentos. (100% dos itens identificáveis em PRD+RFC+FDD+ADRs)
- [x] ≥70% das linhas com Fonte = `TRANSCRICAO` e timestamp válido no formato `[hh:mm] Nome`. (84/102 = ~82%)
- [x] ≥5 linhas com Fonte = `CODIGO` e caminho de arquivo real. (18 linhas, todos os caminhos conferidos com `find`)
- [x] Regra prática: se não der pra preencher "Localização" com uma origem concreta, o item provavelmente foi inventado — ajustar ou remover do documento de origem. (aplicada: itens com sourcing incerto, ex. alguns bullets de "Consequências" nos ADRs e alguns códigos de erro do FDD sem grounding direto na transcrição, foram deixados de fora em vez de inventar timestamp)

## Fase 7 — README do processo (`README.md`)

Por último, quando o processo estiver completo.

- [x] **Sobre o desafio**: 1–2 parágrafos com suas palavras.
- [x] **Ferramentas de IA utilizadas**: lista com papel de cada uma. (3 itens: Claude Code, skills customizadas, CLAUDE.md)
- [x] **Workflow adotado**: ordem de produção dos documentos, como organizou a interação com a IA. (tabela com os 7 PRs de documentação)
- [x] **Prompts customizados**: ≥2 prompts relevantes, em blocos de código. (2 blocos: analyze-codebase e write-adrs)
- [x] **Iterações e ajustes**: ≥2 momentos concretos em que a IA errou/foi superficial e você corrigiu; quantas iterações principais até o resultado final. (6 correções concretas, 7 ciclos principais)
- [x] **Como navegar a entrega**: caminho dos arquivos e ordem sugerida de leitura. (11 itens, todos os links conferidos)
- [x] Opcional: manter link/seção referenciando o enunciado original (`EXERCICIO.md`). (link no topo do README)

## Fase 8 — Revisão final contra os critérios de aceite

Passar item a item pela checklist de "Critérios de Aceite" do `EXERCICIO.md` (seção completa) antes do push final. Pontos de atenção especial:

- [ ] Nenhum requisito/decisão/restrição nos documentos contradiz a transcrição ou o código.
- [ ] Nenhum arquivo de código citado nos documentos é inexistente no repositório real (conferir cada caminho citado com `ls`/busca).
- [ ] Itens explicitamente descartados na reunião NÃO aparecem como requisito em nenhum documento.
- [ ] Estrutura de pastas confere com a estrutura obrigatória do entregável.

## Fase 9 — Push final

- [ ] Commit e push para o fork público no GitHub.
- [ ] Conferir que o repositório está público.

---

## Log de Atividades

Registro cronológico das interações com a IA durante o desafio: quando, qual skill/prompt foi usado, e o que foi produzido. Serve de matéria-prima para as seções "Prompts customizados" e "Iterações e ajustes" do README (Fase 7).

| Data/Hora | Skill/Prompt | Arguments |
|---|---|---|
| 2026-07-18 12:22 | `/analyze-codebase` | Pedir à IA uma exploração do código (`src/`, `prisma/schema.prisma`) para mapear: módulos existentes (auth, users, customers, products, orders), máquina de estados do pedido, controle de estoque, auditoria de mudanças de status, padrões de erro/exceções usados. |
| 2026-07-18 12:43 | `/summarize-meeting-transcript` | resumo estruturado da transcrição, separando explicitamente: decisões fechadas, requisitos funcionais explícitos, restrições, pontos de integração com código existente, itens descartados, itens adiados para fases futuras, detalhes técnicos secundários. |
| 2026-07-19 08:29 | `/write-adrs` | This is an MBA assignment.<br><br>Your goal is to write the Architecture Decision Records (ADRs) required by the exercise.<br><br>Use the following documents in order of priority:<br><br>1. `EXERCICIO.md` — Defines the assignment requirements and grading criteria. Your ADRs must satisfy all of these requirements.<br>3. `RESUMO_TRANSCRICAO.md` — A summary of the transcription that can be used for navigation and context, but defer to `TRANSCRIPTION.md` whenever there is any ambiguity.<br>2. `TRANSCRIPTION.md` — The primary source of truth for the architectural decisions discussed. Do not invent decisions that are not supported by this document.<br>4. `ANALIZE_CODEBASE.md` — Provides additional context about the implementation and codebase. Use it to validate and enrich the ADRs where appropriate, but do not let it override the decisions documented in the transcription.<br><br>When writing the ADRs:<br>- Act as the software architect who participated in these decisions and is documenting them after the fact.<br>- Write professional ADRs that reflect real engineering decision-making, not an academic exercise.<br>- Ensure every ADR is traceable to evidence in the transcription or the codebase analysis.<br>- Do not fabricate rationale, alternatives, or consequences that are unsupported by the available information. If information is missing, explicitly state the assumption or limitation.<br>- The final output should read as production-quality architecture documentation while fully satisfying the exercise requirements. |
| 2026-07-19 10:51 | `/write-rfc` | This is an MBA assignment.<br><br>Your goal is to write the Request for Comments (RFC) required by the exercise.<br><br>Use the following documents in order of priority:<br><br>1. `EXERCICIO.md` — Defines the assignment requirements and grading criteria. The RFC must satisfy all of these requirements.<br>2. `TRANSCRIPTION.md` — The primary source of truth for the problem, context, discussions, and architectural reasoning. Do not invent information that is not supported by this document.<br>3. `RESUMO_TRANSCRICAO.md` — A summary of the transcription for quick reference. If there is any conflict or ambiguity, `TRANSCRIPTION.md` takes precedence.<br>4. `ADR-XXX-*.md` — The Architecture Decision Records that document the architectural decisions already made. Use them as the authoritative source for approved decisions.<br>5. `ANALIZE_CODEBASE.md` — Provides additional context about the existing implementation and codebase. Use it to validate and enrich the RFC where appropriate, but do not let it override the transcription or the ADRs.<br><br>When writing the RFC:<br>- Act as the software architect proposing the solution and requesting feedback from other engineers and stakeholders before implementation.<br>- Write a professional engineering RFC, not an academic report.<br>- Clearly explain the problem being solved, the proposed solution, the motivation behind it, the trade-offs considered, and the impact on the system.<br>- Use the ADRs as established architectural decisions. The RFC should build upon them rather than redefine them.<br>- Keep the discussion at the architectural level. Focus on **what** is being proposed and **why**, not **how** it will be implemented.<br>- Do not duplicate the level of implementation detail found in the FDD or other design documents.<br>- Do not fabricate rationale, alternatives, or consequences that are unsupported by the available information. If information is missing, explicitly state the assumption or limitation.<br>- The final document should read as a production-quality RFC that could be circulated internally for review and feedback. |
| 2026-07-19 11:29 | `/write-tracker` | This is an MBA assignment.<br><br>Your goal is to create the `docs/TRACKER.md` file required by the exercise.<br><br>The tracker is a cross-reference document that ensures all documented decisions, requirements, constraints, and other important items can be traced back to their original source. Its purpose is to prevent unsupported assumptions or AI hallucinations by proving where each documented item originated.<br><br>Use the following documents as input:<br><br>1. `EXERCICIO.md` — Defines the assignment requirements and the expected tracker format. The final tracker must satisfy all requirements described here.<br>2. `TRANSCRIPTION.md` — The primary source of truth for discussions, requirements, decisions, trade-offs, and constraints. When documenting items originating from conversations, always reference this source.<br>3. `RESUMO_TRANSCRICAO.md` — A navigation aid and summary of the transcription. Use it for context, but defer to `TRANSCRIPTION.md` whenever details are unclear.<br>4. `PRD.md`, `RFC.md`, `FDD.md`, `ADR-XXX-*.md` — The documents whose items must be mapped in the tracker.<br>5. `ANALIZE_CODEBASE.md` — Provides additional context about the implementation and existing code. Use it to identify items originating from the codebase.<br>6. The codebase itself — Use it as the source when validating implementation details, file paths, constraints, or existing behavior.<br><br>When writing the tracker:<br>- Act as the documentation owner responsible for maintaining traceability between requirements, decisions, and their evidence.<br>- Do not invent tracker entries that cannot be traced to a document and a source.<br>- Every tracked item must have a clear origin:<br>  - `TRANSCRICAO` when the information comes from discussions in `TRANSCRIPTION.md`.<br>  - `CODIGO` when the information comes from the existing implementation/codebase.<br>- Prioritize accuracy over completeness. If an item cannot be confidently traced, do not include it.<br>- Cover at least 80% of identifiable items present in the documentation (`PRD.md`, `RFC.md`, `FDD.md`, and ADRs).<br>- Include requirements, decisions, constraints, trade-offs, and other relevant architectural or product items.<br><br>The output must be exactly a Markdown table following this structure:<br><br>\| ID \| Documento \| Tipo \| Conteúdo (resumo) \| Fonte \| Localização \|<br>\| --- \| --- \| --- \| --- \| --- \| --- \|<br>\| \| \| \| \| \| \|<br><br>Column requirements:<br>- **ID**: A unique identifier for the tracked item (examples: `PRD-FR-01`, `RFC-ALT-02`, `FDD-CONTRATO-03`, `ADR-002`).<br>- **Documento**: The document where the item is documented (examples: `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/adrs/ADR-002-*.md`).<br>- **Tipo**: The category of the item, such as:<br>  - Requisito Funcional<br>  - Requisito Não Funcional<br>  - Decisão<br>  - Restrição<br>  - Trade-off<br>  - Assumption<br>  - Other relevant categories<br>- **Conteúdo (resumo)**: A one-line summary of the item being tracked.<br>- **Fonte**:<br>  - `TRANSCRICAO` for items supported by the transcription.<br>  - `CODIGO` for items supported by the implementation/codebase.<br>- **Localização**:<br>  - For `TRANSCRICAO`: include timestamp and speaker name (example: `[09:17] Diego`).<br>  - For `CODIGO`: include the source file path (example: `src/modules/orders/order.service.ts`).<br><br>The final tracker should be a professional traceability artifact that allows a reviewer to understand where every important documented item came from.<br>, finally instruction as I already created the ADR, AND RFC the tracker should include them already |
| 2026-07-19 11:51 | `/write-fdd` | This is an MBA assignment.<br><br>Your goal is to write the Feature Design Document (FDD) required by the exercise.<br><br>Use the following documents in order of priority:<br><br>1. `EXERCICIO.md` — Defines the assignment requirements and grading criteria. The FDD must satisfy all of these requirements.<br>2. `TRANSCRIPTION.md` — The primary source of truth for the problem, context, discussions, constraints, and architectural reasoning. Do not invent technical decisions or implementation details that are not supported by this document.<br>3. `RESUMO_TRANSCRICAO.md` — A summary of the transcription for quick reference. If there is any conflict or ambiguity, `TRANSCRIPTION.md` takes precedence.<br>4. `ADR-XXX-*.md` — The Architecture Decision Records that document the architectural decisions already made. Use them as the foundation and constraints for the implementation design.<br>5. `RFC.md` — The architecture-level proposal that explains what is being proposed and why. Use it as the basis for describing how the feature will be implemented.<br>6. `ANALIZE_CODEBASE.md` — Provides additional context about the existing implementation and codebase. Use it to validate and enrich the FDD where appropriate, but do not let it override the transcription, ADRs, or RFC.<br>7. `TRACKER.md` — Reference of what has already been documented and completed.<br><br>When writing the FDD:<br>- Act as the software engineer responsible for designing the implementation after the architecture has been approved.<br>- Write a professional Feature Design Document that reflects real engineering design, not an academic exercise.<br>- Focus on **how the feature will be implemented**, not why the architecture was chosen.<br>- Do not duplicate the level of abstraction from the RFC. The RFC answers "what we propose and why"; the FDD answers "how we will build it".<br>- Make the document actionable enough that a developer can use it as a starting point for implementation.<br>- Reuse existing codebase patterns, abstractions, and conventions whenever possible.<br>- Reference real files, classes, methods, and modules from the codebase when describing integrations.<br>- Do not fabricate implementation details, APIs, files, classes, or behaviors that are not supported by the available information. If information is missing, explicitly state the assumption or limitation.<br><br>Process requirements:<br>- Update `TRACKER.md`.<br>- Update `PLANO_DE_TRABALHO.md`.<br>- Use consistent PRs. |
| 2026-07-19 12:23 | `/write-fdd` (re-run após generalização da skill) | I updated the skill as it was NOT generic enough.<br><br>It need to be run again<br><br>/write-fdd<br><br>[mesmos argumentos da execução anterior — ver linha 2026-07-19 11:51] |
| 2026-07-19 12:58 | `/write-prd` | This is an MBA assignment.<br><br>Your goal is to write the Product Requirements Document (PRD) required by the exercise.<br><br>Use the following documents in order of priority:<br><br>1. `EXERCICIO.md` — Defines the assignment requirements and grading criteria. The PRD must satisfy all of these requirements.<br>2. `TRANSCRIPTION.md` — The primary source of truth for the business problem, requirements, context, constraints, and expected outcomes. Do not invent product requirements that are not supported by this document.<br>3. `RESUMO_TRANSCRICAO.md` — A summary of the transcription for quick reference. If there is any conflict or ambiguity, `TRANSCRIPTION.md` takes precedence.<br>4. `ANALIZE_CODEBASE.md` — Provides additional context about the existing implementation and current system capabilities. Use it to validate assumptions and identify constraints, but do not let it override the transcription.<br>5. `ADR-XXX-*.md` — Reference of architectural decisions already documented. Use them only to ensure consistency with the proposed solution.<br>6. `RFC.md` — Reference of the architecture-level proposal. Use it only to ensure the documented requirements remain aligned with the proposed solution.<br>7. `FDD.md` — Reference of the implementation design. Use it only to ensure consistency between the requirements and the implementation details.<br>8. `TRACKER.md` — Reference of what has already been documented and completed.<br><br>When writing the PRD:<br>- Act as the product owner responsible for defining the feature requirements.<br>- Write a professional Product Requirements Document that reflects real product discovery, not an academic exercise.<br>- Focus on **what** the feature should accomplish and **why** it is needed.<br>- Avoid implementation details, architectural decisions, and technical design unless they are necessary to define product constraints.<br>- Do not fabricate requirements, business rules, or acceptance criteria that are not supported by the available information. If information is missing, explicitly state the assumption or limitation.<br><br>Process requirements:<br>- Update `TRACKER.md`.<br>- Update `PLANO_DE_TRABALHO.md`.<br>- Use consistent PRs. |
| 2026-07-19 13:49 | (prompt direto, sem skill — Fase 7: README) | Sem skill dedicada; três pedidos diretos em sequência no Claude Code, colando o checklist da Fase 7 do `EXERCICIO.md`: (1) pedido inicial passando a checklist de "README do processo" (Sobre o desafio, Ferramentas de IA, Workflow adotado, Prompts customizados, Iterações e ajustes, Como navegar a entrega) para gerar `README.md`; (2) pedido explicando o workflow real usado para criar as skills e os prompts de execução (Perplexity para desenhar/generalizar cada skill, ChatGPT para organizar o prompt de execução antes de rodar no Claude, decisão deliberada de usar 3 IAs para não depender de um único modelo), com instrução para incorporar isso ao README; (3) pedido detalhando a ordem real seguida (PLANO_DE_TRABALHO.md gerado primeiro, antes de qualquer documento; RESUMO_TRANSCRICAO.md com pouco efeito prático porque os prompts das skills sempre citavam TRANSCRICAO.md como fonte principal; Tracker construído antes do FDD, fora da ordem sugerida no enunciado, para informar a geração dos documentos seguintes; fases a partir do README executadas com prompt direto, sem pipeline de 3 IAs), com instrução para atualizar a seção "Workflow adotado" com essas informações. |
| 2026-07-19 14:17 | (prompt direto, sem skill — Fase 8: revisão final, sessão separada com Claude Haiku 4.5) | I need you got on the file PLANO_DE_TRABAHO.md at the session Fase 8, and execute the verification if everything that the EXERCICIO.md request is covered in the correct format.<br><br>This is the sinal check before I submit my asignemt for this subject at my MBA |

## Notas de processo

- É esperado 3 a 5 ciclos de geração → revisão crítica → ajuste de prompt → nova geração por documento. Gerar tudo de primeira sem ajustes é sinal de documento genérico demais.
- Prompts vagos ("gere um PRD a partir dessa transcrição") tendem a produzir documentos vazios. Usar prompts dirigidos, citando seções específicas da transcrição/código.
- O tracker é o principal mecanismo de defesa contra alucinação — pode valer a pena começar a esboçar linhas do tracker à medida que cada documento é escrito, em vez de deixar tudo pro fim.
