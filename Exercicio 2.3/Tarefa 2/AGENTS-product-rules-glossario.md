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

## Glossário de Linguagem Ubíqua do Domínio

> Este glossário define os termos que **você deve usar de forma consistente** ao gerar
> código, comentários, nomes de variáveis, mensagens de resposta, casos de teste e
> qualquer artefato deste repositório. Usar o termo errado em código ou em respostas
> não é apenas imprecisão — pode causar falha no retrieval semântico, resposta incorreta
> ao atendente ou inconsistência com os documentos normativos da NovaTech.
>
> Cada entrada tem: definição canônica, fonte normativa, o que **nunca usar** no lugar,
> e uma nota operacional para uso em código e respostas.

---

### Grupo 1 — Tipos de Carga e Modalidades de Frete

---

#### Carga Perigosa
**Definição:** mercadoria classificada nas **classes 1 a 6 da ANTT**, conforme
Resolução ANTT nº 5.947/2021.

| Classe | Tipo |
|---|---|
| 1 | Explosivos |
| 2 | Gases |
| 3 | Líquidos inflamáveis |
| 4 | Sólidos inflamáveis |
| 5 | Oxidantes e peróxidos |
| 6 | Substâncias tóxicas e infectantes |

**Fonte:** POL-001 §3.2
**Nunca usar:** "carga especial", "material perigoso", "carga regulada" (sem referência à classe ANTT)
**Nota operacional:** classes 7 (radioativos), 8 (corrosivos) e 9 (miscelânea) **não estão
cobertas** pela base documental. Respostas sobre essas classes retornam `GAP_DOCUMENTAL`.
Carga Perigosa **não é elegível** ao processo padrão de devolução (POL-001 §3.2) —
encaminhar sempre para Gestão de Riscos, ramal 4500.
Em código, use o literal `'carga_perigosa'` como `topic_tag` ao indexar chunks relacionados.

---

#### Carga Refrigerada
**Definição:** mercadoria que exige controle de temperatura durante o transporte,
monitorada por sensor IoT com registro contínuo. A **cadeia de frio é considerada
rompida** quando a temperatura fica fora da faixa especificada na nota fiscal por
mais de **30 minutos contínuos**.

**Fonte:** POL-001 §3.2
**Nunca usar:** "carga fria", "produto refrigerado"
**Nota operacional:** a faixa de temperatura é definida na nota fiscal, não em nenhum
documento da base. O assistente não tem acesso à NF nem ao sensor IoT — ao responder
sobre ruptura de cadeia de frio, sempre orientar o atendente a verificar o sistema de
rastreamento. Carga Refrigerada com cadeia de frio rompida não é elegível ao processo
padrão de devolução.

---

#### Carga com Lacre Violado
**Definição:** mercadoria entregue com lacre de segurança rompido.
**Exceção:** se a violação for **documentada no ato de entrega** com assinatura do
motorista e do recebedor, a carga pode seguir o processo padrão de devolução.

**Fonte:** POL-001 §3.2
**Nota operacional:** o assistente não tem acesso à documentação física de entrega.
Ao responder sobre lacre violado, sempre perguntar ao atendente se há documentação
assinada antes de orientar o fluxo.

---

#### Frete Especial
**Definição:** modalidade de frete aplicável a cargas com **peso acima de 500kg**.
Calculado pela fórmula:

```
Valor do frete = Valor base × Multiplicador regional × Fator de peso
```

**Fonte:** PROC-042 v2 (vigente para chamados a partir de 01/12/2023)
**Nunca usar:** "frete pesado", "frete diferenciado", "frete para carga grande"
**Nota operacional:** sempre use os parâmetros do PROC-042 **v2** — nunca da v1 (obsoleta).
O Valor base **não está na base documental** (é uma tabela mensal externa) — o assistente
orienta sobre a fórmula e os fatores, mas nunca calcula o valor final. Ver PG-08.

---

#### Frete Padrão
**Definição:** ⚠️ **Termo sem definição formal na base documental.** Inferido como a
modalidade aplicável a cargas abaixo de 500kg.

**Fonte:** ausente — gap documental confirmado
**Nota operacional:** perguntas sobre Frete Padrão retornam obrigatoriamente
`GAP_DOCUMENTAL` com encaminhamento ao Comercial. O assistente **não deve inferir**
regras de frete padrão a partir do PROC-042 (que cobre apenas acima de 500kg).
Em `tests/fixtures/queries.ts`, inclua "frete padrão" como caso de teste de GAP_DOCUMENTAL.

---

#### Multiplicador Regional
**Definição:** fator multiplicador aplicado ao Valor base do frete conforme a região
de destino da carga. Valores vigentes (PROC-042 **v2**):

| Região | Multiplicador vigente (v2) | Multiplicador obsoleto (v1) |
|---|---|---|
| Sul | 1.2 | 1.2 |
| Sudeste | 1.0 | 1.0 |
| Centro-Oeste | **1.4** | 1.3 |
| Nordeste | **1.5** | 1.4 |
| Norte | **1.8** | 1.6 |

**Fonte:** PROC-042 v2 §2.1
**Nota operacional:** os valores da coluna "v1" são **obsoletos** — nunca os use em
respostas ou testes. Em `tests/fixtures/chunks.ts`, ao criar chunks de teste para
PROC-042, use exclusivamente os valores da v2. O `attribute_key` para esses chunks
deve ser `multiplicador_regional_{regiao}` (ex: `multiplicador_regional_norte`).

---

#### Fator de Peso
**Definição:** fator multiplicador aplicado ao frete especial conforme a faixa de peso
da carga. Valores vigentes (PROC-042 **v2**):

| Faixa de peso | Fator vigente (v2) | Fator obsoleto (v1) |
|---|---|---|
| 500kg a 1.000kg | 1.0 | 1.0 |
| 1.001kg a 3.000kg | **1.15** | 1.2 |
| Acima de 3.000kg | **1.4** | 1.5 |

**Fonte:** PROC-042 v2 §2
**Nota operacional:** idem ao Multiplicador Regional — use exclusivamente os valores
da v2. O `attribute_key` para esses chunks deve ser `fator_peso_{faixa}`.

---

#### Valor Base
**Definição:** tarifa publicada na **tabela mensal de fretes**, disponível em
`\\novatech-fs\comercial\tabelas\frete-base-AAAAMM.xlsx`. Atualizada mensalmente.

**Fonte:** PROC-042 v2 §2
**Nota operacional:** este valor **não está indexado** na base do assistente — é
externo e muda todo mês. O assistente nunca retorna um Valor Base específico.
Ao responder sobre cálculo de frete, sempre informar que o atendente precisa
consultar a tabela mensal para obter o Valor Base atual.

---

### Grupo 2 — Processo de Devolução

---

#### Solicitação de Devolução
**Definição:** pedido formal aberto pelo cliente no **Portal do Cliente**
(portal.novatech.com.br), na categoria "Devolução de Mercadoria", contendo
obrigatoriamente: número do CT-e, mínimo de 3 fotos (embalagem externa, etiqueta,
conteúdo) e motivo da devolução.

**Fonte:** POL-001 §3.3
**Nunca usar:** "pedido de devolução", "solicitação de retorno", "chamado de devolução"
**Nota operacional:** abertura no Portal do Cliente é **diferente** de abertura de
chamado no sistema de chamados (Azure DevOps). São canais distintos para propósitos
distintos. O assistente nunca abre a Solicitação de Devolução — orienta o atendente
a fazê-lo.

---

#### Prazo de Devolução
**Definição:** **7 dias úteis** contados a partir da **data de recebimento confirmada
no sistema de tracking**. Exclui sábados, domingos e feriados nacionais.

**Fonte:** POL-001 §3.1
**Nunca usar:** "7 dias corridos", "uma semana", "7 dias a partir da entrega"
**Nota operacional:** a data de referência é o tracking, **não a data informada pelo
cliente**. São datas potencialmente diferentes. Ao responder sobre Prazo de Devolução,
sempre incluir o aviso de que o atendente deve verificar a data no sistema de tracking
antes de comunicar o prazo ao cliente. Ver PG-13.

---

#### CT-e (Conhecimento de Transporte Eletrônico)
**Definição:** documento fiscal eletrônico que comprova a prestação do serviço de
transporte. É o **identificador principal** de uma operação de frete na NovaTech.
Obrigatório em toda Solicitação de Devolução.

**Fonte:** POL-001 §3.3
**Nunca usar:** "nota de transporte", "comprovante de frete", "documento de transporte"
**Nota operacional:** ao orientar o processo de devolução, sempre referenciar o CT-e
pelo nome completo na primeira menção e como "CT-e" nas demais. Em código TypeScript,
use `cteNumber` como nome de campo — nunca `nfe`, `notaFiscal` ou variações.

---

#### Coleta Reversa
**Definição:** **serviço** de retirada da mercadoria devolvida no endereço do cliente,
agendado em até **2 dias úteis** após a aprovação da Solicitação de Devolução.

**Fonte:** POL-001 §3.3
**Nunca usar:** "coleta de devolução", "frete reverso" para se referir ao serviço
**Nota operacional:** Coleta Reversa é o **serviço**. Frete Reverso é o **custo**.
São conceitos distintos — nunca use um pelo outro. Ver PG-14.

---

#### Frete Reverso
**Definição:** **custo** do transporte de retorno da mercadoria devolvida.
Responsabilidade varia conforme o motivo da devolução:

| Motivo | Responsável pelo custo |
|---|---|
| Defeito ou erro da NovaTech (carga errada, avaria em trânsito) | NovaTech — sem custo ao cliente |
| Desistência do cliente (carga correta, sem defeito) | Cliente — mesmos multiplicadores do frete original |
| Prazo de Devolução expirado | Não elegível — encaminhar ao Comercial |

**Fonte:** POL-001 §3.5
**Nunca usar:** "frete de devolução", "coleta reversa" para se referir ao custo
**Nota operacional:** ao responder sobre Frete Reverso por desistência do cliente,
informar que o cálculo usa os mesmos multiplicadores do frete original — mas o
Valor Base atual precisa ser consultado na tabela mensal (o assistente não calcula).

---

#### Devolução Parcial
**Definição:** devolução de **volumes individuais** dentro de uma entrega com múltiplos
volumes. Cada volume devolvido segue o mesmo procedimento da POL-001 §3.3.
O reembolso é proporcional ao peso/valor do volume devolvido, conforme o CT-e.

**Fonte:** POL-001 §3.4
**Nota operacional:** não existe fluxo simplificado para Devoluções Parciais — cada
volume segue o procedimento completo individualmente. O número do CT-e é o mesmo
da entrega original; identificar o volume específico dentro do CT-e é responsabilidade
do atendente.

---

### Grupo 3 — SLA e Classificação de Clientes

---

#### Tier de Cliente
**Definição:** classificação do cliente em **Gold**, **Silver** ou **Standard**, baseada
em volume mensal de operações ou valor do contrato anual. O critério usa operador **OU**:
basta atender a um dos dois para se qualificar no tier.

| Tier | Critério (operador OU) | Revisão |
|---|---|---|
| **Gold** | Contrato anual > R$ 500.000 **OU** > 200 operações/mês | Semestral |
| **Silver** | Contrato entre R$ 100K–500K **OU** 50–200 operações/mês | Semestral |
| **Standard** | Todos os demais | Anual |

**Fonte:** SLA-2024 §1
**Nunca usar:** "categoria de cliente", "nível de cliente", "Platinum", "cliente VIP"
**Nota operacional:** **Platinum não existe** — foi descontinuado em 2022. Perguntas
sobre Platinum retornam `FUNDAMENTADA` informando a descontinuação e listando os três
tiers vigentes (não `GAP_DOCUMENTAL`). O operador OU significa que um cliente com
contrato de R$ 600K mas apenas 30 operações/mês **é Gold** pelo critério de valor.
Em código TypeScript:

```typescript
type ClientTier = 'Gold' | 'Silver' | 'Standard'; // nunca 'Platinum'
```

---

#### Incidente Crítico
**Definição:** chamado classificado como crítico quando atende a **pelo menos um** dos
seguintes critérios (SLA-2024 §3):

1. Carga com valor declarado acima de **R$ 100.000** com status desconhecido há mais
   de **6 horas**
2. Carga Perigosa com qualquer irregularidade de documentação ou rastreamento
3. Mais de **5 chamados** do mesmo cliente nas últimas **24 horas** sobre o mesmo problema
4. Qualquer situação que envolva **risco à segurança de pessoas**

**Fonte:** SLA-2024 §3
**Nunca usar:** "chamado urgente", "chamado P1", "prioridade alta", "prioridade crítica"
**Nota operacional:** o limiar de valor é **R$ 100.000** — não R$ 50.000 (valor mencionado
no FAQ Item 27, que é inconsistente com o documento contratual e deve ser ignorado).
Em código, use `'incidente_critico'` como `topic_tag` e inclua os 4 critérios explicitamente
nos chunks de metadados do SLA-2024.

---

#### SLA de Primeira Resposta
**Definição:** tempo máximo entre a abertura do chamado e o **primeiro retorno ao
cliente** — mesmo que seja apenas "estamos verificando o seu caso". Não implica que o
problema foi resolvido.

| Tier | Chamados gerais | Incidentes críticos |
|---|---|---|
| Gold | Até 2h úteis | Até 30 minutos |
| Silver | Até 4h úteis | Até 1h |
| Standard | Até 8h úteis | Até 2h |

**Fonte:** SLA-2024 §2
**Distinção crítica:** diferente de SLA de Resolução. Primeira Resposta é o contato
inicial; Resolução é o encerramento do problema.
**Nota operacional:** ao responder sobre SLA, sempre especificar se é "Primeira Resposta"
ou "Resolução" — nunca responda com um prazo genérico de "SLA do cliente Gold" sem
qualificar qual SLA.

---

#### SLA de Resolução
**Definição:** tempo máximo entre a abertura do chamado e o **encerramento efetivo do
problema**.

| Tier | Chamados gerais | Incidentes críticos |
|---|---|---|
| Gold | Até 24h úteis | Até 4h |
| Silver | Até 48h úteis | Até 8h |
| Standard | Até 72h úteis | Até 24h |

**Fonte:** SLA-2024 §2
**Nota operacional:** o documento não define o que constitui "resolução" para cada tipo
de chamado — é uma ambiguidade conhecida. Ao responder, use os prazos acima e oriente
que a definição de "resolvido" pode variar por tipo de chamado.

---

#### Relógio de SLA
**Definição:** mecanismo de contagem do tempo de SLA, iniciado a partir do timestamp
de abertura do chamado no Azure DevOps. Possui **dois comportamentos distintos**:

- **Chamados gerais** (qualquer tier): relógio **pausa** fora do horário comercial
  (08h–18h, dias úteis)
- **Incidentes Críticos de clientes Gold**: relógio **não pausa** — corre **24/7**

**Fonte:** SLA-2024 §5
**Nota operacional:** esta distinção é obrigatória em toda resposta sobre prazos de SLA.
Nunca responda "o SLA pausa fora do horário" sem especificar que isso não se aplica a
Incidentes Críticos Gold. Ver PG-09. Em código, use `clockMode: '24x7' | 'business_hours'`
como campo no tipo de chamado.

---

#### Penalidade de SLA
**Definição:** crédito concedido ao cliente por violação do SLA, escalonado por
recorrência **no mesmo mês**:

| Ocorrência no mês | Consequência |
|---|---|
| 1ª violação | Registro interno — sem impacto contratual |
| 2ª violação | Crédito de **5%** sobre o valor do frete do **chamado afetado** |
| 3ª violação ou mais | Crédito de **10%** + reunião obrigatória |

**Fonte:** SLA-2024 §4
**Nota operacional:** a penalidade é calculada sobre o frete do chamado afetado —
**não sobre o contrato total**. O assistente não tem acesso ao valor do frete do
chamado — sempre orientar o atendente a verificar no sistema.

---

### Grupo 4 — Governança Documental

---

#### Documento Normativo
**Definição:** documento que estabelece regras de cumprimento obrigatório.
Identificado pelo prefixo **POL** (política) ou **PROC** (procedimento).
Exemplos na base: POL-001, PROC-042 v2.

**Nota operacional:** respostas baseadas em Documento Normativo retornam estado
`FUNDAMENTADA`. Em código, `doc_type: 'normativo'`. Ao indexar, esse tipo tem
**precedência** sobre documentos informais quando cobrem o mesmo tema.

---

#### Documento Contratual
**Definição:** documento que estabelece compromissos formais da NovaTech com seus
clientes. Identificado pelo prefixo **SLA**. Violações têm consequência financeira direta.
Exemplo na base: SLA-2024.

**Nota operacional:** respostas baseadas em Documento Contratual retornam estado
`FUNDAMENTADA`. Em código, `doc_type: 'contratual'`. Citações de documentos contratuais
exigem especial precisão — um prazo errado pode gerar disputa contratual.

---

#### Documento Informal
**Definição:** documento sem responsável formal, sem validação por Compliance ou
Operações, mantido colaborativamente pelo time. Exemplo na base: FAQ-Atendimento.

**Nota operacional:** respostas baseadas **exclusivamente** em Documento Informal retornam
estado `COM_RESSALVA` com disclaimer obrigatório. Em código, `doc_type: 'informal'`.
**Nunca** equipare um Documento Informal a um normativo em termos de confiabilidade.
Quando normativo e informal cobrem o mesmo tema, use apenas o normativo.

---

#### Versão Vigente
**Definição:** a versão de um documento que deve ser usada como referência atual.
Para o PROC-042: a **v2 é vigente** para chamados abertos a partir de 01/12/2023.
A v1 é tratada como obsoleta nesta implementação.

**Nota operacional:** o PROC-042 v1 tem `status: 'obsoleto'` no índice e nunca é
recuperado. Ao criar chunks em `tests/fixtures/chunks.ts`, marque explicitamente
a versão: `docVersion: 'v2'`. Ao criar golden queries em
`prompts/eval/golden-queries.json`, teste que respostas sobre frete especial
sempre citam a v2.

---

#### Gap Documental
**Definição:** ausência de fonte normativa ou contratual na base para um tema relevante
do atendimento. Gaps conhecidos:

| Tema | Status |
|---|---|
| Frete Padrão (abaixo de 500kg) | Gap — sem documento na base |
| Carga Danificada em Trânsito | Parcial — coberto apenas por FAQ Item 38 (informal) |
| Seguro de Carga | Parcial — coberto apenas por FAQ Item 22 (informal) |
| Frete Expresso para Carga Perigosa | Parcial — coberto apenas por FAQ Item 32 (informal) |
| Classes 7-9 da ANTT | Gap — sem documento na base |
| Processo interno da Gestão de Riscos | Gap — referenciado na POL-001 mas sem PROC documentado |

**Nota operacional:** ao encontrar um Gap Documental durante desenvolvimento ou testes,
registre em `prompts/eval/golden-queries.json` com `expectedState: 'GAP_DOCUMENTAL'`
e em `docs/novatech/gaps.md` com o tema e a fonte parcial disponível. Gaps são
retroalimentados para BC-05 (BC de Auditoria) para priorização de novos documentos.

---

### Termos proibidos — nunca use estes no lugar dos termos canônicos

O uso de termos proibidos em código, comentários, mensagens de resposta ou casos de
teste introduz inconsistência semântica que pode causar falha no retrieval.

| Termo canônico | Nunca usar |
|---|---|
| **Carga Perigosa** | "carga especial", "material perigoso", "carga regulada" |
| **Frete Especial** | "frete pesado", "frete diferenciado", "frete para carga grande" |
| **Frete Padrão** | "frete normal", "frete comum", "frete abaixo de 500kg" |
| **Multiplicador Regional** | "taxa regional", "fator de região", "coeficiente" |
| **Prazo de Devolução** | "7 dias corridos", "uma semana", "prazo de retorno" |
| **CT-e** | "nota de transporte", "comprovante de frete", "NF de transporte" |
| **Coleta Reversa** | "coleta de devolução", "frete reverso" (quando se refere ao serviço) |
| **Frete Reverso** | "coleta reversa" (quando se refere ao custo) |
| **Tier de Cliente** | "categoria de cliente", "nível de cliente", "Platinum" |
| **Incidente Crítico** | "chamado P1", "chamado urgente", "prioridade alta", "prioridade crítica" |
| **SLA de Primeira Resposta** | "SLA de resposta", "tempo de atendimento" |
| **SLA de Resolução** | "SLA de fechamento", "tempo de resolução" (sem "SLA") |
| **Relógio de SLA** | "contador de SLA", "timer de SLA" |
| **Documento Normativo** | "documento oficial", "regra interna" |
| **Documento Informal** | "documento auxiliar", "material de apoio" |
| **Gap Documental** | "informação não disponível", "sem documentação" |

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
