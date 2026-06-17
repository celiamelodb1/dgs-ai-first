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

---

## Code Constraints

> Esta seção define restrições que se aplicam diretamente ao código gerado.
> São derivadas dos schemas de request/response (`specs/query-endpoint/requirements.md §8 AC-01`),
> dos RNFs (`requirements.md §7`), das ADRs e das convenções do repositório (`Anexo C`).
> Cada restrição tem um identificador `CC-NN` para referência em code review e PRs.
>
> Quando gerar código para qualquer arquivo em `src/`, verifique esta seção antes de
> escrever a primeira linha. Violações de CC são bloqueantes em PR — não são sugestões.

---

### Contrato de tipos obrigatório (`src/shared/types.ts`)

**[CC-01]** O tipo `QueryResponse` é a forma canônica de toda resposta do endpoint.
Nenhum handler, service ou função auxiliar pode retornar um objeto de resposta que
não seja atribuível a este tipo. **Nunca** expanda campos inline em handlers —
use sempre o tipo centralizado.

```typescript
// src/shared/types.ts — definição canônica, não altere sem PR aprovado por Tech Lead

export type ResponseState =
  | 'FUNDAMENTADA'
  | 'COM_RESSALVA'
  | 'CONTRADIÇÃO_DETECTADA'
  | 'GAP_DOCUMENTAL';

export type ConfidenceLevel = 'normativo' | 'contratual' | 'informal';

export type Domain =
  | 'frete_logistica'
  | 'devolucao_politica'
  | 'sla_contrato'
  | 'nao_classificado';

export type ClientTier = 'Gold' | 'Silver' | 'Standard'; // Platinum não existe

export interface SourceDocument {
  doc_id:   string;   // ex: 'POL-001', 'PROC-042-v2', 'SLA-2024'
  version:  string;   // ex: '3.1', '2.0', '2024.1'
  section:  string;   // ex: '§3.2', 'seção 2.1'
  chunk_id: string;   // UUID do chunk no índice vetorial
}

export interface QueryResponse {
  state:             ResponseState;
  answer:            string;
  session_id:        string;                  // UUID — sempre presente
  domain_classified: Domain;                  // sempre presente, mesmo em GAP_DOCUMENTAL
  sources:           SourceDocument[];        // vazio apenas em GAP_DOCUMENTAL e CONTRADIÇÃO_DETECTADA
  // Campos condicionais — presente apenas nos estados indicados:
  confidence_level?:  ConfidenceLevel;        // FUNDAMENTADA e COM_RESSALVA
  disclaimer?:        string;                 // COM_RESSALVA
  conflicting_docs?:  string[];               // CONTRADIÇÃO_DETECTADA
  routing_suggestion?: string;               // GAP_DOCUMENTAL
  max_score_obtained?: number;               // GAP_DOCUMENTAL — para rastreabilidade em BC-05
}

export interface QueryRequest {
  query:      string;       // obrigatório, max 1000 chars
  session_id: string;       // UUID — ausente = nova sessão criada pelo handler
  user_id:    string;       // AAD object ID — obrigatório
  tier_hint?: ClientTier;   // opcional — ver OQ-05/OQ-06
}
```

> Derivado de: `requirements.md §8 AC-01` · `ADR-002` · `ADR-003`

---

**[CC-02]** O campo `sources[].version` deve ser populado a partir do metadado
`doc_version` do chunk efetivamente recuperado no retrieval — **nunca** a partir
do documento mais recente disponível na base.

```typescript
// CORRETO — usa a versão do chunk recuperado
sources: chunks.map(c => ({
  doc_id:   c.metadata.doc_id,
  version:  c.metadata.doc_version,   // vem do chunk, não do documento
  section:  c.metadata.section,
  chunk_id: c.id,
}))

// ERRADO — busca a versão mais recente do documento, ignora o chunk
sources: chunks.map(c => ({
  doc_id:   c.metadata.doc_id,
  version:  await getLatestDocVersion(c.metadata.doc_id),  // bug — não faça isso
  ...
}))
```

Se `doc_version` não estiver presente nos metadados do chunk, **lance um erro de
ingestão** — não silencie com um fallback. Chunks sem `doc_version` são inválidos.

> Derivado de: `INC-02` · `DEVE-02` · `guardrails.md §5 L-02`

---

**[CC-03]** Todo chunk indexado em `src/pipeline/indexer.ts` deve carregar os
seguintes metadados obrigatórios. Ingestão de chunk sem qualquer um desses campos
deve lançar `ChunkMetadataError` e interromper o pipeline — nunca indexar silenciosamente.

```typescript
// src/shared/types.ts
export interface ChunkMetadata {
  doc_id:       string;        // ex: 'POL-001'
  doc_version:  string;        // ex: '3.1'
  doc_type:     'normativo' | 'contratual' | 'informal';
  status:       'vigente' | 'obsoleto';
  topic_tag:    string;        // ex: 'devolucao', 'frete_especial', 'sla_gold'
  attribute_key: string;       // ex: 'prazo_devolucao', 'multiplicador_regional_norte'
  section:      string;        // ex: '§3.2'
  source_file:  string;        // nome do arquivo original
}

// src/shared/errors.ts
export class ChunkMetadataError extends Error {
  constructor(public missingFields: string[], public sourceFile: string) {
    super(`Chunk inválido em ${sourceFile}: campos ausentes [${missingFields.join(', ')}]`);
  }
}
```

`topic_tag` e `attribute_key` são os campos que habilitam a detecção determinística
de contradição (ADR-006 / PG-03). Sem eles, `CONTRADIÇÃO_DETECTADA` nunca será
acionado e conflitos entre documentos passarão silenciosamente ao LLM.

> Derivado de: `ADR-006` · `DEVE-04` · `NAO-01` · `guardrails.md §4.4 DEVE-04`

---

### Validação de input com Zod (`src/functions/query/validator.ts`)

**[CC-04]** Todo input do endpoint deve ser validado via Zod **antes** de qualquer
lógica de negócio. O schema Zod é a fonte de verdade para validação de entrada —
não implemente validações manuais paralelas.

```typescript
// src/functions/query/validator.ts
import { z } from 'zod';

export const QueryRequestSchema = z.object({
  query:      z.string().min(1).max(1000),
  session_id: z.string().uuid().optional(),
  user_id:    z.string().min(1),   // AAD object ID — não valide formato, apenas presença
  tier_hint:  z.enum(['Gold', 'Silver', 'Standard']).optional(),
});

export type ValidatedQueryRequest = z.infer<typeof QueryRequestSchema>;
```

Erros de validação retornam **HTTP 400** com o campo `errors` descrevendo os campos
inválidos. Nunca retorne HTTP 500 para erro de validação de input.

```typescript
// src/functions/query/handler.ts — padrão de tratamento de erro de validação
const parsed = QueryRequestSchema.safeParse(body);
if (!parsed.success) {
  return {
    status: 400,
    body: { errors: parsed.error.flatten().fieldErrors },
  };
}
```

> Derivado de: `Anexo C — src/functions/query/validator.ts` · `requirements.md §8 AC-01`

---

### Guards de output (`src/services/response-validator.ts`)

**[CC-05]** O `response-validator.ts` implementa os guards de output que interceptam
respostas do LLM antes de chegarem ao atendente. São verificações determinísticas —
não dependem do LLM para funcionar.

Guard obrigatório **[CC-05a] — valor monetário em resposta de frete:**

```typescript
// src/services/response-validator.ts
const MONETARY_VALUE_PATTERN = /R\$\s*[\d.,]+/i;

export function guardMonetaryValue(answer: string, domain: Domain): void {
  if (domain === 'frete_logistica' && MONETARY_VALUE_PATTERN.test(answer)) {
    throw new OutputGuardError(
      'monetary_value_in_freight',
      'Resposta contém valor monetário calculado de frete — ' +
      'o assistente não tem o Valor Base atual e não pode calcular o frete final.',
    );
  }
}
```

Guard obrigatório **[CC-05b] — claim de status em tempo real:**

```typescript
const REALTIME_STATUS_PATTERNS = [
  /chamado\s+(está|foi|encontra-se)/i,
  /status\s+atual/i,
  /no\s+momento/i,
  /agora\s+mesmo/i,
];

export function guardRealtimeStatus(answer: string): void {
  const matched = REALTIME_STATUS_PATTERNS.find(p => p.test(answer));
  if (matched) {
    throw new OutputGuardError(
      'realtime_status_claim',
      'Resposta contém claim de status em tempo real — ' +
      'o assistente não tem acesso ao Azure DevOps.',
    );
  }
}
```

Quando um guard lança `OutputGuardError`, o handler retorna `GAP_DOCUMENTAL` com
`routing_suggestion` genérico — **nunca** propaga o texto do LLM que acionou o guard.

```typescript
// src/shared/errors.ts
export class OutputGuardError extends Error {
  constructor(public guardId: string, message: string) {
    super(message);
  }
}
```

> Derivado de: `NAO-06` · `NAO-08` · `PG-08` · `guardrails.md §4.4 NAO-06`

---

### HTTP e comportamento do endpoint

**[CC-06]** Mapa canônico de status HTTP. Não use códigos fora desta tabela sem
aprovação em PR — inconsistência de status quebra o bot do Teams.

| Situação | Status HTTP | Body obrigatório |
|---|---|---|
| Resposta gerada com sucesso (qualquer estado) | `200` | `QueryResponse` completo |
| Input inválido (Zod validation fail) | `400` | `{ errors: Record<string, string[]> }` |
| Token AAD ausente ou inválido | `401` | `{ error: 'unauthorized' }` |
| Timeout do LLM (>8s end-to-end) | `504` | `{ error: 'timeout', message: 'O assistente está demorando mais que o esperado. Tente novamente em instantes.' }` |
| Erro interno não tratado | `500` | `{ error: 'internal_error', request_id: string }` — nunca exponha stack traces |

`GAP_DOCUMENTAL` **não é um erro** — retorna HTTP 200 com `state: 'GAP_DOCUMENTAL'`.
`CONTRADIÇÃO_DETECTADA` **não é um erro** — retorna HTTP 200 com `state: 'CONTRADIÇÃO_DETECTADA'`.

> Derivado de: `requirements.md RNF-01` · `ADR-005`

---

**[CC-07]** O campo `request_id` deve estar presente em todo response de erro (status
4xx e 5xx) para rastreabilidade em logs. Nunca retorne um erro sem `request_id`.

```typescript
// src/shared/logger.ts — padrão de geração de request_id
import { randomUUID } from 'crypto';

export function createRequestId(): string {
  return randomUUID();
}

// src/functions/query/handler.ts — injetar no início do handler
const requestId = createRequestId();
context.log.info({ requestId, userId: parsed.data.user_id }, 'query received');
```

> Derivado de: `Anexo C — src/shared/logger.ts (pino)` · convenção de observabilidade

---

### Restrições de performance (`src/functions/query/handler.ts`)

**[CC-08]** O timeout de 8 segundos é implementado com `Promise.race`. Nunca confie
no timeout padrão do Azure Functions — implemente explicitamente no handler.

```typescript
// src/functions/query/handler.ts
const TIMEOUT_MS: Record<ResponseState, number> = {
  'FUNDAMENTADA':           8000,
  'COM_RESSALVA':           8000,
  'CONTRADIÇÃO_DETECTADA':  8000,
  'GAP_DOCUMENTAL':         3000, // não aciona LLM — mais rápido
};

async function withTimeout<T>(promise: Promise<T>, ms: number): Promise<T> {
  const timeout = new Promise<never>((_, reject) =>
    setTimeout(() => reject(new TimeoutError()), ms),
  );
  return Promise.race([promise, timeout]);
}
```

Em timeout, o handler preserva a sessão no Redis (não executa DELETE) e retorna
HTTP 504 com o body fixo definido em CC-06. Nunca retorne o resultado parcial
de uma geração interrompida.

> Derivado de: `requirements.md RNF-01` · `DEVE-12` · `PG-08`

---

**[CC-09]** O TTL da sessão no Redis é renovado com `EXPIRE` **após receber a
mensagem do atendente** — não após enviar a resposta. A inatividade é contada do
lado do atendente (ausência de input), não do sistema (ausência de output).

```typescript
// src/functions/query/handler.ts — renovar TTL no início do processamento
await redis.expire(sessionKey, SESSION_TTL_SECONDS); // imediatamente após receber o request
// ... processar query ...
// NÃO renove o TTL após enviar a resposta
```

> Derivado de: `requirements.md RN-08` · `PG-07`

---

### Configuração de ambiente (`src/shared/config.ts`)

**[CC-10]** Todas as variáveis de ambiente obrigatórias são validadas na inicialização
do serviço. Se qualquer variável estiver ausente, o serviço **não deve iniciar** —
falhar rápido é preferível a runtime silencioso com comportamento indefinido.

```typescript
// src/shared/config.ts
function requireEnv(key: string): string {
  const value = process.env[key];
  if (!value) throw new Error(`Variável de ambiente obrigatória ausente: ${key}`);
  return value;
}

export const config = {
  // Azure AI Search
  searchEndpoint:   requireEnv('AZURE_SEARCH_ENDPOINT'),
  searchApiKey:     requireEnv('AZURE_SEARCH_API_KEY'),
  searchIndexName:  requireEnv('AZURE_SEARCH_INDEX_NAME'),

  // Azure OpenAI
  openaiEndpoint:   requireEnv('AZURE_OPENAI_ENDPOINT'),
  openaiApiKey:     requireEnv('AZURE_OPENAI_API_KEY'),
  openaiDeployment: requireEnv('AZURE_OPENAI_DEPLOYMENT'),

  // Redis (sessão)
  redisUrl:         requireEnv('REDIS_URL'),

  // Configurações de negócio — lidas de env, nunca hardcoded
  sinitrosEmail:    requireEnv('SINISTROS_EMAIL'),

  // Configurações com default seguro
  sessionTtlSeconds: parseInt(process.env['SESSION_TTL_SECONDS'] ?? '3600', 10),
  maxQueryChars:     parseInt(process.env['MAX_QUERY_CHARS'] ?? '1000', 10),
} as const;
```

Valores de negócio que podem mudar sem redeploy (limiares de threshold, e-mails,
textos de encaminhamento) são lidos de variáveis de ambiente ou arquivos de
configuração em `prompts/` — **nunca** em constantes no código TypeScript.

> Derivado de: `DEVE-15` · `PG-10` · `Anexo C — src/shared/config.ts`

---

### Convenções TypeScript obrigatórias

**[CC-11]** O projeto usa `"strict": true` no `tsconfig.json`. Isso não é negociável.
As implicações práticas para este projeto:

- **`noImplicitAny`:** todo parâmetro de função tem tipo explícito. Nunca use `any`
  — use `unknown` e narrowing explícito quando o tipo não é conhecido em compile-time.
- **`strictNullChecks`:** acesse campos opcionais do `QueryResponse` apenas após
  verificar presença. Nunca use `!` (non-null assertion) sem comentário justificando.
- **`noUncheckedIndexedAccess`:** ao acessar `sources[0]`, verifique se o array não
  está vazio antes. Respostas `GAP_DOCUMENTAL` têm `sources: []`.

```typescript
// ERRADO
const firstSource = response.sources[0].doc_id; // pode ser undefined

// CORRETO
const firstSource = response.sources.at(0);
if (firstSource) {
  console.log(firstSource.doc_id);
}
```

> Derivado de: `Anexo C — tsconfig.json (strict: true)`

---

**[CC-12]** Erros de domínio são classes tipadas em `src/shared/errors.ts`.
Nunca lance `new Error('string genérica')` em código de produção —
o tipo do erro carrega contexto estruturado para o logger.

```typescript
// src/shared/errors.ts — catálogo de erros do projeto
export class ChunkMetadataError extends Error {
  readonly kind = 'ChunkMetadataError' as const;
  constructor(public missingFields: string[], public sourceFile: string) {
    super(`Chunk inválido em ${sourceFile}: campos ausentes [${missingFields.join(', ')}]`);
  }
}

export class OutputGuardError extends Error {
  readonly kind = 'OutputGuardError' as const;
  constructor(public guardId: string, message: string) { super(message); }
}

export class TimeoutError extends Error {
  readonly kind = 'TimeoutError' as const;
  constructor() { super('LLM response timeout exceeded'); }
}

export class SessionNotFoundError extends Error {
  readonly kind = 'SessionNotFoundError' as const;
  constructor(public sessionKey: string) {
    super(`Sessão não encontrada ou expirada: ${sessionKey}`);
  }
}

export class ConflictDetectedError extends Error {
  readonly kind = 'ConflictDetectedError' as const;
  constructor(public conflictingDocs: string[], public attributeKey: string) {
    super(`Contradição detectada em ${attributeKey}: docs [${conflictingDocs.join(', ')}]`);
  }
}
```

> Derivado de: `Anexo C — src/shared/errors.ts`

---

### Convenções de testes (`tests/`)

**[CC-13]** Todo teste de unidade que envolva chunks deve usar os fixtures de
`tests/fixtures/chunks.ts` — nunca criar objetos de chunk inline em arquivos de teste.
Fixtures são a fonte canônica de dados de teste; objetos inline divergem silenciosamente.

```typescript
// tests/fixtures/chunks.ts — exemplos de fixtures obrigatórios
export const CHUNK_POL001_PRAZO_GERAL: ChunkMetadata = {
  doc_id:        'POL-001',
  doc_version:   '3.1',
  doc_type:      'normativo',
  status:        'vigente',
  topic_tag:     'devolucao',
  attribute_key: 'prazo_devolucao',
  section:       '§3.1',
  source_file:   'POL-001-politica-devolucao.md',
};

export const CHUNK_POL001_EXCECAO_CARGA_PERIGOSA: ChunkMetadata = {
  doc_id:        'POL-001',
  doc_version:   '3.1',
  doc_type:      'normativo',
  status:        'vigente',
  topic_tag:     'devolucao',
  attribute_key: 'elegibilidade_carga_perigosa',  // topic_tag igual, attribute_key diferente de prazo
  section:       '§3.2',
  source_file:   'POL-001-politica-devolucao.md',
};

export const CHUNK_PROC042_V1_OBSOLETO: ChunkMetadata = {
  doc_id:        'PROC-042-v1',
  doc_version:   '1.0',
  doc_type:      'normativo',
  status:        'obsoleto',   // nunca deve ser recuperado pelo retrieval
  topic_tag:     'frete_especial',
  attribute_key: 'multiplicador_regional_norte',
  section:       '§2.1',
  source_file:   'PROC-042-frete-especial-v1.md',
};
```

**[CC-14]** Os três incidentes simulados (INC-01, INC-02, INC-03) são casos de teste
de regressão obrigatórios. Cada um deve ter um teste de integração correspondente
em `tests/integration/` que falha se o incidente puder se reproduzir.

| Incidente | Arquivo de teste sugerido | Assert principal |
|---|---|---|
| INC-01 — "prazo 7 dias para carga perigosa" | `tests/integration/inc-01-carga-perigosa-devolucao.test.ts` | `expect(response.state).not.toBe('FUNDAMENTADA')` quando query sobre carga perigosa + devolução |
| INC-02 — "multiplicadores da v1 com citação de v2" | `tests/integration/inc-02-proc042-versao.test.ts` | `expect(source.version).toBe('2.0')` para toda resposta sobre frete especial |
| INC-03 — "falso GAP_DOCUMENTAL para SLA Gold" | `tests/integration/inc-03-sla-gold-gap.test.ts` | `expect(response.state).not.toBe('GAP_DOCUMENTAL')` quando SLA-2024 está indexado e query é sobre Gold |

> Derivado de: `guardrails.md §5` · `DEVE-09` · `DEVE-07`

---

**[CC-15]** Toda golden query em `prompts/eval/golden-queries.json` deve ter os
campos `expectedState` e `expectedSourceDocId` — não apenas o texto da resposta
esperada. O harness de avaliação valida estrutura antes de validar conteúdo.

```json
// prompts/eval/golden-queries.json — formato obrigatório por entrada
{
  "id": "gq-001",
  "query": "Qual o prazo de devolução para carga perigosa?",
  "tier_hint": null,
  "expectedState": "GAP_DOCUMENTAL",
  "expectedSourceDocId": null,
  "expectedRoutingKeyword": "Gestão de Riscos",
  "tags": ["carga_perigosa", "devolucao", "INC-01"]
}
```

Campos obrigatórios por entrada: `id`, `query`, `expectedState`.
Campos condicionais: `expectedSourceDocId` (obrigatório se `expectedState` for
`FUNDAMENTADA` ou `COM_RESSALVA`), `expectedRoutingKeyword` (obrigatório se
`expectedState` for `GAP_DOCUMENTAL`).

> Derivado de: `Anexo C — prompts/eval/golden-queries.json` · `PG-16` · `PG-12`

---

### Resumo das restrições por arquivo

| Arquivo | CCs que se aplicam |
|---|---|
| `src/shared/types.ts` | CC-01, CC-03 |
| `src/shared/config.ts` | CC-10 |
| `src/shared/errors.ts` | CC-12 |
| `src/shared/logger.ts` | CC-07 |
| `src/functions/query/handler.ts` | CC-06, CC-07, CC-08, CC-09 |
| `src/functions/query/validator.ts` | CC-04 |
| `src/functions/query/response-builder.ts` | CC-01, CC-02 |
| `src/services/search.ts` | CC-02, CC-03 (leitura de metadados) |
| `src/services/response-validator.ts` | CC-05a, CC-05b |
| `src/pipeline/indexer.ts` | CC-03 |
| `tests/fixtures/chunks.ts` | CC-13 |
| `tests/integration/` | CC-14 |
| `prompts/eval/golden-queries.json` | CC-15 |

---

## Spec References

> Esta seção mapeia os documentos de especificação do repositório para que você saiba
> **o que ler antes de tocar em cada parte do sistema**. O repositório segue a
> estrutura SDD — cada módulo tem `requirements.md`, `plan.md` e `tasks.md` em
> `specs/{modulo}/`. Leia o documento correspondente à tarefa **antes** de gerar
> código, testes ou configuração. Não adivinhe intenção — leia a spec.

---

### SR-01 — Mapa de specs por módulo

O projeto tem cinco módulos, cada um com sua própria pasta em `specs/`.
A tabela abaixo indica qual spec ler para cada tipo de tarefa.

| Módulo | Pasta | O que cobre | Ler quando for... |
|---|---|---|---|
| **Query Endpoint** | `specs/query-endpoint/` | Endpoint HTTP de consulta RAG — states, schemas, RNs, ACs, RNFs | Tocar em `src/functions/query/`, `src/services/search.ts`, `src/services/completion.ts`, `src/services/response-validator.ts` |
| **Pipeline de Ingestão** | `specs/pipeline-ingestao/` | Extração, chunking, embedding e indexação de documentos | Tocar em `src/pipeline/` (extractor, chunker, embedder, indexer) |
| **Feedback API** | `specs/feedback-api/` | Endpoint de coleta de feedback do atendente (útil / não útil) | Tocar em `src/functions/feedback/` |
| **Teams Bot** | `specs/teams-bot/` | Lógica do bot no Microsoft Teams — Adaptive Cards, session handling, autenticação AAD | Tocar em `src/bot/`, `src/bot/cards/` |
| **Painel Web** | `specs/painel-web/` | Dashboard de auditoria e melhoria (BC-05) — métricas, gaps recorrentes, feedback | Tocar em `src/web/` |

---

### SR-02 — Três artefatos SDD por módulo: quando usar cada um

Cada pasta de módulo tem exatamente três arquivos. Leia na ordem certa.

```
specs/{modulo}/
├── requirements.md   ← O QUÊ o sistema deve fazer (Product Specialist)
├── plan.md           ← COMO o sistema vai fazer (Tech Lead)
└── tasks.md          ← QUAIS são as tarefas de implementação (Dev + IA)
```

**`requirements.md`** — leia antes de qualquer coisa. Define o comportamento
esperado, regras de negócio, critérios de aceite e restrições de escopo.
Se o requirements não cobre um cenário, **não implemente por intuição** —
abra uma OQ (Open Question) no próprio arquivo ou consulte o Tech Lead.

**`plan.md`** — leia antes de implementar. Define a arquitetura de solução,
decisões técnicas (ADRs), interfaces entre componentes e sequência de
implementação. Se o plan conflitar com o requirements, o requirements prevalece
— registre o conflito antes de continuar.

**`tasks.md`** — use durante a implementação. Lista as tarefas discretas com
critérios de done, dependências e estimativas. Quando uma tarefa estiver
concluída, atualize o status no arquivo — o `tasks.md` é o estado vivo do sprint.

> Derivado de: `Anexo C — Convenções de organização /specs/`

---

### SR-03 — Specs do Query Endpoint (módulo principal desta fase)

O Query Endpoint é o módulo central desta fase. Estes são os documentos de spec
mais relevantes para o trabalho atual.

#### `specs/query-endpoint/requirements.md` (v1.1.0)
**Quem escreveu:** Product Specialist · **Aprovado por:** Tech Lead
**Leia antes de tocar em:** qualquer arquivo em `src/functions/query/` ou `src/services/`

Seções críticas que você precisa conhecer:

| Seção | O que define | Por que importa |
|---|---|---|
| §2 Prior Decisions (ADR-001–006) | 6 decisões arquiteturais fixas | Antes de propor qualquer mudança de arquitetura, verifique se já há uma ADR cobrindo o tema |
| §3 Scope Boundaries | O que está dentro e fora do endpoint | Se uma tarefa pede algo que está na tabela §3.2 (fora do escopo), recuse e documente |
| §5 Estados de Resposta | Os 4 estados válidos e seus triggers | Toda lógica de `response-builder.ts` deriva desta seção |
| §6 Regras de Negócio (RN-01–10) | 10 regras com rastreabilidade a documentos | Cada RN tem um test case obrigatório correspondente |
| §7 RNFs | Latência p95 por estado, concorrência (45 req), disponibilidade (99%) | Referência para testes de performance e configuração de timeouts |
| §8 ACs (AC-01–10) | 10 critérios de aceite verificáveis | A implementação está pronta quando todos os ACs passam — não antes |
| §9 Edge Cases (E1–E8) | 8 cenários de risco com comportamento esperado | Cubra obrigatoriamente nos testes de integração |
| §12 Open Questions (OQ-01–06) | 6 questões pendentes que afetam ACs | Verifique se alguma OQ bloqueia a tarefa que você está implementando |

#### `specs/query-endpoint/plan.md`
**Quem escreve:** Tech Lead · **Status:** a ser criado
**Leia antes de tocar em:** sequenciamento de implementação, interfaces entre serviços

Quando criado, este arquivo deve conter: diagrama de sequência do pipeline de query
(pré-processamento → retrieval → detecção de contradição → geração → validação de output),
decisões de interface entre `handler.ts`, `search.ts`, `completion.ts` e
`response-validator.ts`, e ordem recomendada de implementação por AC.

#### `specs/query-endpoint/tasks.md`
**Quem gera:** Dev com apoio de IA · **Status:** a ser gerado
**Use durante:** implementação das tarefas do módulo

Quando gerado, cada task deve referenciar o AC que ela satisfaz (ex: `satisfaz: AC-01`)
e o arquivo principal que ela modifica (ex: `toca em: src/functions/query/handler.ts`).

---

### SR-04 — Specs do Pipeline de Ingestão

O Pipeline de Ingestão é pré-requisito bloqueante para o Query Endpoint: sem
documentos indexados com os metadados corretos (CC-03), os guardrails PG-02, PG-03
e PG-04 não funcionam.

#### `specs/pipeline-ingestao/requirements.md`
**Status:** a ser escrito · **Responsável:** Product Specialist
**Leia antes de tocar em:** `src/pipeline/` (extractor, chunker, embedder, indexer)

Quando escrito, este requirements deve cobrir obrigatoriamente:

- Regras de atribuição de `topic_tag` e `attribute_key` por tipo de documento
  (os campos que habilitam detecção de contradição — CC-03 / ADR-006)
- Critério de marcação de `status: 'obsoleto'` para PROC-042 v1
- Regras de chunking por tipo de documento (POL vs PROC vs SLA vs FAQ têm
  estruturas diferentes que afetam a granularidade dos chunks)
- Comportamento quando um documento novo substitui um existente
  (re-ingestão, invalidação de chunks antigos, preservação de `doc_version`)
- Campos de metadado obrigatórios por chunk (derivados de CC-03)

**Dependência crítica documentada:** a lacuna L-01 identificada em `guardrails.md §5`
— ausência de mecanismo que force busca de chunks de exceção quando chunk de regra
geral é recuperado — **deve ser requisito do pipeline de ingestão**, não do query
endpoint. Inclua no requirements do pipeline o campo `scope: 'regra_geral' | 'excecao'`
como metadado de chunk.

#### `specs/pipeline-ingestao/plan.md` e `tasks.md`
**Status:** a ser criado e gerado respectivamente

---

### SR-05 — Specs dos módulos de suporte

#### `specs/feedback-api/requirements.md`
**Status:** a ser escrito · **Responsável:** Product Specialist
**Relação com outros módulos:** a Feedback API alimenta BC-05 (Auditoria e Melhoria).
Os logs de `GAP_DOCUMENTAL` com `max_score_obtained` (CC-01 / DEVE-11) são consumidos
por este módulo para calibração de limiares.

Quando escrito, deve cobrir:
- Schema do feedback (útil / não útil, `query_id`, `session_id`, `user_id`)
- Regras de anonimização — PII do atendente não pode ser persistido (CC-10 / NAO-07)
- Conexão com `prompts/eval/eval-results/` para rastreabilidade entre feedback
  produtivo e resultados de avaliação offline

#### `specs/teams-bot/requirements.md`
**Status:** a ser escrito · **Responsável:** Product Specialist
**Relação com outros módulos:** o bot é a camada de transporte entre o atendente e o
Query Endpoint. Decisões de UX do bot (Adaptive Cards, mensagens de estado, UI de
feedback) afetam diretamente como os 4 estados de resposta são apresentados.

Quando escrito, deve cobrir:
- Design dos Adaptive Cards para cada um dos 4 estados (`response-card.ts`,
  `feedback-card.ts` em `src/bot/cards/`)
- Comportamento visual diferenciado para `COM_RESSALVA` (alerta amarelo) e
  `CONTRADIÇÃO_DETECTADA` (alerta vermelho) — derivado de `DEVE-03` e `DEVE-04`
- Fluxo de autenticação AAD para extração do `user_id` (CC-07 / PG-07)
- Formato do campo `tier_hint` no request: preenchimento automático pelo bot
  via integração CRM ou input manual do atendente (OQ-05 / OQ-06 pendentes)

#### `specs/painel-web/requirements.md`
**Status:** a ser escrito · **Responsável:** Product Specialist
**Relação com outros módulos:** painel de operação do BC-05. Consome dados da
Feedback API e logs do Query Endpoint.

Quando escrito, deve cobrir:
- Dashboard de gaps recorrentes por domínio (alimentado por `domain_classified`
  e `max_score_obtained` do QueryResponse — CC-01)
- Alerta de degradação de retrieval: quando taxa de `GAP_DOCUMENTAL` para um
  domínio ultrapassar 15% em 7 dias (lacuna L-03 de `guardrails.md §5`)
- Visualização de contradições detectadas por par de documentos (campo
  `conflicting_docs` — CC-01)

---

### SR-06 — ADRs: decisões arquiteturais já tomadas

As ADRs abaixo estão documentadas no `specs/query-endpoint/requirements.md §2`
e devem ser criadas como arquivos individuais em `docs/adr/` seguindo a nomenclatura
`NNNN-titulo-da-decisao.md`.

| Arquivo sugerido | Decisão | Status |
|---|---|---|
| `docs/adr/0001-arquitetura-rag-base-estatica.md` | RAG sobre base vetorial pré-ingerida — sem busca em tempo real | ✅ Decidida (ADR-001) |
| `docs/adr/0002-classificacao-confiabilidade-documentos.md` | Documentos classificados em normativo / contratual / informal | ✅ Decidida (ADR-002) |
| `docs/adr/0003-controle-versao-metadado-obrigatorio.md` | `doc_version` obrigatório em todo chunk; PROC-042 v1 obsoleto | ✅ Decidida (ADR-003) |
| `docs/adr/0004-contradicao-deterministica-nao-llm.md` | Detecção de contradição por `topic_tag` + `attribute_key`, não por LLM | ✅ Decidida (ADR-004 / ADR-006) |
| `docs/adr/0005-integracao-microsoft-teams.md` | Bot Teams como canal primário; timeout 8s | ✅ Decidida (ADR-005) |

Para criar uma nova ADR, use o template em `docs/adr/template.md` e siga a
nomenclatura sequencial. Toda decisão arquitetural que não caiba em uma OQ de
requirements deve virar uma ADR antes de ser implementada.

> Derivado de: `Anexo C — docs/adr/` · `requirements.md §2`

---

### SR-07 — Skills: leia antes de gerar código para um componente novo

As skills em `skills/` codificam as melhores práticas do projeto. Não são
opcionais — são o padrão de qualidade esperado pelo Tech Lead em code review.

#### Hierarquia de skills

```
skills/
├── foundation/          ← Leia sempre, para qualquer tarefa
│   ├── typescript-conventions.md    ← Padrões TS além do strict — nomes, exports, barrel files
│   ├── error-handling.md            ← Como usar src/shared/errors.ts, padrões de try/catch
│   └── project-structure.md         ← Onde cada tipo de arquivo deve viver
│
├── domain/              ← Leia quando for implementar o componente indicado
│   ├── azure-functions-endpoint.md  ← Antes de tocar em src/functions/
│   ├── azure-ai-search-integration.md ← Antes de tocar em src/services/search.ts ou src/pipeline/
│   ├── react-components.md          ← Antes de tocar em src/web/
│   └── testing-patterns.md          ← Antes de escrever qualquer teste em tests/
│
└── artifact/            ← Leia quando for criar um artefato do tipo indicado do zero
    ├── create-rag-endpoint.md        ← Antes de criar um novo endpoint RAG
    ├── create-integration-test.md    ← Antes de criar um teste em tests/integration/
    └── create-react-card.md          ← Antes de criar um Adaptive Card em src/bot/cards/
```

#### Regra de leitura obrigatória por tarefa

| Tarefa | Skills obrigatórias |
|---|---|
| Qualquer tarefa | `foundation/typescript-conventions.md` + `foundation/error-handling.md` |
| Novo endpoint ou handler | + `domain/azure-functions-endpoint.md` |
| Retrieval ou ingestão no Azure AI Search | + `domain/azure-ai-search-integration.md` |
| Teste de unidade ou integração | + `domain/testing-patterns.md` |
| Novo endpoint RAG do zero | + `artifact/create-rag-endpoint.md` |
| Novo teste de integração | + `artifact/create-integration-test.md` |
| Novo Adaptive Card | + `domain/react-components.md` + `artifact/create-react-card.md` |
| Componente React no painel web | + `domain/react-components.md` |

> Derivado de: `Anexo C — Convenções de organização /skills/`

---

### SR-08 — Prompts: documentos versionados em `prompts/`

Os arquivos em `prompts/` são configuração de produto versionada — não são código,
mas afetam diretamente o comportamento do assistente em runtime.

| Arquivo | O que contém | Quem mantém | Quando atualizar |
|---|---|---|---|
| `prompts/system-prompt.md` | Instruções do assistente derivadas das PGs e do glossário | Product Specialist + Tech Lead | Toda mudança de regra de negócio (seguir PG-17) |
| `prompts/prompt-changelog.md` | Histórico de mudanças no prompt com data, autor, motivo e resultado esperado | Quem fez a mudança | A cada commit que altera `system-prompt.md` |
| `prompts/synonym-table.json` | Tabela de sinônimos não-canônicos → termos canônicos (PG-06 / CC-10) | Product Specialist | Quando novos sinônimos são identificados em produção via BC-05 |
| `prompts/routing-table.json` | Encaminhamentos por domínio para `GAP_DOCUMENTAL` (PG-10 / CC-10) | Product Specialist | Quando mudar área responsável ou contato de encaminhamento |
| `prompts/eval/golden-queries.json` | Perguntas de referência com `expectedState` e `expectedSourceDocId` (CC-15) | Product Specialist + Dev | Toda nova regra de negócio ou edge case identificado em produção |
| `prompts/eval/eval-results/` | Resultados das rodadas de avaliação offline | Automático (CI) | Gerado automaticamente pelo harness de avaliação |

**Protocolo de mudança em `system-prompt.md`** (ver PG-17):
1. Atualizar `guardrails.md`
2. Atualizar a seção relevante deste AGENTS.md
3. Atualizar `prompts/system-prompt.md`
4. Registrar em `prompts/prompt-changelog.md`
5. Executar harness em `prompts/eval/` e confirmar que golden queries passam
6. Abrir PR com todos os arquivos alterados em um único commit

Nunca atualize apenas `system-prompt.md` sem os passos anteriores.

> Derivado de: `Anexo C — Convenções de organização /prompts/` · `PG-17`

---

### SR-09 — Documentação de negócio NovaTech em `docs/novatech/`

Os documentos de negócio da NovaTech vivem em `docs/novatech/` e são a fonte de
verdade do domínio. São os mesmos documentos ingeridos pelo pipeline no índice
vetorial. Se você encontrar divergência entre o código e um documento aqui, o
documento prevalece — registre como bug.

| Arquivo | Documento | Tipo | Versão |
|---|---|---|---|
| `docs/novatech/POL-001-politica-devolucao.md` | Política de Devolução de Mercadorias | Normativo | 3.1 |
| `docs/novatech/PROC-042-frete-especial-v1.md` | Procedimento de Frete Especial (obsoleto) | Normativo | 1.0 — `status: obsoleto` |
| `docs/novatech/PROC-042-v2-frete-especial-revisado.md` | Procedimento de Frete Especial Revisado (vigente) | Normativo | 2.0 |
| `docs/novatech/SLA-2024-tabela-sla-clientes.md` | Tabela de SLA por Tipo de Cliente | Contratual | 2024.1 |
| `docs/novatech/FAQ-atendimento.md` | Perguntas Frequentes do Time de Suporte | Informal | Não controlada |

**Regra de consulta:** antes de responder uma dúvida de domínio ou implementar uma
regra de negócio, leia o documento fonte aqui. Não confie apenas no glossário do
AGENTS.md — o glossário resume, o documento é a fonte primária.

**Documentos com gaps conhecidos** (ver glossário §Grupo 4 — Gap Documental):
frete padrão, carga danificada, seguro de carga e frete expresso para carga perigosa
não têm documentos normativos. Se uma tarefa exigir esses temas, consulte o Tech Lead
antes de implementar qualquer lógica de negócio.

> Derivado de: `Anexo C — filesystem MCP aponta para ./docs/novatech/`

---

### SR-10 — Índice de navegação por tarefa

Use esta tabela como ponto de entrada quando receber uma tarefa nova.
Encontre a linha correspondente e leia os documentos indicados antes de escrever
qualquer código.

| Estou trabalhando em... | Leia antes de começar |
|---|---|
| `src/functions/query/handler.ts` | `specs/query-endpoint/requirements.md` → §5, §7, §8 · `skills/domain/azure-functions-endpoint.md` · CC-06, CC-07, CC-08 |
| `src/functions/query/validator.ts` | `specs/query-endpoint/requirements.md` → §8 AC-01 · CC-04 |
| `src/functions/query/response-builder.ts` | `specs/query-endpoint/requirements.md` → §5 · CC-01, CC-02 |
| `src/functions/feedback/` | `specs/feedback-api/requirements.md` · `skills/domain/azure-functions-endpoint.md` |
| `src/services/search.ts` | `specs/query-endpoint/requirements.md` → §6 RN-01, RN-09 · `skills/domain/azure-ai-search-integration.md` · CC-02, CC-03, PG-02 |
| `src/services/completion.ts` | `specs/query-endpoint/requirements.md` → §2 ADR-001 · `prompts/system-prompt.md` |
| `src/services/response-validator.ts` | `specs/query-endpoint/requirements.md` → §6 RN-03 · CC-05, PG-08 · `guardrails.md §2 NAO-06, NAO-08` |
| `src/pipeline/extractor.ts` | `specs/pipeline-ingestao/requirements.md` · `skills/domain/azure-ai-search-integration.md` |
| `src/pipeline/chunker.ts` | `specs/pipeline-ingestao/requirements.md` · CC-03 · `guardrails.md §5 L-01` |
| `src/pipeline/indexer.ts` | `specs/pipeline-ingestao/requirements.md` · CC-03 · PG-02 |
| `src/bot/` | `specs/teams-bot/requirements.md` · `skills/domain/azure-functions-endpoint.md` · PG-07 |
| `src/bot/cards/` | `specs/teams-bot/requirements.md` · `skills/artifact/create-react-card.md` |
| `src/web/` | `specs/painel-web/requirements.md` · `skills/domain/react-components.md` |
| `src/shared/types.ts` | CC-01, CC-03, CC-12 · `specs/query-endpoint/requirements.md` → §8 AC-01 |
| `src/shared/config.ts` | CC-10 · `specs/query-endpoint/requirements.md` → §6 RN-09, RN-10 |
| `src/shared/errors.ts` | CC-12 · `skills/foundation/error-handling.md` |
| `tests/unit/` | `skills/domain/testing-patterns.md` · spec do módulo correspondente |
| `tests/integration/` | `skills/domain/testing-patterns.md` · `skills/artifact/create-integration-test.md` · CC-14 |
| `tests/fixtures/` | CC-13 · `specs/query-endpoint/requirements.md` → §9 Edge Cases |
| `prompts/system-prompt.md` | PG-17 · todo o AGENTS.md · `guardrails.md` v1.3.0 |
| `prompts/synonym-table.json` | PG-06 · glossário §Termos proibidos |
| `prompts/routing-table.json` | PG-10 · `specs/query-endpoint/requirements.md` → §6 RN-10 |
| `prompts/eval/golden-queries.json` | CC-15 · `specs/query-endpoint/requirements.md` → §8 ACs · `guardrails.md §5` |
| `docs/adr/` | `docs/adr/template.md` · SR-06 |
| `infra/` | Bicep é **estado narrativo** nesta fase — não provisionar recursos reais |

---

### SR-11 — MCP: configuração dos servers do projeto (`.mcp/mcp.json`)

O arquivo `.mcp/mcp.json` configura os MCP servers disponíveis para o agente neste
repositório. Ele **ainda não existe** — deve ser criado nesta fase.

**Servers esperados e seus escopos:**

| Server | Comando | Aponta para | Usa para |
|---|---|---|---|
| `filesystem` | `npx @modelcontextprotocol/server-filesystem` | `./src ./specs ./skills ./docs ./data` | Ler e editar código, specs, skills e documentação |
| `git` | `uvx mcp-server-git --repository .` | `.` (repo local) | Histórico, diffs, branches — substitui GitHub nesta fase |
| `memory` | `npx @modelcontextprotocol/server-memory` | grafo local | Persistir glossário, decisões e contexto entre sessões |
| `everything` | `npx @modelcontextprotocol/server-everything` | — | Explorar primitivas MCP (tools, resources, prompts) |

**Regras ao criar ou editar `.mcp/mcp.json`:**

- Antes de configurar qualquer server, leia o README oficial do repositório
  `modelcontextprotocol/servers` — nomes de pacote e comandos evoluem.
- O server `filesystem` deve incluir `./data` no path para que o agente acesse
  `data/retrieval-corpus/` (corpus de chunks simulados — ver SR-14).
- Não adicione servers que exijam tokens ou contas externas nesta fase — o projeto
  opera localmente sem Azure, GitHub nem Confluence. O server de GitHub foi arquivado
  no upstream; use `filesystem` + `git` para cobrir as mesmas necessidades.
- Toda mudança em `.mcp/mcp.json` deve ser registrada no PR com justificativa.

**Nota sobre o server `memory`:** use para persistir a linguagem ubíqua do glossário
(seção anterior) e decisões arquiteturais entre sessões. Não use para armazenar PII
ou dados de chamados de clientes.

> Derivado de: `Anexo C — .mcp/mcp.json` · `Anexo C — Exemplo de configuração MCP`

---

### SR-12 — Corpus de retrieval simulado (`data/retrieval-corpus/`)

Durante o desenvolvimento, antes do pipeline de ingestão real estar operacional,
o diretório `data/retrieval-corpus/` contém chunks pré-processados que simulam
o índice vetorial. É a fonte que o MCP server `filesystem` usa para simular
o Azure AI Search nas sessões de desenvolvimento local.

**Estrutura esperada:**

```
data/retrieval-corpus/
├── POL-001/                  # Chunks da Política de Devolução
│   ├── chunk-001.json        # Prazo geral §3.1
│   ├── chunk-002.json        # Exceções §3.2 (carga perigosa, refrigerada, lacre)
│   └── chunk-003.json        # Procedimento §3.3
├── PROC-042-v2/              # Chunks do Frete Especial (versão vigente)
│   ├── chunk-001.json        # Fórmula e multiplicadores §2
│   └── chunk-002.json        # Condições especiais §4
├── PROC-042-v1/              # Chunks obsoletos — status: 'obsoleto'
│   └── chunk-001.json        # Marcado com status: 'obsoleto' — nunca deve ser retornado
├── SLA-2024/                 # Chunks do SLA por Cliente
│   ├── chunk-001.json        # Classificação de tiers §1
│   ├── chunk-002.json        # Tabela de SLAs §2
│   └── chunk-003.json        # Incidente crítico §3
└── FAQ-atendimento/          # Chunks do FAQ — doc_type: 'informal'
    ├── chunk-001.json        # Item 3 — carga perigosa
    ├── chunk-002.json        # Item 8 — frete especial
    └── chunk-003.json        # Item 27 — tracking + limiar R$50K (superseded_by: SLA-2024-§3)
```

**Formato de cada chunk JSON** (conforme CC-03):

```json
{
  "id": "uuid-do-chunk",
  "content": "texto do chunk...",
  "metadata": {
    "doc_id":        "POL-001",
    "doc_version":   "3.1",
    "doc_type":      "normativo",
    "status":        "vigente",
    "topic_tag":     "devolucao",
    "attribute_key": "prazo_devolucao",
    "section":       "§3.1",
    "source_file":   "POL-001-politica-devolucao.md"
  }
}
```

**Regras para este diretório:**
- O chunk do PROC-042 v1 deve ter `"status": "obsoleto"` — nunca `"vigente"`.
  Esse metadado é o que habilita o filtro do PG-02 e CC-02.
- O chunk do FAQ Item 27 deve ter `"superseded_by": "SLA-2024-§3"` nos metadados
  para sinalizar que o limiar de R$50K foi substituído pelo limiar oficial de R$100K.
- Os chunks de `data/retrieval-corpus/` devem ser a fonte dos fixtures de teste em
  `tests/fixtures/chunks.ts` — mantenha os dois sincronizados.

> Derivado de: `Anexo C — filesystem MCP aponta para ./data` · `CC-03` · `PG-02`

---

### SR-13 — Infraestrutura como código (`infra/`)

Os arquivos Bicep em `infra/` são **estado narrativo** nesta fase — descrevem a
infraestrutura Azure que será provisionada no ambiente real, mas **nenhum recurso
Azure real é criado ou necessário agora**.

**Estrutura:**

```
infra/
├── main.bicep               # Definição principal
├── modules/
│   ├── ai-search.bicep      # Azure AI Search
│   ├── openai.bicep         # Azure OpenAI
│   ├── functions.bicep      # Azure Functions
│   └── cosmos.bicep         # CosmosDB (sessão / logs)
└── parameters/
    ├── dev.bicepparam        # Parâmetros do ambiente de dev
    ├── staging.bicepparam
    └── prod.bicepparam
```

**Regras para esta fase:**
- Não execute `az deploy` nem `bicep build` — não há subscription Azure vinculada.
- Não modifique arquivos em `infra/` sem aprovação do Tech Lead e sem atualizar o
  parâmetro correspondente em todos os três ambientes (`dev`, `staging`, `prod`).
- Ao implementar `src/shared/config.ts` (CC-10), os nomes das variáveis de ambiente
  devem coincidir com os `outputs` definidos nos módulos Bicep — isso garante que
  o deploy futuro funcione sem ajustes.

**Referência cruzada:**
Os módulos Bicep correspondem a estas dependências do `requirements.md §10`:

| Módulo Bicep | Dependência correspondente |
|---|---|
| `ai-search.bicep` | Azure AI Search (AZURE_SEARCH_ENDPOINT, AZURE_SEARCH_API_KEY) |
| `openai.bicep` | Azure OpenAI (AZURE_OPENAI_ENDPOINT, AZURE_OPENAI_API_KEY) |
| `functions.bicep` | Azure Functions host do Query Endpoint e Feedback API |
| `cosmos.bicep` | Armazenamento de logs para BC-05 (Painel Web) |

> Derivado de: `Anexo C — infra/` · `specs/query-endpoint/requirements.md §10`

---

### SR-14 — Documentação operacional (`docs/runbooks/` e `docs/onboarding.md`)

#### `docs/onboarding.md`
**Status:** a ser escrito · **Responsável:** Tech Lead

Guia para novos membros do projeto. Quando escrito, deve cobrir:
- Pré-requisitos de ambiente (Node.js, pnpm/npm, uvx para MCP servers)
- Como configurar `.mcp/mcp.json` local (ver SR-11)
- Como popular `data/retrieval-corpus/` (ver SR-12)
- Como executar os testes: `npx vitest run` para unitários, `npx vitest run tests/integration` para integração
- Como executar o harness de avaliação de prompts: `prompts/eval/`
- Referência ao AGENTS.md como ponto de entrada para entender o projeto

#### `docs/runbooks/`
**Status:** a ser escrito · **Responsável:** Tech Lead + Dev Sênior

Runbooks operacionais para situações recorrentes. Candidatos prioritários baseados
nos incidentes e lacunas identificados:

| Runbook sugerido | Conteúdo | Origem |
|---|---|---|
| `recalibrar-threshold-dominio.md` | Como ajustar os limiares em `RETRIEVAL_THRESHOLDS` quando a taxa de GAP_DOCUMENTAL ultrapassar 15% em um domínio | L-03 · `guardrails.md §5` |
| `atualizar-sinonimos.md` | Como adicionar novos sinônimos a `prompts/synonym-table.json` via PR sem redeploy | PG-06 · SR-08 |
| `nova-versao-documento.md` | Como ingerir um novo documento ou uma nova versão, marcar a versão anterior como obsoleta e re-executar o pipeline | PG-02 · SR-04 |
| `responder-incident-falso-gap.md` | Como investigar e corrigir um falso GAP_DOCUMENTAL em produção | INC-03 · PG-05 |

> Derivado de: `Anexo C — docs/runbooks/` · `docs/onboarding.md`

---

### Referências consolidadas

| Artefato | Localização no repositório | Status | Versão |
|---|---|---|---|
| Constitution do projeto | `AGENTS.md` (este arquivo) | ✅ Vigente | — |
| Guardrails completos | `guardrails.md` | ✅ Vigente | v1.3.0 |
| Requirements Query Endpoint | `specs/query-endpoint/requirements.md` | ✅ Vigente | v1.1.0 |
| Requirements Pipeline Ingestão | `specs/pipeline-ingestao/requirements.md` | 🔲 A escrever | — |
| Requirements Feedback API | `specs/feedback-api/requirements.md` | 🔲 A escrever | — |
| Requirements Teams Bot | `specs/teams-bot/requirements.md` | 🔲 A escrever | — |
| Requirements Painel Web | `specs/painel-web/requirements.md` | 🔲 A escrever | — |
| Plan Query Endpoint | `specs/query-endpoint/plan.md` | 🔲 A escrever | — |
| Tasks Query Endpoint | `specs/query-endpoint/tasks.md` | 🔲 A gerar | — |
| ADR-001 a ADR-005 | `docs/adr/0001-*.md` a `docs/adr/0005-*.md` | 🔲 A criar | — |
| Template ADR | `docs/adr/template.md` | ✅ Existente | — |
| Onboarding | `docs/onboarding.md` | 🔲 A escrever | — |
| Runbooks | `docs/runbooks/` | 🔲 A escrever | — |
| MCP config | `.mcp/mcp.json` | 🔲 A criar | — |
| System prompt | `prompts/system-prompt.md` | ⚠️ Versão básica | — |
| Prompt changelog | `prompts/prompt-changelog.md` | 🔲 A criar | — |
| Tabela de sinônimos | `prompts/synonym-table.json` | 🔲 A criar | — |
| Tabela de encaminhamentos | `prompts/routing-table.json` | 🔲 A criar | — |
| Golden queries | `prompts/eval/golden-queries.json` | 🔲 A criar | — |
| Corpus de retrieval | `data/retrieval-corpus/` | ⚠️ Semeado pelo Anexo B | — |
| Docs NovaTech | `docs/novatech/` | ✅ Semeado pelo Anexo A | Anexo A |
| Skills foundation | `skills/foundation/` | 🔲 Pastas criadas, arquivos a escrever | — |
| Skills domain | `skills/domain/` | 🔲 Pastas criadas, arquivos a escrever | — |
| Skills artifact | `skills/artifact/` | 🔲 Pastas criadas, arquivos a escrever | — |
| Infra Bicep | `infra/` | ⚠️ Estado narrativo — não provisionar | — |
