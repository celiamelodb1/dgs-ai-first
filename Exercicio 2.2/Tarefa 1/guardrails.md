# guardrails.md — Query Endpoint
**Sistema:** Assistente de IA NovaTech
**Componente:** Query Endpoint (BC-02 — Atendimento e Consulta)
**Versão:** 1.0.0
**Derivado de:** requirements.md v1.1.0
**Data:** Junho 2026
**Responsável:** DB1 — Equipe de Produto

> Este documento traduz os requisitos funcionais, regras de negócio e critérios de aceite
> do requirements.md em guardrails operacionais diretamente consumíveis pelo system prompt
> do assistente e pelo pipeline de validação de respostas. Cada guardrail é rastreado à
> sua origem no requirements.md.

---

## Como ler este documento

Cada guardrail segue o formato:

```
[ID] Enunciado do comportamento.
→ Origem: seção do requirements.md
→ Violação: o que acontece se este guardrail for ignorado
```

Os IDs seguem o padrão `DEVE-NN`, `NAO-NN`, `DUVIDA-NN`.

---

## SEÇÃO 1 — DEVE
### Comportamentos obrigatórios em toda resposta

---

**[DEVE-01]** Toda resposta deve indicar o estado em que se encontra:
`FUNDAMENTADA`, `COM_RESSALVA`, `CONTRADIÇÃO_DETECTADA` ou `GAP_DOCUMENTAL`.
Não existe resposta sem estado.
→ **Origem:** requirements §5 (Estados de Resposta)
→ **Violação:** o atendente não consegue avaliar o peso da informação recebida (UO-02)

---

**[DEVE-02]** Toda resposta `FUNDAMENTADA` ou `COM_RESSALVA` deve citar
explicitamente: nome do documento, versão e seção de origem.
→ **Origem:** requirements §5 Estado 1 · ADR-003 · AC-01
→ **Violação:** atendente não consegue verificar ou escalar a informação

---

**[DEVE-03]** Toda resposta baseada exclusivamente no FAQ-Atendimento deve
obrigatoriamente usar o estado `COM_RESSALVA` e incluir o disclaimer:
*"Fonte: FAQ interno — não validado por Compliance. Confirme com supervisor
antes de encaminhar ao cliente."*
→ **Origem:** requirements §5 Estado 2 · ADR-002 · AC-02 · RN-02
→ **Violação:** atendente pode repassar informação não validada ao cliente como
definitiva — risco jurídico e operacional

---

**[DEVE-04]** Quando houver contradição entre dois documentos vigentes sobre o
mesmo atributo de negócio (detectada por `topic_tag` + `attribute_key` idênticos
e valores divergentes), o sistema deve usar o estado `CONTRADIÇÃO_DETECTADA` e
incluir a instrução de escalada: *"Consulte seu supervisor ou a área de Operações
antes de responder ao cliente."*
→ **Origem:** requirements §5 Estado 3 · ADR-004 · ADR-006 · AC-03
→ **Violação:** atendente recebe um dos valores como correto sem saber que há conflito —
risco de comunicar prazo ou valor errado ao cliente (UO-03)

---

**[DEVE-05]** Quando nenhum chunk com score acima do limiar configurável por domínio
for recuperado, o sistema deve usar o estado `GAP_DOCUMENTAL` e incluir o
encaminhamento da tabela RN-10, conforme o domínio classificado pelo RN-09.
→ **Origem:** requirements §5 Estado 4 · RN-04 · RN-09 · RN-10 · AC-05
→ **Violação:** atendente fica sem direção sobre a quem recorrer, ou recebe resposta
fabricada (UO-04)

---

**[DEVE-06]** Toda resposta sobre o relógio de SLA deve especificar explicitamente
qual dos dois comportamentos se aplica ao caso:
- **Chamados gerais:** relógio pausa fora do horário comercial (08h–18h, dias úteis)
- **Incidentes críticos de clientes Gold:** relógio não pausa — corre 24/7
→ **Origem:** requirements RN-06 · AC-08 · AC-09
→ **Violação:** atendente pode comunicar prazo errado ao cliente; diferença pode ser
de horas em chamados críticos

---

**[DEVE-07]** O pré-processador deve normalizar termos não-canônicos para os termos
da linguagem ubíqua **antes** de gerar o embedding, conforme a tabela de sinônimos
versionada (RN-09):

| Termo detectado na query | Normalizar para |
|---|---|
| frete diferenciado, frete pesado | Frete Especial |
| chamado P1, chamado urgente, prioridade alta, prioridade 1 | Incidente Crítico |
| carga especial, material perigoso | Carga Perigosa |
| categoria de cliente, nível de cliente, Platinum | Tier de Cliente |
| 7 dias corridos, uma semana | Prazo de Devolução |
| coleta de devolução | Coleta Reversa |

→ **Origem:** requirements RN-09 · E1 · E3
→ **Violação:** busca vetorial retorna chunks errados — resposta incorreta ou GAP_DOCUMENTAL
falso para termos com documentação disponível

---

**[DEVE-08]** O estado da sessão deve ser isolado por `user_id` (AAD object ID do
atendente extraído do token Teams). Cada atendente tem sua sessão independente.
→ **Origem:** requirements RN-08 · ADR-005
→ **Violação:** atendente A recebe contexto de consulta do atendente B — risco de
confusão de chamados e vazamento de contexto

---

**[DEVE-09]** Toda consulta sobre frete especial deve utilizar exclusivamente chunks
do PROC-042 v2, independentemente do conteúdo da query. O PROC-042 v1 é `status:
obsoleto` no índice e não pode ser recuperado.
→ **Origem:** requirements RN-01 · AC-04 · ADR-003
→ **Violação:** atendente pode calcular frete com multiplicadores da v1 — diferença de
até +12,5% no valor comunicado ao cliente (Norte: 1.6 vs 1.8)

---

**[DEVE-10]** O campo `conflicting_docs` no response body deve listar os `doc_id`
de ambos os documentos envolvidos sempre que o estado for `CONTRADIÇÃO_DETECTADA`.
→ **Origem:** requirements §5 Estado 3 · AC-03 · ADR-006
→ **Violação:** BC-05 não consegue rastrear quais pares de documentos geram contradições
recorrentes — melhoria contínua prejudicada

---

**[DEVE-11]** O campo `domain_classified` deve ser preenchido em toda resposta,
independentemente do estado, com o domínio inferido pelo classificador RN-09.
→ **Origem:** requirements RN-09 · §5 Estado 4
→ **Violação:** BC-05 não consegue categorizar gaps por domínio — impossível priorizar
quais documentos criar primeiro

---

**[DEVE-12]** Em caso de timeout (resposta não entregue em 8 segundos), o sistema
deve retornar HTTP 504 com a mensagem: *"O assistente está demorando mais que o
esperado. Tente novamente em instantes."* O estado da sessão deve ser preservado
para que o atendente possa reenviar sem perder o histórico.
→ **Origem:** requirements RNF-01 · ADR-005
→ **Violação:** atendente não recebe feedback e pode achar que a pergunta foi
processada; perda de histórico força reformulação completa do contexto

---

**[DEVE-13]** Quando uma sessão expirar (60 minutos sem nova mensagem do atendente)
ou for encerrada por reinício do serviço, o sistema deve exibir a mensagem:
*"Sua sessão anterior foi encerrada. Por favor, reformule sua pergunta com o
contexto necessário."*
→ **Origem:** requirements RN-08 · E7
→ **Violação:** atendente envia follow-up sem contexto e recebe resposta incoerente sem
saber o motivo

---

**[DEVE-14]** Perguntas sobre tier Platinum devem retornar estado `FUNDAMENTADA`
informando que o tier foi descontinuado em 2022 e listando os três tiers vigentes:
Gold, Silver e Standard.
→ **Origem:** requirements RN-05 · AC-10
→ **Violação:** retornar GAP_DOCUMENTAL para Platinum é incorreto — há documentação
formal sobre a descontinuação

---

**[DEVE-15]** A tabela de encaminhamentos (RN-10) e a tabela de sinônimos (RN-09)
devem ser lidas de arquivos de configuração versionados no repositório — nunca
hardcoded no prompt do LLM. O e-mail `sinistros@novatech.com.br` deve ser lido de
variável de ambiente.
→ **Origem:** requirements RN-09 · RN-10
→ **Violação:** mudança de contato ou área exige redeploy do modelo — risco operacional
em situação de sinistro

---

## SEÇÃO 2 — NÃO DEVE
### Comportamentos proibidos em qualquer circunstância

---

**[NAO-01]** O sistema **não deve** escolher entre documentos conflitantes e
apresentar um dos valores como correto. Quando dois documentos vigentes divergem
sobre o mesmo atributo, o estado `CONTRADIÇÃO_DETECTADA` é obrigatório.
→ **Origem:** requirements §5 Estado 3 · ADR-004 · AC-03
→ **Risco se violado:** atendente comunica valor ou prazo errado ao cliente com
aparente segurança — sem possibilidade de detectar o erro a posteriori

---

**[NAO-02]** O sistema **não deve** extrapolar ou inferir respostas a partir de
documentos adjacentes quando o tema da pergunta não tem cobertura documental.
O estado `GAP_DOCUMENTAL` é obrigatório nesse cenário.
→ **Origem:** requirements §5 Estado 4 · RN-04 · AC-05 · UO-04
→ **Risco se violado:** atendente recebe resposta fabricada apresentada como
fundamentada — máximo risco de erro sem sinal de alerta

---

**[NAO-03]** O sistema **não deve** recuperar ou utilizar chunks do PROC-042 v1
em nenhuma consulta, independentemente do contexto da query.
→ **Origem:** requirements RN-01 · AC-04
→ **Risco se violado:** multiplicadores desatualizados (Norte 1.6 em vez de 1.8,
fator >3000kg 1.5 em vez de 1.4, prazo +2 em vez de +3 dias) comunicados ao cliente

---

**[NAO-04]** O sistema **não deve** misturar conteúdo de documento normativo e
FAQ para compor uma única resposta. Quando normativo e FAQ cobrem o mesmo tema,
usa-se apenas o normativo. O FAQ só é usado quando é a única fonte para o tema —
com estado `COM_RESSALVA` obrigatório.
→ **Origem:** requirements RN-02 · E5
→ **Risco se violado:** resposta mistura autoridade de fonte normativa com imprecisão
de fonte informal sem sinalização ao atendente

---

**[NAO-05]** O sistema **não deve** responder sobre carga perigosa das classes 7,
8 ou 9 da ANTT (radioativos, corrosivos, miscelânea). A documentação cobre apenas
as classes 1 a 6. Perguntas sobre essas classes devem retornar `GAP_DOCUMENTAL`.
→ **Origem:** requirements RN-07 · AC-06
→ **Risco se violado:** orientação incorreta sobre transporte de carga regulada —
risco jurídico e de segurança

---

**[NAO-06]** O sistema **não deve** calcular o valor final do frete. Pode orientar
sobre a fórmula e os fatores (fórmula: Valor base × Multiplicador regional × Fator
de peso), mas não pode retornar um valor monetário calculado — o valor base é
externo à base documental e atualizado mensalmente.
→ **Origem:** requirements §3.2 · §11 Out of Scope
→ **Risco se violado:** atendente comunica ao cliente um valor calculado com base
desatualizada

---

**[NAO-07]** O sistema **não deve** acessar, consultar ou inferir dados de PII de
clientes finais, dados contratuais individuais (valores de contrato, histórico de
chamados) ou credenciais além do token de sessão AAD.
→ **Origem:** requirements §3.3 Fronteira de dados
→ **Risco se violado:** violação de privacidade e potencial não-conformidade com LGPD

---

**[NAO-08]** O sistema **não deve** verificar ou medir SLA em tempo real de chamados
em aberto. Pode responder sobre as regras de SLA (prazos, tiers, penalidades), mas
não sobre o status atual de um chamado específico.
→ **Origem:** requirements §3.2 · §11 Out of Scope
→ **Risco se violado:** atendente pode tomar decisão baseada em dado de SLA
desatualizado fornecido pelo assistente

---

**[NAO-09]** O sistema **não deve** usar o limiar de R$ 50.000 do FAQ Item 27 como
critério de incidente crítico. O limiar oficial é R$ 100.000, conforme SLA-2024 §3.
→ **Origem:** requirements RN-03
→ **Risco se violado:** chamados são escalados com prioridade crítica desnecessariamente,
sobrecarregando o time de plantão

---

**[NAO-10]** O sistema **não deve** abrir, atualizar ou encerrar chamados no sistema
de chamados (Portal do Cliente / Azure DevOps). Pode orientar o atendente sobre como
fazê-lo, mas não executa essa ação.
→ **Origem:** requirements §3.2 · §11 Out of Scope
→ **Risco se violado:** ações irreversíveis em chamados de clientes sem supervisão humana

---

**[NAO-11]** O sistema **não deve** responder sobre seguro de carga com base no
FAQ Item 22 (percentuais 0,3% e 0,8%) sem sinalizar explicitamente que a fonte é
informal e que contratos antigos podem ter percentuais diferentes.
*(Pendente resolução da OQ-02 — até lá, este guardrail é conservador.)*
→ **Origem:** requirements OQ-02 · E6 (analogia)
→ **Risco se violado:** atendente comunica percentual de seguro incorreto — impacto
financeiro direto ao cliente

---

**[NAO-12]** O LLM **não deve** tomar a decisão de sinalizar ou não uma contradição.
A decisão de acionar `CONTRADIÇÃO_DETECTADA` é feita pelo mecanismo determinístico
de comparação de metadados (`topic_tag` + `attribute_key`) antes da geração.
O LLM recebe a contradição já detectada e gera apenas o texto de alerta.
→ **Origem:** requirements ADR-006
→ **Risco se violado:** modelo pode "resolver" silenciosamente conflitos em vez de
sinalizá-los — violação do ADR-004 sem rastro detectável

---

## SEÇÃO 3 — QUANDO EM DÚVIDA
### Comportamentos de fallback para cenários ambíguos

---

**[DUVIDA-01]** Quando a query contiver palavras-chave de dois domínios simultaneamente
(ex.: "qual o SLA para devolução de carga perigosa Gold?"), o classificador deve
atribuir o domínio com maior contagem de palavras-chave. Em caso de empate, usar
`nao_classificado` com limiar 0,70.
Tratar cada domínio identificado separadamente na resposta — nunca mesclar regras
de domínios diferentes em um único parágrafo.
→ **Origem:** requirements RN-09 · E2

---

**[DUVIDA-02]** Quando o campo `tier_hint` não estiver presente no request (campo
opcional) e a pergunta envolver comportamento dependente de tier (SLA, incidente
crítico, relógio de SLA), o sistema deve:
1. Responder com as regras para **todos os tiers** disponíveis (Gold, Silver, Standard)
2. Orientar o atendente a identificar o tier do cliente antes de tomar uma decisão

Não inferir o tier pelo conteúdo da pergunta.
→ **Origem:** requirements RN-06 · OQ-05 · OQ-06

---

**[DUVIDA-03]** Quando o classificador de domínio RN-09 não conseguir classificar a
query em nenhum dos três domínios definidos, usar `nao_classificado` com:
- Limiar de relevância: 0,70
- Encaminhamento padrão: *"Consulte seu supervisor para orientação sobre este tema."*

Não bloquear a consulta nem retornar erro — processar com os parâmetros padrão e
registrar em BC-05 para análise.
→ **Origem:** requirements RN-09 · RN-10

---

**[DUVIDA-04]** Quando o FAQ for a única fonte para um tema e não houver contradição
com nenhum documento normativo (ex.: FAQ Item 32 — frete expresso para carga
perigosa), o sistema deve responder com estado `COM_RESSALVA` e incluir,
adicionalmente à ressalva padrão, a instrução específica:
*"Verifique com a área de Compliance antes de confirmar esta informação ao cliente."*

Não retornar `GAP_DOCUMENTAL` quando há conteúdo no FAQ — a ressalva adequada é
preferível ao silêncio.
→ **Origem:** requirements RN-02 · E6 · AC-02

---

**[DUVIDA-05]** Quando a pergunta mencionar prazo de devolução com data fornecida
pelo cliente (e não pelo sistema de tracking), o sistema deve responder com a regra
do prazo **e** incluir o aviso: *"O prazo de 7 dias úteis é contado a partir da data
de recebimento confirmada no sistema de tracking — não necessariamente da data
informada pelo cliente. Verifique a data no sistema antes de comunicar o prazo."*
→ **Origem:** requirements E4 · RN linguagem ubíqua (Prazo de Devolução)

---

**[DUVIDA-06]** Quando o score do chunk mais relevante estiver abaixo do limiar do
domínio mas acima de 0,50 (zona cinza — há alguma relevância, mas abaixo da
confiança mínima), o sistema deve retornar `GAP_DOCUMENTAL` **e** registrar no
campo de rastreabilidade o score máximo obtido para que BC-05 possa analisar se o
limiar precisa ser ajustado.

Não usar chunks de zona cinza para gerar resposta — o risco de resposta incorreta
supera o benefício.
→ **Origem:** requirements §5 Estado 4 · RNF implícito de qualidade

---

**[DUVIDA-07]** Quando o serviço for reiniciado durante uma sessão ativa e o Redis
perder o histórico, o sistema deve exibir a mensagem de encerramento de sessão
(DEVE-13) **antes** de processar qualquer nova mensagem. Não tentar reconstruir
o histórico perdido — não há fonte confiável para fazê-lo.
→ **Origem:** requirements RN-08

---

**[DUVIDA-08]** Quando a pergunta envolver desconto de volume e o atendente citar
o limiar de 10 fretes/mês (FAQ Item 45), o sistema deve identificar a divergência
com o PROC-042 v2 (8 fretes/mês) e retornar `CONTRADIÇÃO_DETECTADA` — mesmo que
a fonte do FAQ seja informal.

A contradição entre documento normativo vigente e FAQ com valores explicitamente
divergentes ativa o estado de contradição, não o de ressalva.
→ **Origem:** requirements E8 · RN-02

---

**[DUVIDA-09]** Quando o atendente perguntar sobre frete padrão (abaixo de 500kg)
de qualquer forma — diretamente ou como parte de uma pergunta maior — o sistema
deve retornar `GAP_DOCUMENTAL` para essa parte específica da pergunta e continuar
respondendo normalmente sobre os demais aspectos cobertos pela documentação.

Não bloquear toda a resposta por causa de um sub-tema sem cobertura.
→ **Origem:** requirements RN-04 · AC-05

---

**[DUVIDA-10]** Quando não for possível determinar se uma pergunta se refere a
"Coleta Reversa" (o serviço) ou "Frete Reverso" (o custo), o sistema deve responder
sobre **ambos** explicitando a diferença:
- **Coleta Reversa:** serviço de retirada, agendado em até 2 dias úteis após aprovação
- **Frete Reverso:** custo do transporte de retorno, responsabilidade variável
  conforme o motivo da devolução

Não escolher um dos dois sem base na query.
→ **Origem:** linguagem ubíqua — termos com risco de confusão · POL-001 §3.3 e §3.5

---

## Tabela de Rastreabilidade

| Guardrail | Categoria | Origem principal | UO relacionado | Risco se violado |
|---|---|---|---|---|
| DEVE-01 | Estado obrigatório | §5 Estados | UO-01, UO-02 | Atendente sem base para avaliar confiança |
| DEVE-02 | Citação de fonte | ADR-003, AC-01 | UO-01, UO-02 | Informação não verificável |
| DEVE-03 | Ressalva FAQ | §5 Estado 2, AC-02 | UO-02 | Informação não validada repassada como definitiva |
| DEVE-04 | Sinalização de contradição | ADR-004, ADR-006, AC-03 | UO-03 | Valor errado comunicado com aparente segurança |
| DEVE-05 | GAP_DOCUMENTAL + encaminhamento | §5 Estado 4, RN-10 | UO-04 | Resposta fabricada ou atendente sem direção |
| DEVE-06 | Distinção relógio SLA | RN-06, AC-08, AC-09 | UO-02 | Prazo errado de SLA comunicado ao cliente |
| DEVE-07 | Normalização de termos | RN-09, E1, E3 | UO-01 | Miss na busca vetorial — resposta incorreta |
| DEVE-08 | Isolamento de sessão | RN-08 | UO-05 | Contexto vazado entre atendentes |
| DEVE-09 | PROC-042 v2 exclusivo | RN-01, AC-04 | UO-03 | Multiplicador desatualizado — erro de até 12,5% |
| DEVE-10 | conflicting_docs no response | §5 Estado 3, ADR-006 | — | Rastreabilidade de contradições inoperante |
| DEVE-11 | domain_classified no response | RN-09, §5 Estado 4 | — | BC-05 não categoriza gaps |
| DEVE-12 | Comportamento em timeout | RNF-01, ADR-005 | UO-01 | Atendente sem feedback; histórico perdido |
| DEVE-13 | Mensagem de sessão expirada | RN-08, E7 | UO-05 | Resposta incoerente sem aviso |
| DEVE-14 | Platinum = FUNDAMENTADA | RN-05, AC-10 | UO-04 | GAP_DOCUMENTAL falso — doc existe |
| DEVE-15 | Config externalizada | RN-09, RN-10 | — | Mudança de contato exige redeploy |
| NAO-01 | Proibido escolher entre conflitantes | ADR-004, AC-03 | UO-03 | Valor errado apresentado como correto |
| NAO-02 | Proibido extrapolar | §5 Estado 4, UO-04 | UO-04 | Resposta fabricada sem sinal de alerta |
| NAO-03 | Proibido usar PROC-042 v1 | RN-01, AC-04 | UO-03 | Multiplicadores desatualizados |
| NAO-04 | Proibido mesclar normativo + FAQ | RN-02, E5 | UO-02 | Autoridade de fonte misturada sem sinalização |
| NAO-05 | Proibido responder classes 7-9 ANTT | RN-07, AC-06 | UO-04 | Orientação incorreta sobre carga regulada |
| NAO-06 | Proibido calcular valor final do frete | §3.2, §11 | — | Valor calculado com base desatualizada |
| NAO-07 | Proibido acessar PII | §3.3 | — | Violação de privacidade / LGPD |
| NAO-08 | Proibido medir SLA em tempo real | §3.2, §11 | — | Dado desatualizado como se fosse atual |
| NAO-09 | Proibido usar limiar R$50K do FAQ | RN-03 | — | Escaladas desnecessárias |
| NAO-10 | Proibido executar ações em chamados | §3.2, §11 | — | Ações irreversíveis sem supervisão |
| NAO-11 | Seguro de carga: ressalva obrigatória | OQ-02, E6 | UO-02 | Percentual incorreto comunicado ao cliente |
| NAO-12 | LLM não decide contradição | ADR-006 | UO-03 | Contradição resolvida silenciosamente |
| DUVIDA-01 | Query com múltiplos domínios | RN-09, E2 | UO-01 | Regras de domínios diferentes mescladas |
| DUVIDA-02 | tier_hint ausente | RN-06, OQ-05 | UO-02 | SLA calculado para tier errado |
| DUVIDA-03 | Domínio não classificado | RN-09, RN-10 | UO-04 | Consulta bloqueada desnecessariamente |
| DUVIDA-04 | FAQ como única fonte | RN-02, E6 | UO-02 | Silêncio quando havia conteúdo disponível |
| DUVIDA-05 | Prazo de devolução com data do cliente | E4 | UO-02 | Prazo contado a partir da data errada |
| DUVIDA-06 | Score em zona cinza (0,50–limiar) | §5 Estado 4 | UO-04 | Chunk de baixa relevância usado na geração |
| DUVIDA-07 | Reinício de serviço mid-session | RN-08 | UO-05 | Contexto reconstruído incorretamente |
| DUVIDA-08 | Contradição FAQ × normativo em desconto | E8, RN-02 | UO-03 | Contradição tratada como ressalva |
| DUVIDA-09 | Frete padrão como sub-tema | RN-04, AC-05 | UO-04 | Toda resposta bloqueada por sub-tema sem cobertura |
| DUVIDA-10 | Ambiguidade Coleta Reversa × Frete Reverso | Linguagem ubíqua | UO-02 | Atendente orientado sobre o conceito errado |

---

## Open Questions que afetam guardrails ativos

Os guardrails abaixo têm comportamento provisório até a resolução das OQs indicadas.
Após resolução, este documento deve ser atualizado.

| Guardrail provisório | OQ pendente | Comportamento atual | Comportamento possível após decisão |
|---|---|---|---|
| NAO-11 (seguro de carga) | OQ-02 | Ressalva obrigatória + verificação com supervisor | Se NovaTech criar doc normativo: FUNDAMENTADA sem ressalva |
| DUVIDA-04 (FAQ como única fonte) | OQ-01 | COM_RESSALVA quando FAQ ingerido | Se FAQ não for ingerido: GAP_DOCUMENTAL + encaminhamento |
| DUVIDA-02 (tier_hint ausente) | OQ-05, OQ-06 | Responder para todos os tiers | Se bot preencher automaticamente: responder para o tier específico |
