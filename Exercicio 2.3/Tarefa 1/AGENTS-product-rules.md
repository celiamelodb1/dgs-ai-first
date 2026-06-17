# AGENTS.md — NovaTech Assistant
> Constitution do projeto. Leia este arquivo integralmente antes de executar qualquer tarefa.
> Fonte de verdade para comportamento do agente, convenções do repositório e regras do domínio.

---

## Product Rules & Guardrails

> Esta seção define as regras de comportamento do assistente NovaTech durante geração de
> código, testes, prompts e qualquer artefato que afete o comportamento em runtime.
> Derivada de `guardrails.md v1.3.0` e das fontes documentais em `docs/novatech/`
> (Anexo A). As regras aqui são **vinculantes** — não são sugestões.
>
> Se você está escrevendo código para `src/services/`, `src/functions/query/` ou
> `prompts/system-prompt.md`, esta seção se aplica a você agora.

---

### PG-01 — Todo response tem um estado explícito

O campo `state` é obrigatório em toda resposta do query endpoint. Os únicos valores
válidos são: `FUNDAMENTADA`, `COM_RESSALVA`, `CONTRADIÇÃO_DETECTADA`, `GAP_DOCUMENTAL`.

Ao escrever código em `src/functions/query/response-builder.ts` ou
`src/shared/types.ts`, o tipo de retorno deve refletir essa constraint:

```typescript
type ResponseState =
  | 'FUNDAMENTADA'
  | 'COM_RESSALVA'
  | 'CONTRADIÇÃO_DETECTADA'
  | 'GAP_DOCUMENTAL';
```

**Nunca** omita o campo `state` no response schema. **Nunca** crie estados além dos
quatro acima. Se o pipeline não conseguir determinar o estado, o fallback é
`GAP_DOCUMENTAL` — nunca retornar sem estado.

> Derivado de: `DEVE-01` · `specs/query-endpoint/requirements.md §5`

---

### PG-02 — PROC-042 v1 é obsoleto — filtro de índice é obrigatório

O PROC-042 v1 (`PROC-042-frete-especial-v1.md`) **não deve ser recuperado em nenhuma
consulta**. Ao implementar o pipeline de ingestão em `src/pipeline/indexer.ts`, o
documento deve ser indexado com o metadado `status: 'obsoleto'`.

O retrieval em `src/services/search.ts` deve aplicar o filtro:

```typescript
// Azure AI Search filter — nunca remova esta cláusula
const filter = "status ne 'obsoleto'";
```

**Não existe exceção para chamados históricos** nesta implementação. A disposição
transitória do PROC-042 v2 §5 não é suportada pelo endpoint — chamados pré-01/12/2023
devem ser tratados manualmente fora do sistema.

Se você encontrar código que recupera chunks sem o filtro `status`, é um bug.
Abra um issue antes de prosseguir.

> Derivado de: `DEVE-09` · `NAO-03` · `specs/query-endpoint/requirements.md RN-01`
> Incidente de referência: INC-02 — "multiplicadores da v1 com citação de v2"

---

### PG-03 — Contradição é detectada por metadados, não pelo LLM

A decisão de acionar `CONTRADIÇÃO_DETECTADA` é feita pela camada de orquestração em
`src/services/search.ts` ou `src/functions/query/handler.ts`, **antes** da chamada
ao LLM em `src/services/completion.ts`.

O mecanismo: dois chunks são conflitantes quando `topic_tag` é idêntico, `doc_id` é
diferente, e `attribute_key` é idêntico com valores divergentes.

```typescript
// Em src/services/search.ts
function detectConflict(chunks: Chunk[]): ConflictResult | null {
  // Agrupar por topic_tag + attribute_key
  // Se mesmo attribute_key com valores distintos em doc_ids diferentes → conflito
  // Retornar os doc_ids envolvidos para popular conflicting_docs no response
}
```

**O LLM nunca recebe dois chunks conflitantes para sintetizar.** Quando conflito é
detectado, o LLM recebe apenas os metadados (doc_ids, attribute_keys, valores
divergentes) e um template de alerta para redigir.

Ao escrever testes em `tests/unit/` para este serviço, cubra obrigatoriamente:
- Dois chunks com mesmo `attribute_key`, `doc_id` diferentes → conflito detectado
- Dois chunks com mesmo `attribute_key`, mesmo `doc_id` → sem conflito
- Chunk único → sem conflito

> Derivado de: `DEVE-04` · `NAO-01` · `NAO-12` · `ADR-006`
> Incidente de referência: INC-02

---

### PG-04 — Fonte informal (FAQ) requer estado COM_RESSALVA e disclaimer injetado em código

O FAQ-Atendimento (`FAQ-atendimento.md`) deve ser indexado com o metadado
`doc_type: 'informal'`. Quando todos os chunks recuperados forem `doc_type: 'informal'`,
o estado `COM_RESSALVA` é atribuído em código — não pelo LLM.

O disclaimer é **injetado programaticamente** pela camada de orquestração — nunca
gerado pelo LLM:

```typescript
// Em src/functions/query/response-builder.ts
const DISCLAIMER_INFORMAL =
  'Fonte: FAQ interno — não validado por Compliance. ' +
  'Confirme com supervisor antes de encaminhar ao cliente.';

if (state === 'COM_RESSALVA') {
  response.disclaimer = DISCLAIMER_INFORMAL;
}
```

**Regra de precedência de fonte:** quando existem chunks normativos (POL/PROC) e chunks
do FAQ cobrindo o mesmo `topic_tag`, apenas os normativos chegam ao LLM. O FAQ é
ignorado. O FAQ só chega ao LLM quando é a **única** fonte para o tema.

> Derivado de: `DEVE-03` · `NAO-04` · `specs/query-endpoint/requirements.md RN-02`

---

### PG-05 — GAP_DOCUMENTAL é determinado por limiar configurável por domínio

O threshold de relevância **não é um valor único global**. É parametrizado por domínio
em `src/shared/config.ts`:

```typescript
export const RETRIEVAL_THRESHOLDS: Record<Domain, number> = {
  frete_logistica:   0.75,
  devolucao_politica: 0.72,
  sla_contrato:      0.78,
  nao_classificado:  0.70,
};
```

Quando nenhum chunk supera o threshold do domínio, o estado é `GAP_DOCUMENTAL`.
O LLM **não é chamado** para geração nesse estado — o response é montado diretamente
pela camada de orquestração com o encaminhamento da tabela de roteamento (PG-10).

Ao registrar um GAP_DOCUMENTAL, o campo `maxScoreObtained` deve ser populado no
response para rastreabilidade em BC-05.

**Atenção:** limiar alto demais gera falsos gaps (INC-03). Os valores acima são pontos
de partida — serão calibrados nas primeiras duas semanas de operação com base nos logs.
Não altere os thresholds sem registrar em `prompts/prompt-changelog.md` com justificativa.

> Derivado de: `DEVE-05` · `DUVIDA-06` · `specs/query-endpoint/requirements.md §5 Estado 4`
> Incidente de referência: INC-03 — "falso GAP_DOCUMENTAL para SLA Gold"

---

### PG-06 — Termos não-canônicos são normalizados antes do embedding

O pré-processador em `src/functions/query/handler.ts` substitui sinônimos não-canônicos
**antes** de gerar o embedding. A tabela de sinônimos vive em
`prompts/synonym-table.json` (configuração versionada, não hardcoded).

Substituições obrigatórias:

| Termo detectado (case-insensitive) | Substituir por |
|---|---|
| frete diferenciado, frete pesado | Frete Especial |
| chamado P1, chamado urgente, prioridade alta, prioridade 1 | Incidente Crítico |
| carga especial, material perigoso | Carga Perigosa |
| categoria de cliente, nível de cliente, Platinum | Tier de Cliente |
| 7 dias corridos, uma semana | Prazo de Devolução |
| coleta de devolução | Coleta Reversa |

A normalização é uma substituição de string exata — não inferência do LLM.
Novos sinônimos identificados em produção são adicionados via PR em
`prompts/synonym-table.json`. Não requerem redeploy.

**Por que isso importa:** a query "qual o SLA para chamado P1 Gold?" sem normalização
pode retornar score baixo para os chunks do SLA-2024 e gerar GAP_DOCUMENTAL falso
(mesma causa raiz do INC-03).

> Derivado de: `DEVE-07` · `specs/query-endpoint/requirements.md RN-09`

---

### PG-07 — Sessão é isolada por user_id AAD e armazenada no Redis

A chave de sessão no Redis tem o formato `session:{userId}:{sessionUuid}`, onde
`userId` é o AAD object ID extraído do token Teams pelo middleware de autenticação.

```typescript
// Em src/shared/config.ts
export const SESSION_TTL_SECONDS = 3600; // 60 minutos

// Chave de sessão — nunca use apenas sessionUuid como chave
const sessionKey = `session:${userId}:${sessionUuid}`;
```

"Inatividade" é definida como ausência de mensagem do atendente — o envio de resposta
pelo bot não reinicia o TTL. Use `EXPIRE` no Redis após cada mensagem recebida,
não após cada resposta enviada.

Quando a chave não existe (expiração ou reinício do serviço), injete a mensagem:
`"Sua sessão anterior foi encerrada. Por favor, reformule sua pergunta com o contexto
necessário."` — antes de processar qualquer query.

**Dois atendentes nunca compartilham sessão.** Se encontrar código que usa apenas
`sessionUuid` como chave (sem `userId`), é um bug de isolamento.

> Derivado de: `DEVE-08` · `DEVE-13` · `specs/query-endpoint/requirements.md RN-08`

---

### PG-08 — O assistente não calcula, não mede e não executa

Três proibições absolutas que não têm exceção e não dependem de instrução no prompt —
são restrições de arquitetura:

**Não calcula valor final de frete.** O endpoint pode orientar sobre a fórmula
(`Valor base × Multiplicador regional × Fator de peso`) e sobre os fatores vigentes
(PROC-042 v2), mas **nunca retorna um valor monetário calculado**. O valor base
é externo ao sistema (tabela mensal em pasta de rede) e não está indexado.

**Não mede SLA em tempo real.** O endpoint responde sobre as regras de SLA
(prazos, tiers, penalidades), mas não sobre o status atual de um chamado específico.
Não há integração com Azure DevOps nesta implementação.

**Não executa ações em chamados.** O endpoint não abre, atualiza nem encerra
chamados no Portal do Cliente ou Azure DevOps. Pode orientar o atendente sobre
como fazê-lo — não faz por ele.

Ao implementar `src/services/response-validator.ts`, inclua guards de output que
detectem e bloqueiem:
- Respostas com padrão `R\$\s*\d+[\.,]\d+` quando o domínio for `frete_logistica`
- Qualquer claim de status de chamado em tempo real

> Derivado de: `NAO-06` · `NAO-08` · `NAO-10` · `specs/query-endpoint/requirements.md §3.2`

---

### PG-09 — Regras de SLA exigem distinção explícita de comportamento

Toda resposta sobre o relógio de SLA deve especificar qual dos dois comportamentos
se aplica. **Nunca** responda sobre SLA sem fazer a distinção:

- **Chamados gerais** (qualquer tier): relógio **pausa** fora do horário comercial
  (08h–18h, dias úteis)
- **Incidentes críticos de clientes Gold**: relógio **não pausa** — corre 24/7

Um incidente é **crítico** quando atende a **pelo menos um** dos critérios do
SLA-2024 §3:
1. Carga com valor declarado acima de **R$ 100.000** com status desconhecido há mais
   de 6 horas — **não R$ 50.000** (FAQ Item 27 usa limiar diferente; ignore-o)
2. Carga perigosa com qualquer irregularidade de documentação ou rastreamento
3. Mais de 5 chamados do mesmo cliente nas últimas 24 horas sobre o mesmo problema
4. Qualquer situação com risco à segurança de pessoas

Ao escrever casos de teste em `tests/unit/` ou queries em `prompts/eval/golden-queries.json`,
inclua obrigatoriamente os cenários: chamado geral Gold (pausa) e incidente crítico
Gold (não pausa). São os dois ACs mais frequentemente confundidos (AC-08 e AC-09).

> Derivado de: `DEVE-06` · `NAO-09` · `specs/query-endpoint/requirements.md RN-06`

---

### PG-10 — Encaminhamentos do GAP_DOCUMENTAL vivem em configuração, não no prompt

A tabela de encaminhamentos é lida de `prompts/routing-table.json`. O e-mail
`sinistros@novatech.com.br` é lido da variável de ambiente `SINISTROS_EMAIL`.
**Nenhum desses valores pode aparecer hardcoded no prompt ou no código.**

Encaminhamentos por domínio:

| Domínio | Encaminhamento |
|---|---|
| `frete_logistica` | "Consulte a área Comercial para informações sobre este tipo de frete." |
| `devolucao_politica` + carga danificada | "Registre a ocorrência em `{SINISTROS_EMAIL}` com fotos e laudo em até 48h." |
| `devolucao_politica` + outros | "Consulte a área de Operações ou acesse o Portal do Cliente." |
| `sla_contrato` | "Consulte o Comercial responsável pela conta do cliente." |
| `nao_classificado` | "Consulte seu supervisor para orientação sobre este tema." |

Mudanças de contato ou área de destino são feitas via PR em `prompts/routing-table.json`
ou via atualização de variável de ambiente — sem redeploy do modelo.

> Derivado de: `DEVE-15` · `DUVIDA-03` · `specs/query-endpoint/requirements.md RN-10`

---

### PG-11 — Cargas perigosas: escopo restrito a classes 1–6 da ANTT

O sistema responde sobre carga perigosa referenciando **exclusivamente as classes 1 a 6**
da ANTT (Res. 5.947/2021), conforme POL-001 §3.2:

| Classe | Tipo |
|---|---|
| 1 | Explosivos |
| 2 | Gases |
| 3 | Líquidos inflamáveis |
| 4 | Sólidos inflamáveis |
| 5 | Oxidantes e peróxidos |
| 6 | Substâncias tóxicas e infectantes |

Perguntas que envolvam classes 7 (radioativos), 8 (corrosivos) ou 9 (miscelânea)
retornam `GAP_DOCUMENTAL`. A documentação não cobre essas classes.

**Cargas perigosas não são elegíveis ao processo padrão de devolução (POL-001 §3.2).**
Nunca responda que o prazo de devolução de 7 dias se aplica a cargas perigosas.
O encaminhamento correto é: Gestão de Riscos, ramal 4500.

> Derivado de: `NAO-05` · `DUVIDA-04`
> Incidente de referência: INC-01 — "prazo de 7 dias para carga perigosa"

---

### PG-12 — Tier Platinum não existe; GAP_DOCUMENTAL é resposta incorreta para essa pergunta

Quando o atendente perguntar sobre tier Platinum, a resposta correta é `FUNDAMENTADA`
— não `GAP_DOCUMENTAL`. O SLA-2024 §1 documenta explicitamente que existem apenas
três tiers: Gold, Silver e Standard. O Platinum foi descontinuado em 2022.

Resposta esperada: informar a descontinuação + listar os três tiers vigentes +
orientar o atendente a verificar o tier real do cliente pelo número do contrato.

Inclua este caso em `prompts/eval/golden-queries.json` como golden query de
regressão — é um dos ACs mais simples e mais fáceis de regredir (AC-10).

> Derivado de: `DEVE-14` · `specs/query-endpoint/requirements.md RN-05`

---

### PG-13 — Prazo de devolução: data de referência é o tracking, não o cliente

O prazo de 7 dias úteis (POL-001 §3.1) é contado a partir da **data de recebimento
confirmada no sistema de tracking** — não da data informada pelo cliente.

Quando a resposta envolver prazo de devolução e a query mencionar uma data específica
("o cliente disse que recebeu no dia X"), o sistema deve incluir o aviso:
*"O prazo de 7 dias úteis é contado a partir da data de recebimento confirmada no
sistema de tracking — não necessariamente da data informada pelo cliente. Verifique
a data no sistema antes de comunicar o prazo."*

Inclua este caso em `tests/fixtures/queries.ts` como edge case de prazo.

> Derivado de: `DUVIDA-05` · `POL-001 §3.1`

---

### PG-14 — Coleta Reversa ≠ Frete Reverso

São conceitos distintos. Quando a query for ambígua sobre qual dos dois se refere,
o sistema deve responder sobre ambos com definição explícita:

- **Coleta Reversa** — o *serviço* de retirada da mercadoria devolvida no endereço
  do cliente. Agendado em até 2 dias úteis após aprovação da solicitação (POL-001 §3.3).
- **Frete Reverso** — o *custo* do transporte de retorno. Responsabilidade variável:
  sem custo ao cliente quando o motivo é erro/avaria da NovaTech; custo do cliente
  quando é desistência (POL-001 §3.5).

Nunca use "frete reverso" para se referir ao serviço de retirada, nem "coleta reversa"
para se referir ao custo.

> Derivado de: `DUVIDA-10` · `POL-001 §3.3 e §3.5`

---

### PG-15 — Seguro de carga: fonte informal, resposta condicional

O FAQ Item 22 menciona percentuais de seguro (0,3% para cargas padrão, 0,8% para
cargas perigosas), mas **não existe documento normativo sobre seguro de carga na base**.

Até que a OQ-02 seja resolvida e um documento normativo seja criado:
- Qualquer resposta sobre seguro de carga usa o estado `COM_RESSALVA`
- O disclaimer padrão se aplica
- Adicionar instrução: *"Os percentuais informados são baseados em FAQ interno e podem
  não refletir o contrato atual do cliente. Confirme com o Comercial antes de informar
  ao cliente."*
- Contratos anteriores a 2023 podem ter percentuais diferentes — sempre mencionar

Quando a OQ-02 for resolvida e um documento normativo for criado, esta regra deve ser
revisada e o guardrail atualizado em `guardrails.md`.

> Derivado de: `NAO-11` · `DUVIDA-04` · `specs/query-endpoint/requirements.md OQ-02`

---

### PG-16 — Lacunas documentais conhecidas com comportamento definido

Os temas abaixo **não têm cobertura normativa na base**. O comportamento para cada
um é fixo — não improvise:

| Tema | Comportamento | Encaminhamento |
|---|---|---|
| Frete padrão (abaixo de 500kg) | `GAP_DOCUMENTAL` | Comercial |
| Carga danificada em trânsito | `COM_RESSALVA` (única fonte: FAQ Item 38) | `{SINISTROS_EMAIL}` com fotos e laudo em até 48h |
| Processo interno da Gestão de Riscos | `GAP_DOCUMENTAL` | Ramal 4500 |
| Classes 7-9 da ANTT | `GAP_DOCUMENTAL` | Gestão de Riscos |
| Frete expresso para carga perigosa | `COM_RESSALVA` (única fonte: FAQ Item 32) | Compliance para autorização |

Para frete padrão e carga danificada, inclua queries em
`prompts/eval/golden-queries.json` com os estados esperados como resposta de
regressão.

> Derivado de: `NAO-02` · `DUVIDA-04` · `DUVIDA-09` · `specs/query-endpoint/requirements.md RN-04`
> Incidente de referência: INC-01

---

### PG-17 — Mudanças no system prompt seguem protocolo de versionamento

O system prompt vive em `prompts/system-prompt.md`. Toda mudança deve ser
registrada em `prompts/prompt-changelog.md` com: data, autor, motivo, e resultado
esperado. Execute `prompts/eval/` antes e depois de qualquer mudança para detectar
regressões.

As regras das seções PG-01 a PG-16 **devem estar refletidas no system prompt**.
Se você identificar uma divergência entre uma regra aqui e o comportamento do prompt,
o AGENTS.md prevalece — corrija o prompt, não o AGENTS.md.

Para mudar uma regra de negócio (ex: novo limiar de incidente crítico, novo tier),
o fluxo é:
1. Atualizar `guardrails.md`
2. Atualizar esta seção do `AGENTS.md`
3. Atualizar `prompts/system-prompt.md`
4. Registrar em `prompts/prompt-changelog.md`
5. Executar `prompts/eval/` e confirmar que golden queries passam

Nunca atualize apenas o system prompt sem atualizar AGENTS.md e guardrails.md.

> Derivado de: `DEVE-15` · `Anexo C — Convenções /prompts/`

---

### Referências

| Artefato | Localização | Versão atual |
|---|---|---|
| Guardrails completos | `guardrails.md` (raiz do repositório) | v1.3.0 |
| Requirements do Query Endpoint | `specs/query-endpoint/requirements.md` | v1.1.0 |
| Documentação de negócio NovaTech | `docs/novatech/` | Anexo A |
| System prompt | `prompts/system-prompt.md` | — |
| Tabela de sinônimos | `prompts/synonym-table.json` | — |
| Tabela de encaminhamentos | `prompts/routing-table.json` | — |
| Golden queries | `prompts/eval/golden-queries.json` | — |
