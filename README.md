# Da Reunião ao Documento: Design Docs Gerados por IA

Este README documenta o **processo** que usei para produzir o pacote de design docs deste desafio. O enunciado original está preservado em [`EXERCICIO.md`](./EXERCICIO.md).

## Sobre o desafio

O desafio parte de uma situação bem específica: uma reunião técnica de ~55 minutos entre tech lead, PM, dois engenheiros e segurança decidiu como construir um Sistema de Webhooks de Notificação de Pedidos para um Order Management System (OMS) já existente em produção — mas nada disso ficou registrado além da gravação literal da call (`TRANSCRICAO.md`). O trabalho era transformar essa transcrição, mais o código-fonte do OMS, em um pacote completo de documentação técnica (PRD, RFC, FDD, ADRs e um Tracker de rastreabilidade) acionável o suficiente para um time de engenharia começar a implementar.

A restrição mais importante do desafio não é técnica, é de disciplina: toda informação registrada nos documentos precisa ser rastreável até a transcrição ou até um arquivo real do código. Não é permitido inventar requisito, decisão ou restrição — e identificar o que a reunião **descartou ou adiou** explicitamente é tão importante quanto identificar o que ela decidiu. Usei a IA (Claude Code) como ferramenta principal de produção, mas o trabalho de verificar cada afirmação contra a fonte original — e corrigir o que a IA errou ou generalizou demais — foi manual, item por item.

## Ferramentas de IA utilizadas

Usei três IAs diferentes, cada uma com um papel específico no pipeline — decisão deliberada para não depender de um único modelo/contexto e chegar a um resultado mais genérico e robusto, em vez de deixar uma única IA "viciar" a skill e o prompt no contexto específico desta conversa:

- **Perplexity** (https://www.perplexity.ai/) — usado para **desenhar cada skill** (`.claude/skills/*/SKILL.md`). Para cada documento, enviei um prompt descrevendo o objetivo e colando os requisitos mínimos extraídos do `EXERCICIO.md`. Depois de algumas iterações simples corrigindo detalhes — o problema mais recorrente era a skill sair específica demais para este projeto — cheguei a uma versão genérica, reutilizável em outro projeto/feature, não amarrada a "webhooks".
- **ChatGPT** (https://chatgpt.com/) — usado para **organizar o prompt de execução** antes de rodar cada skill. Com prompts simples, montava e revisava o texto que depois seria colado como argumento do slash command no Claude (ordem de prioridade das fontes, regras específicas do desafio), garantindo que o pedido para o Claude chegasse já estruturado.
- **Claude Code** (CLI agêntico, rodando Claude Sonnet 5) — ferramenta de **execução final**. Recebe a skill (vinda do Perplexity) e o prompt já organizado (vindo do ChatGPT) como argumento do slash command, lê a transcrição e o código-fonte diretamente do repositório, executa comandos de shell para verificar fatos (`grep`, `find`, leitura de arquivos com números de linha) antes de citá-los nos documentos, produz o arquivo final, e geriu todo o fluxo de branches/commits/PRs via `gh` CLI.
- **`CLAUDE.md`** na raiz do repositório — funciona como um "prompt de sistema" do projeto: força saída em português brasileiro por padrão, define estilo de escrita e a ordem de precedência das regras, e vale para todas as skills executadas pelo Claude Code.

## Workflow adotado

**Passo 0 — plano de trabalho antes de qualquer documento.** O primeiro pedido que fiz à IA não foi um documento do pacote, foi o `PLANO_DE_TRABALHO.md` (não é entregável do desafio): um roteiro fase a fase, com checklist do que já estava feito e uma tabela "Log de Atividades" registrando, para cada invocação de skill, o horário e o prompt exato usado. Foi o que tornou possível reconstruir com precisão as seções deste README.

**Contextualização (Fase 1).** Em seguida gerei `ANALIZE_CODEBASE.md` (skill `analyze-codebase`) e `RESUMO_TRANSCRICAO.md` (skill `summarize-meeting-transcript`). A intenção do resumo era ter uma forma reduzida da transcrição para referência rápida, mantendo `TRANSCRICAO.md` como fonte da verdade em caso de dúvida ou ambiguidade. Na prática, isso não deu muito fruto: os prompts usados para executar as skills seguintes sempre listavam `TRANSCRICAO.md` como fonte principal, e o resumo acabou funcionando mais como material de apoio para eu revisar manualmente do que como algo que a IA de fato priorizou.

**ADRs → RFC → Tracker → FDD → PRD, com um desvio deliberado da ordem sugerida.** Segui a ordem do enunciado (ADRs, depois RFC, depois FDD e PRD), com uma exceção: construí o Tracker logo depois do RFC, antes do FDD — o enunciado sugere montá-lo em paralelo ou só no fim. Decidi adiantar porque achei que saber o que já estava documentado/rastreado ajudaria a geração dos documentos seguintes (FDD e PRD), em vez de descobrir só depois lacunas de rastreabilidade que exigiriam retrabalho nos dois.

**Pipeline de 3 IAs para cada um desses documentos.** Para ADRs, RFC, Tracker, FDD e PRD, usei o mesmo processo de 3 etapas descrito em "Ferramentas de IA utilizadas": Perplexity desenha/ajusta a skill, ChatGPT organiza o prompt de execução, Claude Code roda a skill com esse prompt e produz o arquivo.

**Fases finais (README e revisão) — abordagem mais direta.** A partir do README.md (Fase 7) e da revisão final contra os critérios de aceite (Fase 8), não criei skills novas nem passei pelo pipeline Perplexity/ChatGPT — pedi essas etapas direto no Claude Code, com prompts simples descrevendo cada checklist. Fazia sentido nesse ponto: essas fases consolidam e revisam o que já existe, não geram um novo tipo de documento estruturado que justificasse uma skill reutilizável.

Cada etapa virou um commit em uma branch própria e um Pull Request revisado antes do merge — nunca commitei direto em `main`:

| PR | Conteúdo |
| --- | --- |
| #2 | `PLANO_DE_TRABALHO.md` (Passo 0 — plano/roteiro pessoal, não é entregável) |
| #3 | Skill `analyze-codebase` + `ANALIZE_CODEBASE.md` (mapa do código existente) |
| #4 | Skill `summarize-meeting-transcript` + `RESUMO_TRANSCRICAO.md` (resumo estruturado da reunião) |
| #5 | Skill `write-adrs` + 7 ADRs em `docs/adrs/` |
| #6 | Skill `write-rfc` + `docs/RFC.md` |
| #7 | Skill `write-tracker` + primeira versão de `docs/TRACKER.md` (ADRs + RFC) |
| #8 | Skill `write-fdd` + `docs/FDD.md`, Tracker estendido |
| #9 | Skill `write-prd` + `docs/PRD.md`, Tracker estendido, cobertura ≥80% fechada |

Cada PR passou por uma verificação factual antes do merge: conferir se cada caminho de arquivo citado existe de fato (`find`/`ls`), se os números de linha citados batem com o arquivo real, e se nenhum item explicitamente descartado na reunião apareceu como requisito.

## Prompts customizados

Os três exemplos abaixo mostram o pipeline completo (Perplexity → ChatGPT → Claude Code) para um mesmo documento, o PRD.

**1. Perplexity — pedido para desenhar a skill `write-prd`, colando os requisitos mínimos direto do `EXERCICIO.md`:**

```
now I need a skill to write a PRD.md

This is the requirements

deve ser consolidação do que já foi decidido em ADRs/RFC/FDD.

- [ ] Resumo e contexto da feature.
- [ ] Problema e motivação.
- [ ] Público-alvo e cenários de uso.
- [ ] Objetivos e métricas de sucesso — **≥1 objetivo com métrica e meta quantitativa**.
- [ ] Escopo (incluso e fora de escopo) — seção "Fora de escopo" com **≥2 itens**
      explicitamente descartados/adiados na reunião.
- [ ] Requisitos funcionais — **≥8 requisitos** discutidos na reunião.
- [ ] Requisitos não funcionais.
- [ ] Decisões e trade-offs principais.
- [ ] Dependências.
- [ ] Riscos e mitigação — **≥2 riscos**, cada um com probabilidade, impacto e mitigação.
- [ ] Critérios de aceitação.
- [ ] Estratégia de testes e validação.
```

Esse prompt passou por algumas iterações simples no próprio Perplexity — o problema mais comum era a skill sair específica demais para este projeto — até chegar numa versão genérica, reutilizável em outro projeto/feature.

**2. ChatGPT — organização do prompt de execução daquela skill, antes de colar no Claude:**

```
/write-prd

This is an MBA assignment.

Your goal is to write the Product Requirements Document (PRD) required by the exercise.

Use the following documents in order of priority:

1. `EXERCICIO.md` — Defines the assignment requirements and grading criteria. The FDD must satisfy all of these requirements.
2. `TRANSCRIPTION.md` — The primary source of truth for the problem, context, discussions, constraints, and expected outcomes. Do not invent product requirements that are not supported by this document.
3. `RESUMO_TRANSCRICAO.md` — A summary of the transcription for quick reference. If there is any conflict or ambiguity, `TRANSCRIPTION.md` takes precedence.
4. `ADR-XXX-*.md` — Reference of architectural decisions already documented. Use them only to ensure consistency with the proposed solution.
5. `RFC.md` — Reference of the architecture-level proposal. Use it only to ensure the documented requirements remain aligned with the proposed solution.
6. `FDD.md` - Product Requirements Document
6. `ANALIZE_CODEBASE.md` — Provides additional context about the existing implementation and codebase. Use it to validate and enrich the FDD where appropriate, but do not let it override the transcription, ADRs, or RFC.
7. `TRACKER.md` — Reference of what has already been documented and completed.
```

(Mantido como foi realmente usado, inclusive a numeração duplicada em "6." — o Claude recebeu exatamente esse texto como argumento do slash command.)

**3. Claude Code — execução (`/write-prd`) com o prompt acima como argumento**, produzindo `docs/PRD.md` a partir da transcrição, do código e dos documentos já existentes. O texto completo, incluindo as instruções de "When writing the PRD" e "Process requirements" que também fazem parte desse prompt, está registrado verbatim em `PLANO_DE_TRABALHO.md` (tabela "Log de Atividades"), junto com os prompts equivalentes das outras 6 skills.

## Iterações e ajustes

O processo levou **7 ciclos principais** (um por documento/skill de produção, PRs #3 a #9 na tabela acima), e dentro deles, pelo menos **6 correções concretas** que exigiram voltar e ajustar algo já gerado:

1. **Erro factual de código em um ADR.** Um dos ADRs afirmou que `requireRole('ADMIN')` restringe hoje a *criação* de usuários. Ao pedir uma segunda checagem explícita de tudo que os ADRs afirmavam, a IA confirmou com `grep`/leitura direta que, na verdade, esse middleware protege `GET /users/:id` — não existe rota de criação de usuário nesse arquivo. Corrigido antes do merge.
2. **Redundância entre requisito funcional e não funcional no PRD.** `FR-06` ("URL do webhook deve ser https") e `NFR-03` ("apenas URLs https são aceitas") diziam essencialmente a mesma coisa com palavras diferentes. Ao questionar se FR e NFR não deveriam ser mutuamente exclusivos, reescrevi `NFR-03` para focar no atributo de qualidade que ele deveria ter (isolamento de secret por cliente), em vez de repetir o fato já coberto por `FR-06`.
3. **Skill genérica demais tratada como específica, depois generalizada.** A skill `write-fdd`, na primeira versão gerada no Perplexity, tinha o padrão `WEBHOOK_*` embutido como se fosse universal — funcionava bem para este desafio, mas não seria reutilizável em outro projeto. Depois de iterar no Perplexity para generalizá-la, atualizei o arquivo no repositório e pedi ao Claude Code para rodar de novo; em vez de regenerar o `docs/FDD.md` às cegas, o Claude auditou seção por seção e confirmou que o conteúdo continuava correto, porque `EXERCICIO.md` (prioridade 1) já fixa esses detalhes especificamente para este projeto.
4. **Tradução literal que não soava natural.** Uma frase técnica no FDD usava "lançar" como verbo intransitivo ("se o passo 3 lançar") — tradução literal de "throws" que não faz sentido em português sem objeto. Corrigido primeiro para "lançar uma exceção", depois trocado para o termo em inglês ("throw") por preferência de estilo, já que é vocabulário técnico comum no dia a dia de quem vai ler o documento.
5. **Log de prompts não fiel ao que foi realmente digitado.** A primeira versão da tabela "Log de Atividades" resumia/traduzia o prompt usado em vez de registrar o texto literal. Corrigido para guardar o argumento verbatim daquele ponto em diante — é o que tornou possível reproduzir os prompts customizados acima com exatidão.
6. **Skill não reconhecida por erro de nome de arquivo.** Uma skill nova (`write-prd`) não aparecia na lista após `/reload-skills`. O arquivo estava salvo como `write-prd.md` em vez do nome exigido pela convenção do projeto, `SKILL.md` (o único padrão usado por todas as outras skills). Renomeado e a skill passou a carregar normalmente.

## Como navegar a entrega

Ordem sugerida de leitura:

1. **[`README.md`](./README.md)** (este arquivo) — o processo.
2. **[`EXERCICIO.md`](./EXERCICIO.md)** — enunciado original do desafio, para contexto.
3. **[`TRANSCRICAO.md`](./TRANSCRICAO.md)** — fonte de verdade primária: a transcrição literal da reunião.
4. **[`RESUMO_TRANSCRICAO.md`](./RESUMO_TRANSCRICAO.md)** — resumo estruturado da transcrição (decisões, requisitos, restrições, itens descartados/adiados), útil para navegar sem reler os 55 minutos inteiros.
5. **[`ANALIZE_CODEBASE.md`](./ANALIZE_CODEBASE.md)** — mapa do código existente (módulos, máquina de estados do pedido, padrões de erro) usado como contexto pelos demais documentos.
6. **[`docs/adrs/`](./docs/adrs/)** — as 7 decisões arquiteturais, na ordem `ADR-001` a `ADR-007`; é o esqueleto sobre o qual o resto foi construído.
7. **[`docs/RFC.md`](./docs/RFC.md)** — a proposta técnica de arquitetura, com links de volta para os ADRs.
8. **[`docs/FDD.md`](./docs/FDD.md)** — a especificação de implementação: fluxos, contratos HTTP, matriz de erros, integração com o código existente.
9. **[`docs/PRD.md`](./docs/PRD.md)** — a consolidação em linguagem de produto (o quê e por quê), construída por último entre os grandes documentos.
10. **[`docs/TRACKER.md`](./docs/TRACKER.md)** — a matriz de rastreabilidade: 102 linhas ligando cada item dos documentos acima à transcrição ou ao código.
11. **[`PLANO_DE_TRABALHO.md`](./PLANO_DE_TRABALHO.md)** — não é um entregável do desafio, mas registra o histórico completo de execução, incluindo o prompt exato usado em cada uma das 8 invocações de skill.
