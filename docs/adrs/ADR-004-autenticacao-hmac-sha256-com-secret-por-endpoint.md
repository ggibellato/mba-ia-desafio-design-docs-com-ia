# Autenticação HMAC-SHA256 com secret por endpoint

- Status: Aceito
- Data: 2026-07-19 (data de registro deste ADR; a decisão foi tomada na reunião técnica descrita em `TRANSCRICAO.md`)
- Deciders: Sofia (Engenharia de Segurança), Diego (Engenheiro Sênior, Plataforma)
- Relacionado: `TRANSCRICAO.md` [09:19–09:22]

## Status

Aceito.

## Contexto

Ao entrar na parte de segurança da reunião, Sofia levantou que o sistema passará a expor eventos com dados de pedidos para endpoints fora da infraestrutura da empresa, e que o cliente precisa conseguir validar que a requisição realmente veio da plataforma e que o payload não foi adulterado em trânsito (`[09:19]`). Ela propôs o padrão HMAC: assinar o payload com uma secret compartilhada entre a plataforma e o cliente, enviando a assinatura em um header (`X-Signature`), com o cliente verificando a assinatura do lado dele. Questionada sobre o algoritmo, Sofia especificou HMAC-SHA256 por ser padrão de mercado, com suporte em praticamente qualquer biblioteca cliente (`[09:20]`).

Sofia também exigiu que cada endpoint de webhook cadastrado por um cliente tenha uma secret única — não uma secret global da plataforma — para que o vazamento de uma secret não comprometa todos os clientes (`[09:21]`). Bruno confirmou que a tabela de configuração de webhook armazenaria `url`, `secret`, `customer_id` e estado ativo. Sofia acrescentou que a secret precisa ser rotacionável via API e que, durante a rotação, a secret antiga permanece válida por 24 horas em paralelo com a nova, dando tempo ao cliente de migrar seus sistemas (`[09:21]`). Diego reforçou a importância dessa exigência citando um incidente real: um cliente já vazou uma secret em log de aplicação (`[09:22]`).

## Decisão

Assinar o corpo de cada requisição de webhook com HMAC-SHA256, usando uma secret exclusiva por endpoint de webhook cadastrado (armazenada junto com `url`, `customer_id` e estado ativo/inativo na configuração do webhook). A assinatura é enviada no header `X-Signature`. A secret pode ser rotacionada pelo cliente via API; durante a rotação, a secret anterior permanece válida por um grace period de 24 horas antes de ser definitivamente invalidada.

## Alternativas Consideradas

- **Secret global única para toda a plataforma.** Rejeitada explicitamente por Sofia: o vazamento de uma única secret comprometeria a autenticidade de todos os webhooks de todos os clientes simultaneamente (`[09:21]`).
- **Rotação de secret sem grace period (troca imediata, sem período de validade dupla).** Implicitamente descartada em favor do grace period de 24h: uma troca imediata quebraria a integração do cliente no exato momento da rotação, sem janela para ele atualizar sua configuração (`[09:21]`).

## Consequências

### Positivas

- Cada cliente pode verificar de forma independente que um webhook recebido realmente veio da plataforma e não foi adulterado em trânsito.
- O isolamento de secret por endpoint limita o raio de impacto de um vazamento a um único cliente/endpoint, em vez de comprometer a plataforma inteira.
- O grace period de 24h permite rotação de secret sem downtime de integração para o cliente.

### Negativas

- Exige que cada cliente implemente corretamente a verificação HMAC do lado dele; falhas de implementação no cliente (comuns em integrações de terceiros) podem gerar rejeições ou, pior, aceitação de payloads não verificados.
- Durante o grace period de 24h, duas secrets ficam simultaneamente válidas para o mesmo endpoint, ampliando levemente a janela de exposição em caso de vazamento da secret antiga.
- Armazenar e gerenciar uma secret por endpoint (em vez de uma única secret de plataforma) aumenta a superfície de dados sensíveis a proteger no banco.
