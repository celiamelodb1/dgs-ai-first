# guardrails.md — Query Endpoint
**Sistema:** Assistente de IA NovaTech
**Componente:** Query Endpoint (BC-02 — Atendimento e Consulta)
**Versão:** 1.3.0
**Derivado de:** requirements.md v1.1.0
**Data:** Junho 2026
**Responsável:** DB1 — Equipe de Produto

### Changelog
| Versão | Alterações |
|---|---|
| 1.0.0 | Versão inicial — 37 guardrails em DEVE / NÃO DEVE / QUANDO EM DÚVIDA |
| 1.1.0 | Seção 4 adicionada: classificação de enforcement `[DET]` / `[HBR]` / `[PRB]` para todos os 37 guardrails, análise de risco residual e coluna de enforcement na tabela de rastreabilidade |
| 1.2.0 | Seção 4.4 adicionada: justificativas individuais de classificação para todos os 37 guardrails |
| 1.3.0 | Seção 5 adicionada: conexão de cada guardrail com 3 incidentes simulados, análise de causa-raiz, cobertura por incidente e 3 lacunas identificadas (L-01, L-02, L-03) |

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

---

## SEÇÃO 4 — Classificação de Enforcement

Cada guardrail é classificado em uma das três categorias:

| Categoria | Símbolo | Definição |
|---|---|---|
| **Determinístico** | `[DET]` | Garantido por lógica de código — filtro, validação de schema, lookup em tabela, TTL, comparação de metadados. O LLM não participa da decisão. |
| **Probabilístico** | `[PRB]` | Dependente do LLM gerar o comportamento correto a partir de instrução no system prompt. Pode falhar silenciosamente. |
| **Híbrido** | `[HBR]` | A decisão estrutural é feita em código (ex: detecção de estado, seleção de fonte), mas o conteúdo gerado pelo LLM ainda carrega risco residual de desvio. |

---

### 4.1 — Classificação individual

| ID | Enunciado resumido | Enforcement | Mecanismo |
|---|---|---|---|
| DEVE-01 | Toda resposta deve indicar o estado | `[DET]` | O estado é determinado pelo pipeline antes da chamada ao LLM (resultado da etapa de retrieval + detecção de contradição + limiar). O campo `state` é populado em código e não pode ser omitido — validação de schema na saída. |
| DEVE-02 | Resposta FUNDAMENTADA/COM_RESSALVA cita fonte, versão e seção | `[HBR]` | A presença dos campos `sources[].doc_id`, `sources[].version` e `sources[].section` no response é validada em código (schema obrigatório). O **conteúdo** da citação no texto gerado pelo LLM ainda depende de instrução no prompt. |
| DEVE-03 | Resposta baseada em FAQ usa COM_RESSALVA + disclaimer | `[HBR]` | O estado `COM_RESSALVA` é atribuído em código ao detectar `informal: true` no chunk. O texto do disclaimer é injetado programaticamente no response — não gerado pelo LLM. O **conteúdo da resposta** em si ainda é gerado pelo LLM. |
| DEVE-04 | Contradição → CONTRADIÇÃO_DETECTADA + escalada | `[HBR]` | A detecção de contradição é determinística (ADR-006: `topic_tag` + `attribute_key`). O estado `CONTRADIÇÃO_DETECTADA` é atribuído em código. O texto de alerta e escalada pode ser template fixo injetado programaticamente — mas se for gerado pelo LLM, é probabilístico. |
| DEVE-05 | Score abaixo do limiar → GAP_DOCUMENTAL + encaminhamento | `[DET]` | Limiar por domínio é verificado em código após o retrieval. O estado `GAP_DOCUMENTAL` e o encaminhamento da RN-10 são atribuídos por lookup em tabela de configuração — sem participação do LLM. |
| DEVE-06 | Resposta sobre SLA distingue chamado geral vs incidente crítico Gold | `[PRB]` | Depende do LLM identificar corretamente o tipo de chamado na query e aplicar a distinção. Instrução no system prompt. Sem mecanismo de verificação estrutural do conteúdo gerado. |
| DEVE-07 | Normalizar termos não-canônicos antes do embedding | `[DET]` | Substituição por correspondência exata (case-insensitive) em tabela de sinônimos antes da geração do embedding — operação de string em código, sem participação do LLM. |
| DEVE-08 | Sessão isolada por user_id (AAD) | `[DET]` | A chave de sessão no Redis é composta por `user_id` extraído do token AAD — operação de código no middleware de autenticação. LLM nunca vê o `user_id`. |
| DEVE-09 | Usar exclusivamente PROC-042 v2 | `[DET]` | Filtro no índice vetorial por `status != obsoleto` antes do retrieval. Chunks com `doc_id: PROC-042-v1` têm `status: obsoleto` — nunca retornados independentemente da query. |
| DEVE-10 | Preencher conflicting_docs no response | `[DET]` | O campo `conflicting_docs` é populado pelo mecanismo de detecção de contradição (ADR-006) em código, antes da chamada ao LLM. Validação de schema na saída garante presença quando estado é `CONTRADIÇÃO_DETECTADA`. |
| DEVE-11 | Preencher domain_classified no response | `[DET]` | O domínio é classificado pelo pré-processador RN-09 em código antes do retrieval. O campo é populado no response pela camada de orquestração — sem participação do LLM. |
| DEVE-12 | Timeout 8s → HTTP 504 + sessão preservada | `[DET]` | Timer implementado na camada de orquestração. Se LLM não responder em 8s, o middleware retorna 504 e preserva o estado da sessão no Redis — independente do LLM. |
| DEVE-13 | Sessão expirada → mensagem de encerramento | `[DET]` | TTL do Redis (60 min sem mensagem do atendente). Ao detectar chave expirada, a camada de orquestração injeta a mensagem de encerramento — sem participação do LLM. |
| DEVE-14 | Platinum → FUNDAMENTADA informando descontinuação | `[PRB]` | Depende do LLM recuperar o chunk correto (SLA-2024 §1) e gerar a resposta adequada. A normalização de "Platinum" → "Tier de Cliente" pelo RN-09 (DEVE-07) ajuda no retrieval, mas o conteúdo final é probabilístico. |
| DEVE-15 | Tabela de sinônimos e encaminhamentos externalizadas (não hardcoded) | `[DET]` | Leitura de arquivo de configuração versionado e variável de ambiente na inicialização do serviço — convenção de código, auditável em PR. |
| NAO-01 | Nunca escolher entre documentos conflitantes | `[HBR]` | A detecção de contradição é determinística (ADR-006). O estado `CONTRADIÇÃO_DETECTADA` é atribuído em código — o LLM nunca recebe os dois chunks conflitantes para sintetizar. Risco residual: se o mecanismo de detecção falhar (metadados ausentes), o LLM recebe ambos os chunks e pode sintetizar silenciosamente. |
| NAO-02 | Nunca extrapolar sem cobertura documental | `[HBR]` | O estado `GAP_DOCUMENTAL` é atribuído em código por limiar de score (DEVE-05). Risco residual: o LLM ainda é chamado em outros estados e pode extrapolar além dos chunks recuperados — instrução no prompt é necessária como segunda camada. |
| NAO-03 | Nunca usar PROC-042 v1 | `[DET]` | Filtro no índice vetorial por `status: obsoleto` (DEVE-09). Garantia estrutural completa — o LLM nunca vê chunks da v1. |
| NAO-04 | Nunca mesclar normativo + FAQ | `[HBR]` | A seleção de fonte é feita em código por RN-02 (normativo tem precedência). Apenas um tipo de chunk é passado ao LLM. Risco residual: se o LLM receber chunks mistos por falha na seleção, pode mesclar conteúdo. |
| NAO-05 | Nunca responder sobre classes 7-9 ANTT | `[HBR]` | Depende do classificador RN-09 identificar a classe mencionada e do limiar de score não retornar chunks de carga perigosa para classes sem cobertura. Risco residual: sem filtro explícito por classe ANTT no índice — o LLM pode receber chunks de classes 1-6 e extrapolar para 7-9 via instrução no prompt. |
| NAO-06 | Nunca calcular valor final do frete | `[PRB]` | Não há dado de valor base no índice vetorial (fonte externa), o que torna o cálculo impossível estruturalmente. Porém, o LLM pode estimar ou aluciná-lo a partir de contexto parcial — instrução explícita no prompt é necessária. |
| NAO-07 | Nunca acessar PII de clientes | `[DET]` | O endpoint não tem integração com CRM/ERP (§3.2). Fronteira de dados garantida pela ausência de conexão — não é uma instrução ao LLM, é uma restrição de infraestrutura. |
| NAO-08 | Nunca medir SLA em tempo real | `[DET]` | O endpoint não tem integração com Azure DevOps (§3.2). Mesma lógica do NAO-07 — restrição de infraestrutura, não instrução ao LLM. |
| NAO-09 | Nunca usar limiar R$50K do FAQ | `[PRB]` | O chunk do FAQ Item 27 pode ser recuperado se a query for sobre valor de carga. O LLM precisa ser instruído a ignorar o limiar do FAQ e usar o SLA-2024 §3 (R$100K). Sem filtro estrutural que impeça o chunk de ser recuperado. |
| NAO-10 | Nunca executar ações em chamados | `[DET]` | O endpoint não tem integração com Portal do Cliente ou Azure DevOps (§3.2). Restrição de infraestrutura. |
| NAO-11 | Seguro de carga: ressalva obrigatória | `[PRB]` | FAQ Item 22 pode ser recuperado. O estado `COM_RESSALVA` é atribuído em código quando a fonte é informal (DEVE-03). A instrução adicional "verifique com Compliance" depende do prompt. |
| NAO-12 | LLM não decide sobre contradição | `[DET]` | A decisão de acionar `CONTRADIÇÃO_DETECTADA` é feita antes da chamada ao LLM (ADR-006). O LLM só é acionado para gerar o texto de alerta — não vê os dois chunks conflitantes simultaneamente. |
| DUVIDA-01 | Query com múltiplos domínios → tratar separadamente | `[HBR]` | O classificador RN-09 atribui um único domínio por contagem de palavras-chave (código). Mas a instrução de "tratar cada domínio separadamente na resposta" depende do LLM — probabilístico. |
| DUVIDA-02 | tier_hint ausente → responder para todos os tiers | `[HBR]` | A ausência do campo `tier_hint` é detectável em código (campo opcional no schema). A instrução de "responder para todos os tiers" depende do LLM — probabilístico. |
| DUVIDA-03 | Domínio não classificado → nao_classificado + parâmetros padrão | `[DET]` | Fallback para `nao_classificado` com limiar 0,70 é implementado em código no classificador RN-09. O encaminhamento padrão é lido da tabela RN-10 — sem participação do LLM. |
| DUVIDA-04 | FAQ como única fonte → COM_RESSALVA + instrução de Compliance | `[HBR]` | O estado `COM_RESSALVA` é atribuído em código (DEVE-03). A instrução adicional "verifique com Compliance" pode ser injetada como template para categorias de risco jurídico — mas identificar que é "risco jurídico" depende de configuração ou do LLM. |
| DUVIDA-05 | Prazo de devolução com data do cliente → aviso sobre tracking | `[PRB]` | Depende do LLM identificar que a data mencionada é do cliente (não do tracking) e incluir o aviso. Sem mecanismo estrutural para detectar esse padrão na query. |
| DUVIDA-06 | Score em zona cinza (0,50–limiar) → GAP_DOCUMENTAL + log do score | `[DET]` | A comparação de score com o limiar é feita em código. O registro do score máximo no log de rastreabilidade é operação de código. O estado `GAP_DOCUMENTAL` é atribuído sem participação do LLM. |
| DUVIDA-07 | Reinício do serviço → mensagem de encerramento antes de processar | `[DET]` | A ausência da chave no Redis (por reinício) é detectada em código. A mensagem de encerramento é injetada pela camada de orquestração antes de qualquer chamada ao LLM. |
| DUVIDA-08 | Contradição FAQ × normativo em desconto → CONTRADIÇÃO_DETECTADA | `[HBR]` | Requer que FAQ e PROC-042 v2 tenham `attribute_key: desconto_volume` idêntico e valores divergentes nos metadados. Se os metadados estiverem corretos, a detecção é determinística (ADR-006). Risco residual: FAQ pode não ter `attribute_key` atribuído na ingestão — sem esse metadado, a detecção falha e o guardrail vira probabilístico. |
| DUVIDA-09 | Frete padrão como sub-tema → GAP_DOCUMENTAL parcial | `[PRB]` | Requer que o LLM identifique o sub-tema de frete padrão dentro de uma pergunta maior e responda parcialmente. Sem mecanismo estrutural para decomposição de query em sub-temas. |
| DUVIDA-10 | Ambiguidade Coleta Reversa × Frete Reverso → responder ambos | `[PRB]` | Depende do LLM identificar a ambiguidade e incluir ambas as definições. Sem detecção estrutural de ambiguidade de termos na query. |

---

### 4.2 — Distribuição por categoria

| Categoria | Quantidade | % | Guardrails |
|---|---|---|---|
| `[DET]` Determinístico | 18 | 49% | DEVE-01, DEVE-05, DEVE-07, DEVE-08, DEVE-09, DEVE-10, DEVE-11, DEVE-12, DEVE-13, DEVE-15, NAO-03, NAO-07, NAO-08, NAO-10, NAO-12, DUVIDA-03, DUVIDA-06, DUVIDA-07 |
| `[HBR]` Híbrido | 12 | 32% | DEVE-02, DEVE-03, DEVE-04, NAO-01, NAO-02, NAO-04, NAO-05, DUVIDA-01, DUVIDA-02, DUVIDA-04, DUVIDA-08 |
| `[PRB]` Probabilístico | 7 | 19% | DEVE-06, DEVE-14, NAO-06, NAO-09, NAO-11, DUVIDA-05, DUVIDA-09, DUVIDA-10 |

---

### 4.3 — Análise de risco residual

Os guardrails `[PRB]` e `[HBR]` têm risco residual que não é eliminável por código.
A tabela abaixo prioriza os que merecem atenção especial em testes e monitoramento.

| ID | Categoria | Risco residual | Mitigação recomendada |
|---|---|---|---|
| NAO-01 | `[HBR]` | Se `topic_tag` ou `attribute_key` não forem atribuídos na ingestão (BC-01), a detecção de contradição falha e o LLM recebe chunks conflitantes | Validação obrigatória de metadados na ingestão em BC-01. Alerta em BC-05 para chunks sem `attribute_key`. |
| NAO-02 | `[HBR]` | LLM pode extrapolar além dos chunks recuperados mesmo com instrução no prompt | Guard de output: validar se `sources` na resposta contém apenas chunks efetivamente recuperados no retrieval. |
| NAO-05 | `[HBR]` | LLM pode extrapolar de chunks de classes 1-6 para 7-9 | Adicionar filtro explícito por classe ANTT nos metadados do chunk e rejeitar chunks de classes não cobertas. |
| NAO-06 | `[PRB]` | LLM pode aluciná valor base do frete a partir de contexto parcial | Instrução explícita no system prompt: "nunca fornecer valor monetário calculado de frete". Monitorar em BC-05 respostas com padrão de valor monetário. |
| NAO-09 | `[PRB]` | Chunk do FAQ Item 27 (R$50K) pode ser recuperado e o LLM pode usá-lo | Marcar FAQ Item 27 com metadado `superseded_by: SLA-2024-§3` na ingestão. Instrução no prompt para ignorar limiares de FAQ quando SLA-2024 existir. |
| DUVIDA-08 | `[HBR]` | FAQ pode não ter `attribute_key: desconto_volume` — detecção não ocorre | Garantir na ingestão do FAQ que itens com valores numéricos explícitos recebam `attribute_key`. Cobrir em testes de ingestão de BC-01. |
| DUVIDA-09 | `[PRB]` | LLM pode bloquear toda a resposta em vez de tratar frete padrão como sub-tema parcial | Instrução no prompt com exemplo explícito de resposta parcial. Cobrir em testes exploratórios. |

---

### 4.4 — Justificativas individuais de classificação

A justificativa responde a uma única pergunta para cada guardrail:
**por que o comportamento pode (ou não pode) ser garantido sem depender do LLM?**

---

#### DEVE-01 `[DET]`
O estado da resposta é uma **propriedade do pipeline de retrieval**, não do texto gerado. A sequência é: recuperar chunks → comparar metadados → calcular scores → atribuir estado. Tudo isso acontece antes da chamada ao LLM. O campo `state` no response schema é obrigatório e validado por schema — se ausente, a resposta é rejeitada. O LLM recebe o estado já decidido como contexto; não o produz.

---

#### DEVE-02 `[HBR]`
A **presença** dos campos `sources[].doc_id`, `sources[].version` e `sources[].section` no response é garantida em código via schema validation — o pipeline popula esses campos a partir dos metadados dos chunks recuperados. Porém, o **texto da resposta gerado pelo LLM** deve referenciar a fonte de forma coerente com o que está no campo `sources`. Essa coerência entre o campo estruturado e o texto narrativo depende de instrução no prompt — é probabilística. Um LLM pode citar `POL-001 §3.3` no campo `sources` e não mencionar nenhuma fonte no texto, ou mencionar uma seção diferente.

---

#### DEVE-03 `[HBR]`
Dois comportamentos distintos, com garantias distintas. O estado `COM_RESSALVA` é atribuído em código ao detectar `informal: true` no metadado do chunk — determinístico. O texto do disclaimer padrão pode ser injetado como template fixo pela camada de orquestração, sem passar pelo LLM — determinístico. Mas o **conteúdo da resposta** sobre o tema perguntado ainda é gerado pelo LLM a partir do chunk de FAQ. Se o LLM distorcer o conteúdo do FAQ, o disclaimer estará presente mas a informação será incorreta. A parte determinística protege a sinalização; a parte probabilística não protege o conteúdo.

---

#### DEVE-04 `[HBR]`
A **decisão de acionar** `CONTRADIÇÃO_DETECTADA` é determinística (ADR-006): comparação de `topic_tag` + `attribute_key` entre chunks recuperados. O estado é atribuído em código. O texto de escalada pode ser um template fixo injetado programaticamente — se for, torna-se determinístico. O risco híbrido vem de duas fontes: (1) se o texto de escalada for gerado pelo LLM em vez de injetado como template, é probabilístico; (2) se BC-01 não atribuir `attribute_key` corretamente na ingestão, a detecção falha silenciosamente e o LLM recebe os dois chunks conflitantes — sem nenhuma garantia.

---

#### DEVE-05 `[DET]`
Completamente fora do caminho do LLM. O score do chunk mais relevante é calculado pelo motor de busca vetorial. A comparação com o limiar por domínio é feita em código pela camada de orquestração. O encaminhamento é uma lookup na tabela RN-10 por `domain_classified`. Nenhuma dessas operações envolve geração de linguagem natural. Quando o estado é `GAP_DOCUMENTAL`, o LLM não é chamado para geração — o response é montado inteiramente pela camada de orquestração.

---

#### DEVE-06 `[PRB]`
Este guardrail depende de duas capacidades do LLM que não têm garantia estrutural: (1) identificar corretamente o tipo de chamado e o tier do cliente na query — extração de entidades, sujeita a erro; (2) aplicar a distinção correta entre os dois comportamentos do relógio de SLA no texto gerado. O campo `tier_hint` no request (quando presente) ajuda na primeira capacidade, mas não elimina o risco. Não há validação possível do conteúdo da resposta que seja mais barata do que re-executar a lógica de negócio em código — o que tornaria o LLM desnecessário para essa parte.

---

#### DEVE-07 `[DET]`
Substituição de string por correspondência exata antes da geração do embedding. É uma operação de pré-processamento de texto — `str.replace()` ou equivalente aplicado sobre a query antes de qualquer chamada ao modelo. Não há inferência, não há probabilidade. A única dependência é que a tabela de sinônimos esteja completa e atualizada. O processo de adicionar novos sinônimos (via PR, sem redeploy) é operacional, não técnico.

---

#### DEVE-08 `[DET]`
A chave de sessão no Redis é construída pela camada de middleware de autenticação como `session:{user_id}:{session_uuid}`, onde `user_id` é extraído do token AAD antes de qualquer lógica de negócio. O LLM nunca vê o `user_id` e não participa da construção da chave. O isolamento é uma propriedade do sistema de armazenamento, não uma instrução ao modelo.

---

#### DEVE-09 `[DET]`
O filtro `WHERE status != 'obsoleto'` é aplicado na camada de retrieval antes da busca vetorial. Chunks com `doc_id: PROC-042-v1` têm `status: obsoleto` atribuído na ingestão por BC-01. O motor de busca vetorial nunca retorna esses chunks, independentemente do conteúdo da query ou de qualquer instrução no prompt. É uma restrição de índice — equivalente a uma cláusula WHERE em SQL.

---

#### DEVE-10 `[DET]`
O campo `conflicting_docs` é populado pelo mesmo mecanismo que detecta a contradição (ADR-006): ao identificar dois chunks com `attribute_key` idêntico e valores divergentes, a camada de orquestração registra ambos os `doc_id` no array. Essa operação acontece antes da chamada ao LLM. A validação de schema garante que o campo está presente quando `state = CONTRADIÇÃO_DETECTADA`. O LLM não decide quais documentos entram no array.

---

#### DEVE-11 `[DET]`
O classificador RN-09 atribui o domínio antes do retrieval, por contagem de palavras-chave — operação determinística de código. A camada de orquestração popula o campo `domain_classified` no response a partir desse resultado. O LLM recebe o domínio classificado como contexto, mas não o produz. A validação de schema garante que o campo está presente em toda resposta.

---

#### DEVE-12 `[DET]`
O timer de 8 segundos é implementado na camada de orquestração com `Promise.race()` ou equivalente: a chamada ao LLM concorre com um timeout. Se o LLM não responder em 8s, o middleware cancela a chamada, retorna HTTP 504 com o texto fixo e preserva a chave de sessão no Redis sem modificação (operação de não-escrita). Nenhuma dessas ações depende do LLM.

---

#### DEVE-13 `[DET]`
O TTL da chave no Redis é configurado para 3.600 segundos (60 minutos) e reiniciado a cada mensagem do atendente — operação de `EXPIRE` no Redis. Ao tentar recuperar uma chave expirada (ou inexistente após reinício do serviço), a camada de orquestração recebe `null` e injeta a mensagem de encerramento como texto fixo antes de criar uma nova sessão. O LLM não participa dessa detecção nem da geração da mensagem.

---

#### DEVE-14 `[PRB]`
Ao contrário do NAO-03 (v1 nunca recuperada por filtro de índice), não há filtro que force o retrieval de um documento específico para a query "Platinum". O comportamento correto depende de: (1) o pré-processador RN-09 normalizar "Platinum" → "Tier de Cliente" (determinístico — DEVE-07); (2) o chunk correto do SLA-2024 §1 ter score alto o suficiente para ser recuperado (dependente do índice); (3) o LLM gerar a resposta correta a partir do chunk recuperado (probabilístico). A normalização do passo 1 ajuda, mas não garante os passos 2 e 3.

---

#### DEVE-15 `[DET]`
Leitura de arquivo de configuração e variável de ambiente são operações de inicialização do serviço — acontecem no boot, antes de qualquer request. É uma convenção de código auditável em PR e em pipeline de CI (testes que verificam se as variáveis de ambiente obrigatórias estão definidas). O LLM nunca tem acesso às tabelas de configuração diretamente — recebe apenas o resultado da lookup já processado pela camada de orquestração.

---

#### NAO-01 `[HBR]`
A proibição de escolher entre conflitantes é garantida pela arquitetura: quando `CONTRADIÇÃO_DETECTADA` é acionado, o LLM **não recebe os dois chunks conflitantes para sintetizar** — recebe apenas os metadados dos documentos e um template de alerta. Isso é determinístico. O risco híbrido é a **falha de pré-condição**: se BC-01 não atribuir `attribute_key` ao chunk, a comparação não ocorre, o conflito não é detectado, e ambos os chunks chegam ao LLM sem sinalização. Nesse cenário, o guardrail se torna completamente probabilístico — dependente de instrução no prompt que o LLM pode ou não seguir.

---

#### NAO-02 `[HBR]`
O estado `GAP_DOCUMENTAL` é determinístico por limiar de score — quando nenhum chunk passa do threshold, o LLM não é chamado para geração (DEVE-05). Isso elimina a extrapolação no caso mais comum. O risco híbrido está nos outros três estados: nos estados FUNDAMENTADA, COM_RESSALVA e CONTRADIÇÃO_DETECTADA, o LLM recebe chunks e **pode extrapolar além deles**. Por exemplo, ao responder sobre frete especial com chunks do PROC-042 v2, o LLM pode inferir regras de frete padrão (não cobertas por nenhum documento) a partir do contexto geral. Essa extrapolação não é detectável estruturalmente — requer instrução explícita no prompt ("responda apenas com base nos trechos fornecidos") e monitoramento em BC-05.

---

#### NAO-03 `[DET]`
Filtro de índice aplicado na camada de retrieval (`status: obsoleto`). Idêntico em mecanismo ao DEVE-09 — são o mesmo guardrail visto por perspectivas diferentes (DEVE-09 é o comportamento positivo obrigatório; NAO-03 é a proibição correspondente). A garantia é estrutural e completa: independentemente do que o prompt instrua, o motor de busca vetorial nunca retorna chunks com `status: obsoleto`.

---

#### NAO-04 `[HBR]`
A regra de precedência (normativo > FAQ) é implementada em código na camada de seleção de chunks: após o retrieval, se existirem chunks normativos e chunks de FAQ para o mesmo `topic_tag`, apenas os normativos são passados ao LLM. Isso é determinístico. O risco híbrido ocorre quando **nenhum chunk normativo existe para o tema** — nesse caso, apenas chunks de FAQ são passados ao LLM com `informal: true`, e o estado `COM_RESSALVA` é atribuído em código. Porém, o LLM ainda pode gerar texto que extrapola além do FAQ recebido, misturando conhecimento interno do modelo com o conteúdo do chunk — o que configura uma mesclagem não detectável estruturalmente.

---

#### NAO-05 `[HBR]`
Não há filtro explícito por número de classe ANTT no índice vetorial — os chunks da POL-001 §3.2 descrevem classes 1-6 por nome (explosivos, gases, etc.), não necessariamente com o número da classe como campo de metadado indexado. A garantia parcial vem do fato de que não há documentos sobre classes 7-9 na base — então o retrieval provavelmente retornará score baixo para queries sobre essas classes, acionando GAP_DOCUMENTAL (determinístico via DEVE-05). O risco híbrido é o LLM extrapolar de chunks sobre classes 1-6 para inferir regras sobre classes 7-9 com base em conhecimento interno do modelo. Uma instrução explícita no prompt reduz esse risco, mas não o elimina.

---

#### NAO-06 `[PRB]`
A ausência do valor base do frete na base documental cria uma **barreira estrutural parcial**: o LLM não pode calcular o valor final porque não tem o dado. No entanto, LLMs têm capacidade de alucinação numérica — podem fabricar um valor base plausível e apresentar um cálculo completo como se fosse fundamentado. Essa capacidade existe independentemente de qualquer instrução. A única mitigação estrutural possível seria um guard de output que detecte padrões de valor monetário calculado na resposta (regex sobre `R\$ \d+,\d+`) e bloqueie — mas isso não está especificado no requirements. Sem esse guard, o guardrail depende inteiramente do prompt.

---

#### NAO-07 `[DET]`
O endpoint não tem integração provisionada com nenhum sistema que contenha PII de clientes (CRM, ERP, Portal do Cliente). A "proibição" é, na prática, uma impossibilidade arquitetural — o LLM não pode acessar dados que não estão no índice vetorial e não chegam no request. Não é uma instrução ao modelo; é uma restrição de conectividade. A única superfície de risco seria se o atendente digitasse PII diretamente na query — mas nesse caso o dado seria processado pelo LLM como texto livre, não "acessado" pelo sistema. Essa superfície está fora do escopo deste guardrail.

---

#### NAO-08 `[DET]`
Mesmo raciocínio do NAO-07. O endpoint não tem integração com Azure DevOps. O LLM não pode retornar o status real de um chamado porque esse dado não existe em nenhuma fonte acessível ao sistema. A "proibição" é uma consequência da ausência de integração, não uma instrução ao modelo. O risco residual seria o LLM inventar um status — mas isso é coberto pelo NAO-02 (proibição de extrapolar), que tem categoria `[HBR]`.

---

#### NAO-09 `[PRB]`
O chunk do FAQ Item 27 (que menciona R$50K) pode ser recuperado pelo retrieval quando a query envolve "valor da carga" ou "prioridade". O estado `COM_RESSALVA` seria acionado em código (chunk informal), mas o **conteúdo do disclaimer padrão** não impede especificamente o uso do limiar errado. O LLM pode citar R$50K como informação do FAQ com ressalva e o atendente pode agir sobre ela. A única mitigação estrutural possível seria marcar o chunk do FAQ Item 27 com metadado `superseded_by: SLA-2024-§3` e bloquear sua recuperação quando um chunk do SLA-2024 com o mesmo `attribute_key` existir — mas esse mecanismo não está implementado no requirements atual. Sem ele, o guardrail depende de instrução no prompt.

---

#### NAO-10 `[DET]`
Mesmo raciocínio dos NAO-07 e NAO-08. Ausência de integração com Portal do Cliente e Azure DevOps. O LLM não tem ferramentas (function calls, API calls) disponíveis para executar ações — apenas gera texto. A orientação ao atendente sobre como abrir um chamado é texto, não ação. É uma restrição de arquitetura, não uma instrução ao modelo.

---

#### NAO-11 `[PRB]`
O FAQ Item 22 pode ser recuperado e o estado `COM_RESSALVA` é acionado em código (determinístico). O disclaimer padrão ("fonte informal, confirme com supervisor") é injetado como template (determinístico). Porém, a instrução adicional e específica de "verifique com Compliance" — necessária dado o risco jurídico dos percentuais de seguro — não é gerada em código: depende de o LLM identificar que o tema é seguro de carga e incluir a instrução adicional. Enquanto a OQ-02 não for resolvida e não houver template específico para esse tema, o guardrail permanece probabilístico nessa parte crítica.

---

#### NAO-12 `[DET]`
A decisão de acionar `CONTRADIÇÃO_DETECTADA` é feita antes da chamada ao LLM pelo mecanismo ADR-006. Quando a contradição é detectada, o LLM recebe: (a) os metadados dos documentos conflitantes (doc_id, versão, attribute_key, valores divergentes) e (b) um template de prompt instruindo-o a gerar o texto de alerta. O LLM **não recebe os dois chunks com conteúdo completo** para comparar e decidir — recebe a conclusão já processada. Isso inverte a responsabilidade: o LLM não decide se há contradição, apenas redige o comunicado de uma contradição já detectada. A decisão é determinística; apenas a redação é probabilística.

---

#### DUVIDA-01 `[HBR]`
A **atribuição de domínio único** quando há palavras-chave de múltiplos domínios é determinística: o classificador RN-09 conta palavras-chave por domínio e seleciona o de maior contagem (empate → `nao_classificado`). Isso garante que o limiar e o encaminhamento corretos sejam aplicados. Porém, a instrução de "tratar cada domínio identificado separadamente na resposta e não mesclar regras" depende do LLM: o classificador atribui um domínio, mas a query pode ainda conter entidades de múltiplos domínios nos chunks recuperados. O LLM precisa ser instruído a manter a separação no texto gerado — não há validação estrutural do conteúdo da resposta que garanta isso.

---

#### DUVIDA-02 `[HBR]`
A **ausência do campo `tier_hint`** é detectável em código: o schema define o campo como opcional, e a camada de orquestração verifica sua presença antes de montar o prompt. Se ausente e a query envolver SLA, a orquestração pode injetar no contexto do LLM: "tier do cliente não informado — responda para todos os tiers". Essa injeção é determinística. Porém, o comportamento de "responder para todos os tiers de forma clara e completa sem omitir nenhum" depende do LLM seguir a instrução — probabilístico. O LLM pode, por exemplo, responder apenas para Gold (o tier com SLA mais restritivo) assumindo que é o caso mais relevante.

---

#### DUVIDA-03 `[DET]`
Quando o classificador RN-09 não encontra palavras-chave de nenhum dos três domínios definidos, o fallback para `nao_classificado` é o comportamento padrão do código — equivalente a um `else` no switch. O limiar 0,70 e o encaminhamento "Consulte seu supervisor" são lidos da tabela RN-10 por lookup de `nao_classificado`. Nenhum LLM é chamado para tomar essa decisão. A query ainda segue para o retrieval com os parâmetros padrão, e o resultado do retrieval determina o estado (DEVE-05).

---

#### DUVIDA-04 `[HBR]`
O estado `COM_RESSALVA` quando o FAQ é a única fonte é determinístico (DEVE-03). O disclaimer padrão é injetado como template em código. Porém, a instrução adicional "verifique com Compliance" é específica para categorias de risco jurídico — e identificar que um tema é "risco jurídico" requer ou (a) uma lista explícita de `topic_tag` de alto risco configurada na tabela de encaminhamentos (tornando-a determinística), ou (b) o LLM inferir que o tema é sensível (probabilístico). Atualmente, a RN-10 não inclui uma coluna de "nível de risco jurídico" por domínio — a instrução adicional depende do LLM.

---

#### DUVIDA-05 `[PRB]`
Identificar que uma data mencionada na query foi fornecida pelo cliente (e não é a data do sistema de tracking) requer compreensão semântica do texto — não há campo estruturado no request que distinga as duas. O LLM precisa inferir, a partir do contexto da pergunta ("o cliente disse que recebeu no dia X", vs. "o sistema mostra data X"), que a data é proveniente do cliente. Essa inferência é probabilística. Não há como expressar essa distinção como uma operação de código sem processar linguagem natural — o que nos remete ao LLM.

---

#### DUVIDA-06 `[DET]`
A comparação entre o score do chunk mais relevante e os limiares (limiar do domínio e 0,50) é aritmética simples feita em código após o retrieval. O estado `GAP_DOCUMENTAL` é atribuído pela mesma lógica do DEVE-05. O registro do score máximo no campo de rastreabilidade é uma escrita em log pela camada de orquestração. Nenhuma dessas operações envolve o LLM. A "zona cinza" (0,50 a limiar) é apenas um sub-intervalo da condição de GAP_DOCUMENTAL — a lógica é a mesma.

---

#### DUVIDA-07 `[DET]`
A detecção de reinício do serviço é implícita: ao tentar recuperar uma chave de sessão no Redis após reinício, a operação retorna `null` (chave não encontrada). A camada de orquestração trata `null` como sessão inexistente, injeta a mensagem de encerramento como texto fixo e cria uma nova sessão. Essa lógica é idêntica ao DEVE-13 — a diferença é apenas a causa da chave não existir (expiração por TTL vs. perda por reinício). Em ambos os casos, o comportamento do código é o mesmo: `if (session == null) → inject_expiry_message()`.

---

#### DUVIDA-08 `[HBR]`
A detecção de contradição entre FAQ Item 45 (10 fretes/mês) e PROC-042 v2 (8 fretes/mês) depende de ambos os chunks terem `attribute_key: desconto_volume` atribuído na ingestão. Se esse metadado existir em ambos, a detecção é completamente determinística (ADR-006) e o estado `CONTRADIÇÃO_DETECTADA` é acionado em código. O risco híbrido é que o FAQ, sendo um documento informal com estrutura não padronizada, pode não receber `attribute_key` na ingestão — especialmente se BC-01 não tiver regras explícitas de atribuição de metadados para documentos informais. Sem `attribute_key`, a contradição não é detectada e o LLM recebe os dois chunks, podendo ou não sinalizar o conflito dependendo do prompt.

---

#### DUVIDA-09 `[PRB]`
Decompor uma query em sub-temas e responder parcialmente para um sub-tema enquanto retorna GAP_DOCUMENTAL para outro requer que o LLM: (1) identifique que a pergunta contém múltiplos sub-temas; (2) processe cada um independentemente; (3) combine as respostas em uma única resposta estruturada. Esse comportamento de decomposição de query não tem representação estrutural no pipeline atual — o retrieval opera sobre a query completa, não sobre sub-temas decompostos. Sem um mecanismo de query decomposition em código, o guardrail depende inteiramente de instrução no prompt.

---

#### DUVIDA-10 `[PRB]`
Detectar ambiguidade entre dois termos da linguagem ubíqua ("Coleta Reversa" vs. "Frete Reverso") requer interpretação semântica da query. A normalização RN-09 (DEVE-07) converte "coleta de devolução" → "Coleta Reversa", mas não resolve o caso em que o atendente usa termos genéricos como "devolução" ou "frete de retorno" sem especificar qual conceito está perguntando. O retrieval pode retornar chunks de ambos os conceitos com scores similares — mas a decisão de "responder sobre ambos explicitando a diferença" em vez de escolher um é uma instrução ao LLM, não uma regra de código. Não há heurística estrutural que distinga uma query ambígua de uma query específica sem processar linguagem natural.

| Guardrail | Enforcement | Categoria | Origem principal | UO relacionado | Risco se violado |
|---|---|---|---|---|---|
| DEVE-01 | `[DET]` | Estado obrigatório | §5 Estados | UO-01, UO-02 | Atendente sem base para avaliar confiança |
| DEVE-02 | `[HBR]` | Citação de fonte | ADR-003, AC-01 | UO-01, UO-02 | Informação não verificável |
| DEVE-03 | `[HBR]` | Ressalva FAQ | §5 Estado 2, AC-02 | UO-02 | Informação não validada repassada como definitiva |
| DEVE-04 | `[HBR]` | Sinalização de contradição | ADR-004, ADR-006, AC-03 | UO-03 | Valor errado comunicado com aparente segurança |
| DEVE-05 | `[DET]` | GAP_DOCUMENTAL + encaminhamento | §5 Estado 4, RN-10 | UO-04 | Resposta fabricada ou atendente sem direção |
| DEVE-06 | `[PRB]` | Distinção relógio SLA | RN-06, AC-08, AC-09 | UO-02 | Prazo errado de SLA comunicado ao cliente |
| DEVE-07 | `[DET]` | Normalização de termos | RN-09, E1, E3 | UO-01 | Miss na busca vetorial — resposta incorreta |
| DEVE-08 | `[DET]` | Isolamento de sessão | RN-08 | UO-05 | Contexto vazado entre atendentes |
| DEVE-09 | `[DET]` | PROC-042 v2 exclusivo | RN-01, AC-04 | UO-03 | Multiplicador desatualizado — erro de até 12,5% |
| DEVE-10 | `[DET]` | conflicting_docs no response | §5 Estado 3, ADR-006 | — | Rastreabilidade de contradições inoperante |
| DEVE-11 | `[DET]` | domain_classified no response | RN-09, §5 Estado 4 | — | BC-05 não categoriza gaps |
| DEVE-12 | `[DET]` | Comportamento em timeout | RNF-01, ADR-005 | UO-01 | Atendente sem feedback; histórico perdido |
| DEVE-13 | `[DET]` | Mensagem de sessão expirada | RN-08, E7 | UO-05 | Resposta incoerente sem aviso |
| DEVE-14 | `[PRB]` | Platinum = FUNDAMENTADA | RN-05, AC-10 | UO-04 | GAP_DOCUMENTAL falso — doc existe |
| DEVE-15 | `[DET]` | Config externalizada | RN-09, RN-10 | — | Mudança de contato exige redeploy |
| NAO-01 | `[HBR]` | Proibido escolher entre conflitantes | ADR-004, AC-03 | UO-03 | Valor errado apresentado como correto |
| NAO-02 | `[HBR]` | Proibido extrapolar | §5 Estado 4, UO-04 | UO-04 | Resposta fabricada sem sinal de alerta |
| NAO-03 | `[DET]` | Proibido usar PROC-042 v1 | RN-01, AC-04 | UO-03 | Multiplicadores desatualizados |
| NAO-04 | `[HBR]` | Proibido mesclar normativo + FAQ | RN-02, E5 | UO-02 | Autoridade de fonte misturada sem sinalização |
| NAO-05 | `[HBR]` | Proibido responder classes 7-9 ANTT | RN-07, AC-06 | UO-04 | Orientação incorreta sobre carga regulada |
| NAO-06 | `[PRB]` | Proibido calcular valor final do frete | §3.2, §11 | — | Valor calculado com base desatualizada |
| NAO-07 | `[DET]` | Proibido acessar PII | §3.3 | — | Violação de privacidade / LGPD |
| NAO-08 | `[DET]` | Proibido medir SLA em tempo real | §3.2, §11 | — | Dado desatualizado como se fosse atual |
| NAO-09 | `[PRB]` | Proibido usar limiar R$50K do FAQ | RN-03 | — | Escaladas desnecessárias |
| NAO-10 | `[DET]` | Proibido executar ações em chamados | §3.2, §11 | — | Ações irreversíveis sem supervisão |
| NAO-11 | `[PRB]` | Seguro de carga: ressalva obrigatória | OQ-02, E6 | UO-02 | Percentual incorreto comunicado ao cliente |
| NAO-12 | `[DET]` | LLM não decide contradição | ADR-006 | UO-03 | Contradição resolvida silenciosamente |
| DUVIDA-01 | `[HBR]` | Query com múltiplos domínios | RN-09, E2 | UO-01 | Regras de domínios diferentes mescladas |
| DUVIDA-02 | `[HBR]` | tier_hint ausente | RN-06, OQ-05 | UO-02 | SLA calculado para tier errado |
| DUVIDA-03 | `[DET]` | Domínio não classificado | RN-09, RN-10 | UO-04 | Consulta bloqueada desnecessariamente |
| DUVIDA-04 | `[HBR]` | FAQ como única fonte | RN-02, E6 | UO-02 | Silêncio quando havia conteúdo disponível |
| DUVIDA-05 | `[PRB]` | Prazo de devolução com data do cliente | E4 | UO-02 | Prazo contado a partir da data errada |
| DUVIDA-06 | `[DET]` | Score em zona cinza (0,50–limiar) | §5 Estado 4 | UO-04 | Chunk de baixa relevância usado na geração |
| DUVIDA-07 | `[DET]` | Reinício de serviço mid-session | RN-08 | UO-05 | Contexto reconstruído incorretamente |
| DUVIDA-08 | `[HBR]` | Contradição FAQ × normativo em desconto | E8, RN-02 | UO-03 | Contradição tratada como ressalva |
| DUVIDA-09 | `[PRB]` | Frete padrão como sub-tema | RN-04, AC-05 | UO-04 | Toda resposta bloqueada por sub-tema sem cobertura |
| DUVIDA-10 | `[PRB]` | Ambiguidade Coleta Reversa × Frete Reverso | Linguagem ubíqua | UO-02 | Atendente orientado sobre o conceito errado |

---

## Open Questions que afetam guardrails ativos

Os guardrails abaixo têm comportamento provisório até a resolução das OQs indicadas.
Após resolução, este documento deve ser atualizado.

| Guardrail provisório | OQ pendente | Comportamento atual | Comportamento possível após decisão |
|---|---|---|---|
| NAO-11 (seguro de carga) | OQ-02 | Ressalva obrigatória + verificação com supervisor | Se NovaTech criar doc normativo: FUNDAMENTADA sem ressalva |
| DUVIDA-04 (FAQ como única fonte) | OQ-01 | COM_RESSALVA quando FAQ ingerido | Se FAQ não for ingerido: GAP_DOCUMENTAL + encaminhamento |
| DUVIDA-02 (tier_hint ausente) | OQ-05, OQ-06 | Responder para todos os tiers | Se bot preencher automaticamente: responder para o tier específico |

---

## SEÇÃO 5 — Conexão com Incidentes Simulados

### Incidentes de referência

| ID | Descrição |
|---|---|
| **INC-01** | O assistente respondeu que o prazo de devolução para carga perigosa é 7 dias, quando na verdade cargas perigosas **não podem ser devolvidas** pelo processo padrão. |
| **INC-02** | O assistente citou "PROC-042, seção 2" mas os multiplicadores informados eram da **versão 1 (desatualizada)**, não da v2 (vigente). |
| **INC-03** | O assistente disse "Não encontrei informação sobre isso" para uma pergunta sobre **SLA Gold**, mas o documento SLA-2024 estava indexado e continha a resposta. |

---

### 5.1 — Causa-raiz dos incidentes

#### INC-01 — "Prazo de 7 dias para carga perigosa"

**O que aconteceu no pipeline:**
O retrieval recuperou o chunk da POL-001 §3.1 (prazo geral de 7 dias úteis), mas não recuperou — ou ignorou — o chunk da POL-001 §3.2 (exceções, incluindo cargas perigosas classes 1-6). O LLM gerou a resposta com base apenas no chunk do prazo geral, sem conhecimento da exceção existente no mesmo documento.

**Tipo de falha:** resposta parcialmente correta que omite a regra de exceção crítica. O LLM não inventou nada — respondeu com um chunk real. O problema foi o retrieval não retornar o chunk de exceção junto, ou o LLM não sinalizar que estava respondendo sem verificar exceções.

**Causa raiz principal:** ausência de mecanismo que force a recuperação de chunks de exceção quando o chunk de regra geral é recuperado para o mesmo `topic_tag`.

---

#### INC-02 — "Multiplicadores da v1 com citação de v2"

**O que aconteceu no pipeline:**
O chunk recuperado pertencia ao PROC-042 v1 (com metadado `doc_id: PROC-042-v1`), mas o campo `sources[].version` no response exibiu "v2". Isso indica uma de duas falhas: (a) o filtro de índice `status: obsoleto` não foi aplicado e o chunk da v1 foi recuperado; (b) o campo `sources` foi populado incorretamente — com a versão do documento mais recente disponível, não a versão do chunk efetivamente usado.

**Tipo de falha:** falha de integridade entre o conteúdo gerado e os metadados de rastreabilidade. O atendente recebeu uma citação confiável ("PROC-042 v2") para um conteúdo desatualizado — o pior cenário possível: aparência de correção com conteúdo errado.

**Causa raiz principal:** ou o filtro de índice não estava ativo, ou o pipeline de população do campo `sources` não usava o `doc_id` do chunk efetivamente recuperado.

---

#### INC-03 — "Não encontrei informação sobre SLA Gold"

**O que aconteceu no pipeline:**
O documento SLA-2024 estava indexado, mas o score de similaridade entre a query e os chunks do SLA-2024 ficou abaixo do limiar do domínio `sla_contrato` (0,78). O sistema retornou `GAP_DOCUMENTAL` — tecnicamente correto dado o limiar, mas incorreto do ponto de vista do usuário. A query provavelmente usou "Gold" sem normalização adequada para os termos canônicos do SLA-2024, resultando em score degradado.

**Tipo de falha:** falso GAP_DOCUMENTAL causado por limiar mal calibrado e/ou falha na normalização de termos. O documento existia; o sistema não o encontrou.

**Causa raiz principal:** o classificador RN-09 não normalizou "Gold" para o contexto semântico correto antes do embedding, e o limiar do domínio `sla_contrato` estava alto demais para a variabilidade lexical das queries sobre SLA.

---

### 5.2 — Mapeamento guardrail × incidente

Para cada guardrail, a conexão indica se ele **previne** o incidente (atuação antes da falha) ou **detecta/mitiga** (atuação após a falha ter ocorrido, reduzindo impacto).

| Guardrail | Enforcement | INC-01 | INC-02 | INC-03 | Como conecta |
|---|---|---|---|---|---|
| DEVE-01 | `[DET]` | ✅ Previne | ✅ Previne | — | INC-01: estado correto exigiria que a ausência do chunk de exceção gerasse COM_RESSALVA ou escalada, não FUNDAMENTADA. INC-02: state FUNDAMENTADA seria bloqueado se sources fosse inválido. |
| DEVE-02 | `[HBR]` | — | ✅ Previne | — | A validação de schema garante que `sources[].version` seja populado a partir do `doc_id` do chunk efetivamente recuperado — não do documento mais recente da base. Se implementado corretamente, teria exposto a inconsistência. |
| DEVE-03 | `[HBR]` | ✅ Mitiga | — | — | Se o chunk de prazo geral fosse de FAQ (não era, era normativo), a ressalva teria sinalizado ao atendente para verificar antes de repassar. Não previne diretamente pois a POL-001 é normativo. |
| DEVE-04 | `[HBR]` | ✅ Previne | ✅ Previne | — | INC-01: se §3.1 e §3.2 tivessem `attribute_key` conflitante para "carga perigosa + prazo", teria acionado CONTRADIÇÃO_DETECTADA. INC-02: se v1 e v2 tivessem sido ambas recuperadas, teria detectado conflito de multiplicadores. |
| DEVE-05 | `[DET]` | — | — | ✅ Previne parcialmente | Garante que GAP_DOCUMENTAL só é acionado se o score estiver abaixo do limiar — o incidente ocorreu porque o limiar estava mal calibrado, não porque DEVE-05 falhou. DEVE-05 funcionou como especificado; o problema era o parâmetro. |
| DEVE-06 | `[PRB]` | — | — | ✅ Previne | Se a resposta sobre SLA Gold tivesse sido gerada, este guardrail exigiria que ela distinguisse o comportamento do relógio por tipo de chamado — reduzindo o risco de informação incompleta sobre SLA. |
| DEVE-07 | `[DET]` | — | — | ✅ Previne | A normalização de "Gold" → "Tier de Cliente" antes do embedding aumenta a similaridade semântica com os chunks do SLA-2024, elevando o score acima do limiar. Causa raiz do INC-03 inclui a ausência dessa normalização. |
| DEVE-09 | `[DET]` | — | ✅ Previne | — | O filtro `status: obsoleto` impede que chunks do PROC-042 v1 sejam recuperados. Se ativo, o INC-02 não teria ocorrido — o chunk da v1 nunca teria chegado ao LLM. |
| DEVE-10 | `[DET]` | — | ✅ Detecta | — | Se `conflicting_docs` fosse populado com v1 e v2 no mesmo retrieval, o response teria exposto a coexistência das duas versões — sinal para o atendente e para BC-05 que algo estava errado. |
| DEVE-11 | `[DET]` | — | — | ✅ Previne | O campo `domain_classified` correto garante que o limiar do domínio `sla_contrato` seja aplicado à query de SLA Gold. Se o domínio fosse classificado incorretamente como `nao_classificado`, o limiar 0,70 (menor) poderia ter retornado o chunk correto — o INC-03 pode ter sido agravado por classificação incorreta de domínio. |
| NAO-01 | `[HBR]` | ✅ Previne | ✅ Previne | — | INC-01: impede que o LLM escolha entre chunk de regra geral e chunk de exceção apresentando apenas um como correto. INC-02: impede síntese entre v1 e v2 sem sinalização. |
| NAO-02 | `[HBR]` | ✅ Previne | — | — | Impede o LLM de responder sobre prazo de carga perigosa extrapolando além dos chunks recuperados. Se o chunk de exceção não foi recuperado, o correto seria não responder sobre o caso específico de carga perigosa — não generalizar o prazo geral. |
| NAO-03 | `[DET]` | — | ✅ Previne | — | Causa raiz direta do INC-02: se o filtro estivesse ativo, o chunk da v1 nunca teria sido recuperado e os multiplicadores desatualizados nunca teriam chegado ao LLM. |
| NAO-05 | `[HBR]` | ✅ Previne | — | — | Impede o sistema de responder sobre carga perigosa com chunks que não cobrem o caso específico perguntado. Para INC-01, a cobertura é classes 1-6 — e a exceção de não-elegibilidade ao processo padrão está no §3.2, que deveria ter sido recuperado junto. |
| NAO-12 | `[DET]` | — | ✅ Previne | — | Se v1 e v2 coexistissem no índice (cenário do INC-02 antes do filtro ser aplicado), a detecção determinística teria acionado CONTRADIÇÃO_DETECTADA antes da geração — o LLM não teria recebido os chunks para sintetizar a resposta com multiplicadores errados. |
| DUVIDA-05 | `[PRB]` | ✅ Mitiga | — | — | Não previne o INC-01 diretamente, mas ao exigir que a resposta sobre prazo inclua o aviso sobre a data de referência, sinaliza ao atendente que deve verificar condições adicionais antes de comunicar o prazo ao cliente. |
| DUVIDA-06 | `[DET]` | — | — | ✅ Detecta | O registro do score máximo obtido na zona cinza em BC-05 teria revelado que queries sobre SLA Gold estavam com scores sistematicamente baixos — sinal para recalibrar o limiar do domínio `sla_contrato` antes que o INC-03 se tornasse recorrente. |
| DUVIDA-09 | `[PRB]` | ✅ Mitiga | — | — | Ao exigir que o sistema responda parcialmente para sub-temas cobertos, reduz o risco de o atendente receber uma resposta sobre prazo geral sem a ressalva de que cargas perigosas são tratadas diferentemente. |

---

### 5.3 — Cobertura por incidente

#### INC-01 — "Prazo de 7 dias para carga perigosa"
**Guardrails que previnem:** DEVE-01, DEVE-04, NAO-01, NAO-02, NAO-05
**Guardrails que mitigam:** DEVE-03, DUVIDA-05, DUVIDA-09
**Guardrails que não cobrem:** todos os demais — não relacionados ao domínio de devolução e carga perigosa

**Análise de cobertura:**
8 guardrails cobrem este incidente, mas nenhum deles sozinho o previne completamente. A falha é estrutural: o pipeline não garante que, ao recuperar um chunk de regra geral, o chunk de exceção correspondente seja também recuperado. Nenhum guardrail atual especifica esse comportamento — é uma lacuna de requisito, não uma falha de implementação de guardrail existente.

**Lacuna identificada:** falta um guardrail explícito do tipo `DEVE-XX: quando o chunk recuperado for de regra geral (scope: regra), o pipeline deve obrigatoriamente buscar chunks do mesmo topic_tag com scope: exceção antes de gerar a resposta`. Este comportamento não está em nenhum guardrail atual.

---

#### INC-02 — "Multiplicadores da v1 com citação de v2"
**Guardrails que previnem:** DEVE-09, NAO-03, NAO-12
**Guardrails que detectam:** DEVE-02, DEVE-04, DEVE-10, NAO-01
**Guardrails que não cobrem:** todos os demais — não relacionados ao controle de versão

**Análise de cobertura:**
Este incidente é o **mais coberto** dos três. Três guardrails determinísticos previnem a causa raiz (filtro de índice + detecção de contradição + campo sources a partir do doc_id real). Se todos os `[DET]` estivessem implementados corretamente, o INC-02 não ocorreria. O fato de ter ocorrido indica que pelo menos DEVE-09 ou NAO-03 não estavam ativos — a implementação não respeitou o requisito.

**Lacuna identificada:** ausência de teste de integração que valide que o campo `sources[].version` corresponde ao `doc_id` do chunk efetivamente usado na geração — e não ao documento mais recente da base. Este é um caso de teste faltante, não um guardrail faltante.

---

#### INC-03 — "Falso GAP_DOCUMENTAL para SLA Gold"
**Guardrails que previnem:** DEVE-07, DEVE-11
**Guardrails que detectam:** DUVIDA-06
**Guardrails que cobrem parcialmente:** DEVE-05, DEVE-06
**Guardrails que não cobrem:** todos os demais — não relacionados ao retrieval de SLA

**Análise de cobertura:**
Este incidente é o **menos coberto** — apenas 2 guardrails o previnem diretamente. A causa raiz (limiar mal calibrado + normalização insuficiente de termos de SLA) é endereçada por DEVE-07 e DEVE-11, mas nenhum dos dois garante que o score do chunk correto ficará acima do limiar após a normalização. DUVIDA-06 detectaria o problema em BC-05 — mas apenas depois que o incidente já ocorreu.

**Lacuna identificada:** falta um guardrail explícito para validação periódica dos limiares por domínio — algo como `DEVE-XX: BC-05 deve emitir alerta quando a taxa de GAP_DOCUMENTAL para um domínio específico ultrapassar X% das consultas em 7 dias`. O INC-03 se tornaria detectável proativamente antes de impactar atendentes.

---

### 5.4 — Lacunas de cobertura identificadas

A análise dos incidentes revelou **3 lacunas** não cobertas por nenhum guardrail existente:

| # | Lacuna | Incidente que expôs | Guardrail sugerido |
|---|---|---|---|
| L-01 | Ausência de busca obrigatória de chunks de exceção quando chunk de regra geral é recuperado | INC-01 | `DEVE-16: quando chunks com scope:regra_geral forem recuperados, o pipeline deve obrigatoriamente executar busca secundária por chunks com scope:excecao no mesmo topic_tag antes de gerar a resposta` |
| L-02 | Ausência de teste de integridade entre doc_id do chunk recuperado e version no campo sources | INC-02 | Não é um guardrail de runtime — é um caso de teste de integração obrigatório: `assert response.sources[i].version == chunk_retrieved[i].doc_version` |
| L-03 | Ausência de alerta proativo de degradação de retrieval por domínio | INC-03 | `DEVE-17: BC-05 deve emitir alerta quando taxa de GAP_DOCUMENTAL para um domínio ultrapassar 15% das consultas em janela de 7 dias corridos` |

