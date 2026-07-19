# Da Reunião ao Documento: Design Docs Gerados por IA

Este README documenta o **processo** que usei para produzir o pacote de design docs deste desafio. O enunciado original está preservado em [`EXERCICIO.md`](./EXERCICIO.md).

## Sobre o desafio

O desafio parte de uma situação bem específica: uma reunião técnica de ~55 minutos entre tech lead, PM, dois engenheiros e segurança decidiu como construir um Sistema de Webhooks de Notificação de Pedidos para um Order Management System (OMS) já existente em produção — mas nada disso ficou registrado além da gravação literal da call (`TRANSCRICAO.md`). O trabalho era transformar essa transcrição, mais o código-fonte do OMS, em um pacote completo de documentação técnica (PRD, RFC, FDD, ADRs e um Tracker de rastreabilidade) acionável o suficiente para um time de engenharia começar a implementar.

A restrição mais importante do desafio não é técnica, é de disciplina: toda informação registrada nos documentos precisa ser rastreável até a transcrição ou até um arquivo real do código. Não é permitido inventar requisito, decisão ou restrição — e identificar o que a reunião **descartou ou adiou** explicitamente é tão importante quanto identificar o que ela decidiu. Usei a IA (Claude Code) como ferramenta principal de produção, mas o trabalho de verificar cada afirmação contra a fonte original — e corrigir o que a IA errou ou generalizou demais — foi manual, item por item.

## Ferramentas de IA utilizadas

- **Claude Code** (CLI agêntico, rodando Claude Sonnet 5) — ferramenta principal de produção. Leu a transcrição e o código-fonte diretamente do repositório, executou comandos de shell para verificar fatos (`grep`, `find`, leitura de arquivos com números de linha) antes de citá-los nos documentos, e geriu todo o fluxo de branches/commits/PRs via `gh` CLI.
- **Skills customizadas** (`.claude/skills/`, invocadas como slash commands) — um prompt reutilizável e estruturado por tipo de documento, cada um com regras explícitas de anti-alucinação ("não invente caminho de código", "priorize precisão sobre completude", "se a origem for ambígua, deixe de fora em vez de inventar"): `analyze-codebase`, `summarize-meeting-transcript`, `write-adrs`, `write-rfc`, `write-fdd`, `write-prd`, `write-tracker`.
- **`CLAUDE.md`** na raiz do repositório — funciona como um "prompt de sistema" do projeto: força saída em português brasileiro por padrão, define estilo de escrita e a ordem de precedência das regras, e vale para todas as skills acima.

## Workflow adotado

Segui a ordem sugerida no enunciado (ADRs → RFC → FDD → PRD → Tracker → README), mas com duas adaptações práticas:

1. **Um plano de trabalho pessoal** (`PLANO_DE_TRABALHO.md`, não é entregável do desafio) — um roteiro fase a fase com checklist e uma tabela "Log de Atividades" registrando, para cada invocação de skill, o horário e o prompt exato usado. Foi o que tornou possível reconstruir com precisão as seções abaixo.
2. **Tracker construído incrementalmente, não só no fim** — em vez de varrer todos os documentos prontos de uma vez, atualizei `docs/TRACKER.md` a cada novo documento: ADRs + RFC juntos (48 linhas), depois FDD (+25 → 73), depois PRD (+29 → 102 no total), o que expôs cedo qualquer afirmação sem origem clara.

Cada etapa virou um commit em uma branch própria e um Pull Request revisado antes do merge — nunca commitei direto em `main`:

| PR | Conteúdo |
| --- | --- |
| #3 | Skill `analyze-codebase` + `ANALIZE_CODEBASE.md` (mapa do código existente) |
| #4 | Skill `summarize-meeting-transcript` + `RESUMO_TRANSCRICAO.md` (resumo estruturado da reunião) |
| #5 | Skill `write-adrs` + 7 ADRs em `docs/adrs/` |
| #6 | Skill `write-rfc` + `docs/RFC.md` |
| #7 | Skill `write-tracker` + primeira versão de `docs/TRACKER.md` (ADRs + RFC) |
| #8 | Skill `write-fdd` + `docs/FDD.md`, Tracker estendido |
| #9 | Skill `write-prd` + `docs/PRD.md`, Tracker estendido, cobertura ≥80% fechada |

Cada PR passou por uma verificação factual antes do merge: conferir se cada caminho de arquivo citado existe de fato (`find`/`ls`), se os números de linha citados batem com o arquivo real, e se nenhum item explicitamente descartado na reunião apareceu como requisito.

## Prompts customizados

Dois exemplos representativos — o primeiro é um prompt curto e dirigido; o segundo é um prompt longo, com ordem de prioridade explícita entre fontes e regras anti-alucinação, no estilo dos "prompts do professor" mencionados no enunciado.

**1. Exploração inicial do código (`/analyze-codebase`):**

```
Pedir à IA uma exploração do código (`src/`, `prisma/schema.prisma`) para mapear:
módulos existentes (auth, users, customers, products, orders), máquina de estados
do pedido, controle de estoque, auditoria de mudanças de status, padrões de
erro/exceções usados.
```

**2. Geração dos ADRs (`/write-adrs`), com ordem de prioridade entre fontes:**

```
This is an MBA assignment.

Your goal is to write the Architecture Decision Records (ADRs) required by the exercise.

Use the following documents in order of priority:

1. `EXERCICIO.md` — Defines the assignment requirements and grading criteria.
   Your ADRs must satisfy all of these requirements.
2. `TRANSCRICAO.md` — The primary source of truth for the architectural decisions
   discussed. Do not invent decisions that are not supported by this document.
3. `RESUMO_TRANSCRICAO.md` — A summary of the transcription that can be used for
   navigation and context, but defer to TRANSCRICAO.md whenever there is ambiguity.
4. `ANALIZE_CODEBASE.md` — Provides additional context about the implementation
   and codebase. Use it to validate and enrich the ADRs, but do not let it
   override the decisions documented in the transcription.

When writing the ADRs:
- Act as the software architect who participated in these decisions and is
  documenting them after the fact.
- Ensure every ADR is traceable to evidence in the transcription or the
  codebase analysis.
- Do not fabricate rationale, alternatives, or consequences that are
  unsupported by the available information.
```

A `docs/TRACKER.md` na raiz do desafio guarda mais 6 prompts equivalentes (um por skill), com o texto exato usado em cada execução — inclui também um caso em que corrigi a própria skill no meio do processo (ver "Iterações e ajustes" abaixo).

## Iterações e ajustes

O processo levou **7 ciclos principais** (um por documento/skill de produção, PRs #3 a #9 na tabela acima), e dentro deles, pelo menos **6 correções concretas** que exigiram voltar e ajustar algo já gerado:

1. **Erro factual de código em um ADR.** Um dos ADRs afirmou que `requireRole('ADMIN')` restringe hoje a *criação* de usuários. Ao pedir uma segunda checagem explícita de tudo que os ADRs afirmavam, a IA confirmou com `grep`/leitura direta que, na verdade, esse middleware protege `GET /users/:id` — não existe rota de criação de usuário nesse arquivo. Corrigido antes do merge.
2. **Redundância entre requisito funcional e não funcional no PRD.** `FR-06` ("URL do webhook deve ser https") e `NFR-03` ("apenas URLs https são aceitas") diziam essencialmente a mesma coisa com palavras diferentes. Ao questionar se FR e NFR não deveriam ser mutuamente exclusivos, reescrevi `NFR-03` para focar no atributo de qualidade que ele deveria ter (isolamento de secret por cliente), em vez de repetir o fato já coberto por `FR-06`.
3. **Skill genérica demais tratada como específica, depois generalizada.** A skill `write-fdd`, na primeira versão, tinha o padrão `WEBHOOK_*` embutido como se fosse universal — funcionava bem para este desafio, mas não seria reutilizável em outro projeto. Depois de generalizá-la, pedi para rodar de novo; em vez de regenerar o `docs/FDD.md` às cegas, a IA auditou seção por seção e confirmou que o conteúdo continuava correto, porque `EXERCICIO.md` (prioridade 1) já fixa esses detalhes especificamente para este projeto.
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
