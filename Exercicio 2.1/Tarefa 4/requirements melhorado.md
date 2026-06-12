# requirements.md — Query Endpoint
**Sistema:** Assistente de IA NovaTech
**Componente:** Query Endpoint (BC-02 — Atendimento e Consulta)
**Versão:** 1.1.0
**Data:** Junho 2026
**Responsável:** DB1 — Equipe de Produto

### Changelog
| Versão | Data | Alterações |
|---|---|---|
| 1.0.0 | Junho 2026 | Versão inicial |
| 1.1.0 | Junho 2026 | Resolução de ambiguidades AMB-01 a AMB-10 identificadas em revisão técnica: ADR-006 (mecanismo de detecção de contradição), limiares de GAP_DOCUMENTAL por domínio, RN-09 (classificador de intenção e termos canônicos), RN-10 (tabela de encaminhamento), RN-08 revisada (storage de sessão e isolamento), RN-01 revisada (exceção de chamados históricos removida), AC-03 corrigido (exemplo v1×v2 substituído), AC-07 reformulado (critério verificável), seção RNF adicionada (latência por estado, concorrência, disponibilidade), schema de request/response definido no AC-01, OQ-06 adicionada |

---

## 1. Overview

O Query Endpoint é o ponto de entrada do assistente de IA pelo qual um atendente envia uma pergunta em linguagem natural e recebe uma resposta fundamentada na documentação oficial da NovaTech. É a materialização do BC-02 (Atendimento e Consulta) e o único canal de interação com o sistema RAG para o time de atendimento.

Este documento cobre exclusivamente o comportamento funcional do endpoint de consulta. Decisões sobre ingestão documental, pipeline de vetorização e observabilidade são tratadas nos documentos de requisitos de BC-01 e BC-05.

---

## 2. Prior Decisions

As decisões abaixo foram tomadas na fase de discovery e modelagem de domínio. São tratadas aqui como restrições fixas, não como escolhas em aberto.

### ADR-001 — Arquitetura RAG sobre base documental estática
**Decisão:** O assistente opera sobre uma base vetorial pré-ingerida. Não realiza busca em tempo real no SharePoint, Confluence ou pasta de rede no momento da consulta.
**Consequência para este documento:** O Query Endpoint não possui dependência de latência com fontes externas. Toda resposta parte da base ingerida por BC-01. Documentos não ingeridos não existem para o endpoint.

### ADR-002 — Classificação de confiabilidade por tipo de documento
**Decisão:** Documentos são classificados em três níveis de confiabilidade: normativo (POL/PROC), contratual (SLA) e informal (FAQ). O endpoint deve propagar essa classificação na resposta.
**Consequência:** Respostas baseadas exclusivamente em FAQ devem carregar ressalva explícita. Respostas baseadas em documentos normativos ou contratuais não carregam ressalva.

### ADR-003 — Controle de versão como metadado obrigatório
**Decisão:** Toda resposta deve citar a versão do documento utilizado. Para o PROC-042, apenas a v2 (vigente pós 01/12/2023) deve ser recuperada para chamados novos.
**Consequência:** O endpoint deve rejeitar chunks do PROC-042 v1 para consultas sem contexto de data anterior a 01/12/2023. A v1 é tratada como obsoleta no índice vetorial.

### ADR-004 — Comportamento determinístico para contradições documentais
**Decisão:** Quando o sistema recuperar chunks de documentos conflitantes para a mesma pergunta, o endpoint não deve sintetizar uma resposta mesclando as versões. Deve sinalizar a contradição e acionar o fluxo de escalada.
**Consequência:** Existe um estado explícito de resposta denominado `CONTRADIÇÃO_DETECTADA`. Ver seção 5.

### ADR-005 — Integração via Microsoft Teams como canal primário
**Decisão:** O endpoint será consumido pelo bot integrado ao Teams. O atendente não acessa o endpoint diretamente.
**Consequência:** O contrato de request/response deve ser compatível com o conector do Teams. Latência máxima aceitável: 8 segundos (limite de timeout do Teams para bots).

### ADR-006 — Mecanismo de detecção de contradição: determinístico por metadados
**Decisão:** A detecção de contradição entre documentos é feita por comparação de metadados estruturados no momento do retrieval, não por inferência do LLM. Dois chunks são considerados conflitantes quando satisfazem simultaneamente três condições: (1) o campo `topic_tag` é idêntico nos dois chunks, (2) os campos `doc_id` são diferentes, e (3) o campo `attribute_key` — que identifica o atributo de negócio representado pelo chunk (ex: `multiplicador_regional_norte`, `prazo_adicional_frete`) — é idêntico nos dois chunks com valores divergentes.
**Consequência:** BC-01 é responsável por atribuir `topic_tag` e `attribute_key` a cada chunk no momento da ingestão. Sem esses metadados, a detecção de contradição não funciona. O LLM não participa da decisão de sinalizar contradição — apenas da geração do texto de alerta após a detecção. Isso elimina o risco de o modelo "resolver" silenciosamente o conflito.
**Limitação conhecida:** Contradições semânticas não capturadas por `attribute_key` idêntico (ex: duas políticas que se contradizem em implicação, não em valor explícito) não são detectadas por este mecanismo. Esse escopo expandido de detecção está fora do orçamento do projeto atual.
**Fonte:** AMB-01 identificada na revisão técnica de requisitos.

---

## 3. Scope Boundaries

Derivados diretamente dos bounded contexts definidos na fase de modelagem de domínio.

### 3.1 Dentro do escopo deste endpoint

| Capacidade | Contexto de origem |
|---|---|
| Receber pergunta em linguagem natural | BC-02 |
| Recuperar chunks relevantes da base vetorial | BC-02 → BC-01 |
| Classificar a confiabilidade da resposta | BC-02 |
| Gerar resposta com citação de fonte, versão e seção | BC-02 |
| Sinalizar contradição entre documentos recuperados | BC-02 (ADR-004) |
| Retornar `não encontrado` quando sem cobertura documental | BC-02 |
| Preservar histórico de consultas da sessão corrente | BC-02 |
| Identificar e propagar o tipo de documento utilizado | BC-02 → BC-01 (ADR-002) |

### 3.2 Fora do escopo deste endpoint

| O que não faz | Onde está |
|---|---|
| Calcular o valor final do frete | BC-04 (depende de tabela mensal externa) |
| Verificar o SLA real de um chamado em aberto | BC-03 (depende do Azure DevOps) |
| Consultar o tier atual do cliente | BC-03 (depende de integração CRM/ERP — não provisionada) |
| Abrir ou atualizar chamados | Portal do Cliente / Azure DevOps |
| Ingerir ou atualizar documentos na base vetorial | BC-01 |
| Registrar logs de uso e feedback | BC-05 (consumidor do evento emitido por este endpoint) |
| Editar, criar ou aprovar documentos | Responsabilidade das áreas de negócio da NovaTech |

### 3.3 Fronteira de dados

O endpoint **não acessa** e **não deve acessar**:
- PII de clientes finais da NovaTech
- Dados contratuais individuais de clientes (valores, histórico de chamados)
- Credenciais ou dados de autenticação do atendente além do token de sessão

---

## 4. User Outcomes

Os outcomes abaixo descrevem o que o atendente consegue **fazer** ou **deixar de sofrer** como resultado do endpoint funcionando corretamente. Não descrevem features técnicas.

### UO-01 — Obter a resposta certa sem sair do Teams

> **"Preciso responder ao cliente agora, sem abrir três abas diferentes e sem perguntar para o colega ao lado."**

O atendente digita sua dúvida no Teams e recebe, na mesma janela, a informação de que precisa com a indicação de qual documento e versão a sustenta. Não há mudança de contexto, não há busca manual, não há dependência de quem "sabe de cor".

**Medida de sucesso:** tempo médio de obtenção da informação cai de 12 min para menos de 2 min em chamados com cobertura documental.

---

### UO-02 — Saber quando confiar na resposta

> **"Preciso saber se posso passar essa informação para o cliente ou se devo verificar antes."**

O atendente consegue distinguir, pela resposta do assistente, se está recebendo uma informação de um documento normativo vigente, de um contrato em vigor ou de um FAQ não validado. Não precisa conhecer a estrutura documental da empresa para avaliar o peso da resposta.

**Medida de sucesso:** respostas baseadas em FAQ sempre carregam marcação visual de "fonte informal" no canal do Teams. Atendentes não encaminham informações de FAQ como definitivas sem verificação.

---

### UO-03 — Não ser responsabilizado por informação contraditória

> **"Não quero dar uma resposta ao cliente e descobrir depois que existia outra regra diferente que eu não sabia."**

Quando existem documentos conflitantes sobre o tema da pergunta (ex.: PROC-042 v1 vs v2 com multiplicadores diferentes), o assistente não escolhe silenciosamente um deles. Ele avisa o atendente que há uma contradição e indica o caminho de escalada para obter a resposta oficial.

**Medida de sucesso:** zero casos de atendente encaminhando informação de frete calculada com multiplicadores da v1 após o go-live. Taxa de escaladas por contradição rastreável em BC-05.

---

### UO-04 — Saber quando o assistente não pode ajudar

> **"Prefiro saber que não tem resposta do que receber uma resposta inventada."**

Quando o tema da pergunta não tem cobertura na base documental (ex.: frete padrão abaixo de 500kg, seguro de carga), o assistente diz claramente que não encontrou informação na documentação oficial e sugere a quem recorrer — sem simular uma resposta ou extrapolar com base em documentos próximos.

**Medida de sucesso:** nenhum caso de resposta gerada sobre tema sem cobertura documental. Gaps recorrentes são rastreados em BC-05 e retroalimentam BC-01.

---

### UO-05 — Continuar a conversa sem perder o contexto

> **"Faço várias perguntas sobre o mesmo chamado. Não quero repetir o contexto toda vez."**

Dentro de uma mesma sessão de atendimento, o assistente lembra o que foi perguntado antes. O atendente pode refinar ou detalhar a consulta sem reformular do zero — como em uma conversa, não como em pesquisas avulsas.

**Medida de sucesso:** atendentes conseguem fazer follow-up de consultas sem repetir o contexto do chamado em cada mensagem.

---

## 5. Estados de Resposta

O endpoint sempre retorna um dos quatro estados abaixo. Cada estado tem comportamento, conteúdo e UX distintos. Não existe estado intermediário ou implícito.

```
┌─────────────────────────────────────────────────────────────────┐
│  Pergunta do atendente                                          │
└───────────────────────────┬─────────────────────────────────────┘
                            │
              ┌─────────────▼──────────────┐
              │  Recuperação semântica     │
              │  na base vetorial (BC-01)  │
              └─────────────┬──────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ▼                   ▼                   ▼
  Chunks          Chunks de docs        Nenhum chunk
  de 1 doc        conflitantes          relevante
  consistente     (mesmo tema)          encontrado
        │                   │                   │
        ▼                   ▼                   ▼
  Classifica             CONTRADIÇÃO        GAP
  confiabilidade         DETECTADA       DOCUMENTAL
  do documento               │               │
        │                    ▼               ▼
        ▼              [UO-03]          [UO-04]
  Normativo/         Alerta +         "Não encontrado
  Contratual?        Escalada         na doc. oficial"
        │                             + encaminhamento
        ├── Sim → FUNDAMENTADA
        │         [UO-01, UO-02]
        │
        └── Não (só FAQ) → COM RESSALVA
                            [UO-02]
```

### Estado 1 — FUNDAMENTADA
- Fonte: documento normativo (POL/PROC) ou contratual (SLA) vigente
- Resposta inclui: texto da resposta + nome do documento + versão + seção
- UX: resposta direta, sem aviso adicional
- Rastreabilidade: chunk_id, doc_id, versão, seção

### Estado 2 — COM_RESSALVA
- Fonte: exclusivamente FAQ-Atendimento (documento informal)
- Resposta inclui: texto da resposta + aviso explícito "Fonte: FAQ interno — não validado por Compliance. Confirme com supervisor antes de encaminhar ao cliente."
- UX: marcação visual diferenciada no Teams (ex.: ícone de alerta amarelo)
- Rastreabilidade: chunk_id, doc_id, flag `informal: true`

### Estado 3 — CONTRADIÇÃO_DETECTADA
- Trigger: chunks recuperados de documentos diferentes com conteúdo conflitante sobre o mesmo atributo (ex.: multiplicador regional Norte em PROC-042 v1 e v2)
- Resposta inclui: identificação dos documentos conflitantes + valores divergentes + instrução de escalada ("Consulte seu supervisor ou a área de Operações antes de responder ao cliente.")
- UX: marcação visual de alerta vermelho no Teams. Não exibe nenhum dos valores conflitantes como correto.
- O endpoint **não escolhe** entre os documentos conflitantes — nunca.
- Rastreabilidade: chunk_ids de ambos os documentos, flag `contradição: true`, documentos envolvidos

### Estado 4 — GAP_DOCUMENTAL
- Trigger: nenhum chunk com score de relevância acima do **limiar configurável por categoria de pergunta** (ver tabela abaixo) é recuperado na busca vetorial
- O limiar não é um valor único global — é parametrizado por domínio para evitar falsos gaps em domínios com vocabulário técnico denso e falsos positivos em domínios com documentação esparsa

**Tabela de limiares iniciais por domínio (calibração pós go-live):**

| Domínio | Limiar inicial | Responsável pela calibração |
|---|---|---|
| Frete e logística (BC-04) | 0,75 | DB1 — ajuste nas primeiras 2 semanas de operação |
| Devolução e política (BC-02) | 0,72 | DB1 — ajuste nas primeiras 2 semanas de operação |
| SLA e contratos (BC-03) | 0,78 | DB1 — ajuste nas primeiras 2 semanas de operação |
| Outros / não classificado | 0,70 | DB1 — revisão mensal via BC-05 |

- Os limiares acima são **valores de partida** baseados em benchmarks de RAG para domínios técnicos em português. Serão ajustados com base nos logs de BC-05 após as primeiras duas semanas de operação.
- A categoria do domínio é inferida pelo classificador de intenção no pré-processamento da query (ver seção 6, RN-09). Se a classificação falhar, aplica-se o limiar padrão `0,70`.
- Resposta inclui: "Não encontrei informações sobre este tema na documentação oficial." + encaminhamento baseado na tabela de roteamento da RN-10
- UX: resposta neutra, sem simulação de conteúdo
- O endpoint **nunca extrapola** com base em documentos adjacentes.
- Rastreabilidade: query anonimizada + domínio classificado + score máximo obtido registrados em BC-05 como gap recorrente

---

## 6. Regras de Negócio

### RN-01 — Versão vigente do PROC-042
Para todas as consultas sobre frete especial, o endpoint utiliza exclusivamente chunks do **PROC-042 v2**. Chunks do PROC-042 v1 são marcados como `status: obsoleto` no índice vetorial de BC-01 e **nunca são recuperados**, independentemente do conteúdo da query.

A disposição transitória do PROC-042 v2 §5 (chamados abertos antes de 01/12/2023 usam multiplicadores da v1) **não é suportada pelo endpoint**. Chamados históricos nessa condição devem ser tratados manualmente pelo atendente com consulta direta ao documento v1 arquivado fora do sistema.

**Justificativa da decisão:** a exceção original exigiria que o endpoint identificasse a data de abertura do chamado, o que depende de integração com o Azure DevOps — explicitamente fora do escopo (seção 3.2). Implementar a exceção via inferência do LLM introduz risco de alucinação de data. O custo de tratar os poucos chamados remanescentes manualmente é menor que o risco técnico.

**Fonte:** PROC-042 v2 §5 · ADR-003 · AMB-07 identificada na revisão técnica de requisitos.

---

### RN-02 — Precedência de fonte em respostas mistas
Quando chunks de um documento normativo e de um documento informal (FAQ) abordam o mesmo tema sem contradição direta, a resposta é gerada a partir do documento normativo. O FAQ é ignorado neste caso.

Se o FAQ traz informação que o documento normativo não cobre (ex.: tempo de aprovação de frete expresso para carga perigosa — Item 32), a resposta pode usar o FAQ com o estado COM_RESSALVA obrigatório.

**Fonte:** ADR-002 · BC-01 (classificação de confiabilidade)

---

### RN-03 — Incidente crítico: limiar de valor
O endpoint usa **R$ 100.000** como limiar de valor declarado para classificar um chamado como incidente crítico, conforme SLA-2024 §3. O limiar de R$ 50.000 mencionado no FAQ Item 27 é descartado.

**Fonte:** SLA-2024 §3 · contradição C5 mapeada na fase de domínio

---

### RN-04 — Frete padrão (abaixo de 500kg)
O endpoint não possui documentação de suporte para frete padrão. Qualquer pergunta sobre frete abaixo de 500kg resulta em GAP_DOCUMENTAL com encaminhamento ao Comercial.

Esta regra permanece ativa até que BC-01 registre um documento normativo vigente cobrindo este tema.

**Fonte:** Gap documental identificado na fase de discovery · BC-04 (fora do escopo)

---

### RN-05 — Tier Platinum
Perguntas sobre tier Platinum resultam em resposta FUNDAMENTADA com a informação de que o tier foi descontinuado em 2022 e os três tiers vigentes são Gold, Silver e Standard.

**Fonte:** SLA-2024 §1 · FAQ Item 15 (consistente com o normativo neste ponto)

---

### RN-06 — Relógio de SLA: dois comportamentos
O endpoint distingue dois comportamentos ao responder sobre SLA:

- **Chamados gerais:** relógio pausa fora do horário comercial (08h–18h, dias úteis)
- **Incidentes críticos de clientes Gold:** relógio não pausa — corre 24/7

O endpoint nunca responde sobre SLA sem especificar qual comportamento se aplica. A distinção é obrigatória na resposta.

**Fonte:** SLA-2024 §5 · risco alto identificado na linguagem ubíqua (Relógio de SLA)

---

### RN-07 — Carga perigosa: escopo de classes
O endpoint responde sobre carga perigosa referenciando exclusivamente as **classes 1 a 6 da ANTT** (Res. 5.947/2021), conforme POL-001 §3.2. Para perguntas que envolvam classes 7, 8 ou 9 (radioativos, corrosivos, miscelânea), o endpoint retorna GAP_DOCUMENTAL.

**Fonte:** POL-001 §3.2 · gap identificado na linguagem ubíqua

---

### RN-08 — Sessão e histórico
O contexto de sessão é mantido por **60 minutos sem nova mensagem do atendente** na conversa do Teams. "Inatividade" é definida como ausência de mensagem enviada pelo atendente — o envio de resposta pelo bot não reinicia o timer.

O estado de sessão é armazenado em **cache server-side com TTL de 60 minutos** (Azure Cache for Redis, instância provisionada junto com o backend). Cada sessão é isolada por `user_id` extraído do token AAD do Teams — dois atendentes nunca compartilham sessão.

Em caso de reinício do serviço (deploy, falha, escalamento), o histórico de sessões em andamento é perdido. O atendente recebe a mensagem: *"Sua sessão anterior foi encerrada. Por favor, reformule sua pergunta com o contexto necessário."*

O histórico de sessão **não é armazenado de forma persistente** após o encerramento. BC-05 registra apenas logs anonimizados de consulta, não o histórico completo de sessão.

**Fonte:** AMB-04 identificada na revisão técnica de requisitos.

---

### RN-09 — Classificador de intenção e mapeamento de termos canônicos
Antes do embedding e da busca vetorial, toda query passa por um **pré-processador de normalização** com duas funções:

**Função 1 — Mapeamento de termos proibidos para termos canônicos:**
O pré-processador substitui sinônimos não-canônicos pelo termo canônico da linguagem ubíqua antes de gerar o embedding. A substituição é feita por correspondência exata (case-insensitive) em uma tabela de sinônimos mantida como configuração versionada no repositório do projeto.

| Termo detectado na query | Substituído por |
|---|---|
| frete diferenciado, frete pesado | Frete Especial |
| chamado P1, chamado urgente, prioridade alta, prioridade 1 | Incidente Crítico |
| carga especial, material perigoso | Carga Perigosa |
| categoria de cliente, nível de cliente, Platinum | Tier de Cliente |
| 7 dias corridos, uma semana | Prazo de Devolução |
| coleta de devolução | Coleta Reversa |

A tabela de sinônimos é mantida pelo time de produto (DB1) e revisada mensalmente com base nos logs de BC-05. Novos sinônimos identificados em produção são adicionados via PR no repositório — não requerem redeploy do serviço.

**Função 2 — Classificação de domínio da query:**
O pré-processador classifica a query em um dos domínios abaixo para aplicar o limiar de relevância correto (RN-10) e o encaminhamento adequado no GAP_DOCUMENTAL (RN-10):

| Domínio classificado | Palavras-chave primárias | Limiar RAG |
|---|---|---|
| `frete_logistica` | frete, carga, peso, prazo, entrega, multiplicador, região | 0,75 |
| `devolucao_politica` | devolução, retorno, coleta, reembolso, lacre, CT-e | 0,72 |
| `sla_contrato` | SLA, prazo, resposta, resolução, Gold, Silver, Standard, tier | 0,78 |
| `nao_classificado` | qualquer outra query | 0,70 |

Se a query contiver palavras-chave de dois domínios simultaneamente (edge case E2), o classificador atribui o domínio com maior contagem de palavras-chave. Em caso de empate, aplica `nao_classificado`.

**Fonte:** AMB-03 e AMB-08 identificadas na revisão técnica de requisitos.

---

### RN-10 — Tabela de encaminhamento para GAP_DOCUMENTAL
Quando o estado GAP_DOCUMENTAL é acionado, o encaminhamento sugerido na resposta é determinado pelo domínio classificado pelo RN-09. A tabela abaixo é a fonte única de encaminhamentos — não está hardcoded no prompt do LLM.

| Domínio | Encaminhamento exibido ao atendente |
|---|---|
| `frete_logistica` | "Consulte a área Comercial para informações sobre este tipo de frete." |
| `devolucao_politica` — carga danificada | "Registre a ocorrência em sinistros@novatech.com.br com fotos e laudo em até 48h." |
| `devolucao_politica` — outros | "Consulte a área de Operações ou acesse o Portal do Cliente." |
| `sla_contrato` | "Consulte o Comercial responsável pela conta do cliente." |
| `nao_classificado` | "Consulte seu supervisor para orientação sobre este tema." |

Esta tabela é **configuração de produto** — mantida pelo time DB1 em arquivo versionado no repositório, não no código do LLM. Alterações de e-mail ou área de destino são feitas via PR, sem redeploy do modelo.

O e-mail `sinistros@novatech.com.br` está parametrizado como variável de ambiente — não está hardcoded. Mudanças de contato são feitas na configuração do ambiente, não no código.

**Fonte:** AMB-08 identificada na revisão técnica de requisitos.

---

## 7. Requisitos Não-Funcionais

### RNF-01 — Latência por estado de resposta

| Estado | Latência máxima (p95) | Latência máxima absoluta | Comportamento em timeout |
|---|---|---|---|
| FUNDAMENTADA | 5s | 8s | Retorna erro 504 com mensagem: "O assistente está demorando mais que o esperado. Tente novamente em instantes." |
| COM_RESSALVA | 5s | 8s | Idem |
| CONTRADIÇÃO_DETECTADA | 6s | 8s | Idem — detecção por metadados adiciona ~1s ao pipeline |
| GAP_DOCUMENTAL | 3s | 8s | Idem — não aciona LLM para geração, apenas recupera encaminhamento da tabela RN-10 |

Os 8 segundos são medidos **end-to-end no serviço** (do recebimento do request ao envio do response). O overhead de transporte do Teams não está incluído — esse SLA é responsabilidade da infraestrutura Microsoft.

Em caso de timeout, o estado da sessão é preservado. O atendente pode reenviar a mesma pergunta sem perder o histórico.

### RNF-02 — Concorrência mínima

O endpoint deve suportar **45 requisições simultâneas** sem degradação de latência acima dos limites do RNF-01. Esse valor corresponde ao pico teórico de todos os atendentes consultando o sistema ao mesmo tempo.

- Baseline esperado em operação normal: 8–12 requisições simultâneas (distribuição ao longo do dia)
- Pico estimado: 20–30 requisições simultâneas (início do turno e pós-reuniões)
- Capacidade máxima provisionada: 45 requisições simultâneas (headroom de 50% sobre o pico estimado)

Testes de carga devem ser executados antes do go-live simulando 45 usuários concorrentes com queries representativas dos 4 domínios classificados.

### RNF-03 — Disponibilidade

O endpoint deve ter disponibilidade mínima de **99,0%** em horário comercial (08h–18h, dias úteis), alinhado ao SLA Standard da NovaTech. Fora do horário comercial, não há SLA de disponibilidade definido para o MVP.

---

## 8. Acceptance Criteria

Os critérios abaixo são verificáveis em teste. Cada um está rastreado ao outcome que valida.

### AC-01 · UO-01
**Dado** que um atendente envia uma pergunta coberta por documento normativo vigente,
**Quando** o endpoint processa a consulta,
**Então** a resposta é retornada em até **8 segundos** (p95: 5s) e contém o nome do documento, a versão e a seção de origem.

O request deve seguir o schema abaixo (ver OQ-06 para campos pendentes de decisão):

```json
{
  "query": "string — pergunta em linguagem natural (obrigatório, max 1000 chars)",
  "session_id": "string — UUID da sessão corrente (obrigatório; se ausente, nova sessão é criada)",
  "user_id": "string — AAD object ID do atendente extraído do token Teams (obrigatório)",
  "tier_hint": "enum[Gold, Silver, Standard] — tier do cliente informado pelo atendente (opcional; ver OQ-05)"
}
```

O response deve seguir o schema abaixo:

```json
{
  "state": "enum[FUNDAMENTADA, COM_RESSALVA, CONTRADIÇÃO_DETECTADA, GAP_DOCUMENTAL]",
  "answer": "string — texto da resposta gerada",
  "session_id": "string — UUID da sessão (mesmo do request ou novo se sessão criada)",
  "confidence_level": "enum[normativo, contratual, informal] — presente apenas nos estados FUNDAMENTADA e COM_RESSALVA",
  "disclaimer": "string — presente apenas no estado COM_RESSALVA",
  "sources": [
    {
      "doc_id": "string",
      "version": "string",
      "section": "string",
      "chunk_id": "string"
    }
  ],
  "conflicting_docs": ["string"] ,
  "routing_suggestion": "string — presente apenas no estado GAP_DOCUMENTAL",
  "domain_classified": "enum[frete_logistica, devolucao_politica, sla_contrato, nao_classificado]"
}
```

---

### AC-02 · UO-02
**Dado** que a única fonte relevante para a pergunta é o FAQ-Atendimento,
**Quando** o endpoint retorna a resposta,
**Então** o campo `confidence_level` é `informal` e o campo `disclaimer` contém o texto: *"Fonte: FAQ interno — não validado por Compliance. Confirme com supervisor antes de encaminhar ao cliente."*

---

### AC-03 · UO-03
**Dado** que o sistema recupera chunks de dois documentos vigentes com `attribute_key` idêntico e valores divergentes — por exemplo, POL-001 §3.3 e FAQ Item 38 descrevendo prazos diferentes para o mesmo procedimento de devolução,
**Quando** o endpoint processa a consulta,
**Então** o estado retornado é `CONTRADIÇÃO_DETECTADA`, nenhum dos valores divergentes é apresentado como correto, a instrução de escalada está presente na resposta, e o campo `conflicting_docs` no response body lista os dois `doc_id` envolvidos.

> **Nota:** conflitos entre PROC-042 v1 e v2 não acionam este AC porque a v1 é marcada como `status: obsoleto` no índice e nunca é recuperada (ver RN-01 e AC-04). Este AC cobre contradições entre documentos **ambos vigentes** na base.

---

### AC-04 · UO-03
**Dado** que uma consulta sobre frete especial é feita sem contexto de data,
**Quando** o endpoint recupera chunks,
**Então** apenas chunks do PROC-042 v2 são utilizados na geração da resposta. Chunks do PROC-042 v1 não aparecem no resultado.

---

### AC-05 · UO-04
**Dado** que o atendente pergunta sobre frete padrão (abaixo de 500kg),
**Quando** o endpoint processa a consulta,
**Então** o estado retornado é `GAP_DOCUMENTAL` e a resposta inclui o encaminhamento ao Comercial. Nenhum valor ou regra é sugerido pelo modelo.

---

### AC-06 · UO-04
**Dado** que o atendente pergunta sobre carga classificada como classe 8 da ANTT (corrosivos),
**Quando** o endpoint processa a consulta,
**Então** o estado retornado é `GAP_DOCUMENTAL` com mensagem indicando que a documentação cobre apenas classes 1 a 6.

---

### AC-07 · UO-05
**Dado** que o atendente enviou uma consulta sobre devolução em uma sessão ativa (ex: "qual o prazo de devolução?") e recebeu uma resposta FUNDAMENTADA com `doc_id: POL-001`,
**Quando** o atendente envia uma segunda mensagem de follow-up sem repetir o contexto (ex.: "e o prazo para coleta?"),
**Então:**
1. O request para o LLM contém o histórico da consulta anterior no campo `session_history` (verificável no log do serviço)
2. A resposta retornada referencia o mesmo contexto de devolução — identificado pelo campo `doc_id: POL-001` na rastreabilidade da resposta
3. O campo `session_id` da segunda resposta é igual ao da primeira

**O que este AC não verifica:** coerência semântica do texto gerado pelo LLM — essa validação é feita em testes exploratórios manuais, não em testes automatizados.

---

### AC-08 · UO-02
**Dado** que o atendente pergunta sobre o SLA de um chamado geral de cliente Gold,
**Quando** o endpoint retorna a resposta,
**Então** a resposta especifica explicitamente que o relógio de SLA **pausa** fora do horário comercial para chamados gerais, mesmo que o cliente seja Gold.

---

### AC-09 · UO-02
**Dado** que o atendente pergunta sobre o SLA de um incidente crítico de cliente Gold,
**Quando** o endpoint retorna a resposta,
**Então** a resposta especifica explicitamente que o relógio de SLA **não pausa** e corre 24/7.

---

### AC-10 · UO-01 + UO-04
**Dado** que o atendente pergunta sobre tier Platinum,
**Quando** o endpoint processa a consulta,
**Então** a resposta é FUNDAMENTADA, informa que o tier foi descontinuado em 2022 e lista os tiers vigentes (Gold, Silver, Standard). Estado não é GAP_DOCUMENTAL.

---

## 9. Edge Cases e Cenários de Risco

Os cenários abaixo têm alto potencial de gerar bugs ou comportamentos incorretos silenciosos. Devem ser cobertos em testes de regressão.

| # | Cenário | Risco | Comportamento esperado |
|---|---|---|---|
| E1 | Pergunta menciona "frete diferenciado" (termo proibido para Frete Especial) | Modelo pode não recuperar chunks corretos | Sistema deve mapear sinônimos proibidos para o termo canônico antes da busca |
| E2 | Pergunta mistura dois domínios: "qual o SLA para devolução de carga perigosa Gold?" | Chunks de BC-03 e BC-04 recuperados simultaneamente | Resposta deve tratar cada domínio separadamente sem mesclar regras |
| E3 | Atendente pergunta sobre "chamado P1" (termo proibido para Incidente Crítico) | Modelo pode não reconhecer como incidente crítico | Sistema deve mapear para o termo canônico Incidente Crítico |
| E4 | Pergunta sobre prazo de devolução com data informada pelo cliente (não do tracking) | Modelo pode usar a data errada como base da contagem | Resposta deve explicitar que o prazo começa na data confirmada no sistema de tracking |
| E5 | FAQ e POL-001 cobrem o mesmo tema sem contradição direta, mas com detalhes diferentes | Modelo pode mesclar as duas fontes | Apenas POL-001 deve ser utilizada; FAQ ignorado (RN-02) |
| E6 | Pergunta sobre carga perigosa com frete expresso (FAQ Item 32, sem PROC formal) | Única fonte é FAQ — risco jurídico | Estado COM_RESSALVA obrigatório + instrução de verificar com Compliance antes de confirmar ao cliente |
| E7 | Sessão expirada (>60min) e atendente envia follow-up | Modelo sem contexto pode gerar resposta incoerente | Sistema deve informar que a sessão expirou e solicitar que a pergunta seja reformulada com contexto |
| E8 | Atendente pergunta sobre desconto de volume usando limiar do FAQ (10/mês) | FAQ e PROC-042 v2 divergem (10 vs 8 fretes/mês) | Estado CONTRADIÇÃO_DETECTADA + escalada (contradição C5 mapeada) |

---

## 10. Dependencies

### Dependências bloqueantes (o endpoint não funciona sem elas)

| Dependência | Tipo | Responsável | Status |
|---|---|---|---|
| BC-01 deve ter PROC-042 v1 marcado como obsoleto no índice vetorial | Técnica | DB1 | ⚠️ Pendente — NovaTech precisa formalizar a versão vigente (ADR-003) |
| BC-01 deve expor metadados de fonte (doc_id, versão, seção, tipo) por chunk | Técnica | DB1 | 🔲 A implementar |
| Microsoft Teams Bot provisionado no tenant da NovaTech | Infraestrutura | NovaTech + DB1 | 🔲 A provisionar |
| Azure AI Services provisionado (embedding + LLM) | Infraestrutura | NovaTech | 🔲 A provisionar |

### Dependências não-bloqueantes (endpoint funciona, mas com capacidade reduzida)

| Dependência | Impacto se ausente |
|---|---|
| BC-05 operacional | Logs de uso e gaps não são registrados. Melhoria contínua inoperante. |
| FAQ ingerido com flag `informal: true` | Perguntas sobre temas cobertos apenas pelo FAQ retornam GAP_DOCUMENTAL em vez de COM_RESSALVA |
| Glossário de termos proibidos configurado no pré-processamento | Edge cases E1 e E3 não são tratados — risco de miss na recuperação |

---

## 11. Out of Scope (explícito)

Os itens abaixo foram solicitados ou considerados durante o discovery e foram **deliberadamente excluídos** deste documento e deste componente.

| Item | Motivo da exclusão | Onde tratado |
|---|---|---|
| Cálculo do valor final do frete | Depende de tabela mensal externa não ingerível estaticamente | BC-04 — requirements futuros |
| Consulta ao tier real do cliente via CRM | Integração com CRM/ERP não está no escopo do projeto atual | BC-03 — decisão de integração pendente |
| Verificação do SLA em tempo real de um chamado | Requer leitura do Azure DevOps — fora do modelo RAG | BC-03 — requirements futuros |
| Autoaprendizado com base em feedbacks | Fora do orçamento e do prazo de 3 meses | Roadmap pós go-live |
| Resposta em voz (Teams voice) | Não solicitado pela NovaTech | Não planejado |
| Suporte a outros idiomas | Base documental em português apenas | Não aplicável |

---

## 12. Open Questions

As questões abaixo **bloqueiam ou afetam** critérios de aceite específicos. Precisam de resposta da NovaTech antes do início do desenvolvimento do endpoint.

| # | Questão | Impacto | AC afetado | Prazo sugerido |
|---|---|---|---|---|
| OQ-01 | O FAQ-Atendimento será ingerido na base vetorial? | Define se estado COM_RESSALVA existe ou todos os temas de FAQ viram GAP_DOCUMENTAL | AC-02, E6 | Antes do Sprint 1 |
| OQ-02 | Qual o comportamento esperado para perguntas sobre seguro de carga? | FAQ Item 22 é a única fonte — risco jurídico de repassar percentuais sem validade contratual | AC-05 (escopo) | Antes do Sprint 1 |
| OQ-03 | A NovaTech formalizará a obsolescência do PROC-042 v1 no SharePoint antes do go-live? | Se não formalizado, ADR-003 depende de controle técnico no índice vetorial sem respaldo documental | AC-03, AC-04 | Antes do Sprint 2 |
| OQ-04 | Haverá encaminhamento automático para supervisor via Teams quando estado for CONTRADIÇÃO_DETECTADA, ou apenas orientação textual? | Define se o endpoint precisa de integração com o sistema de chamados | AC-03 | Antes do Sprint 2 |
| OQ-05 | O atendente precisa informar o tier do cliente na pergunta, ou o endpoint deve inferir do contexto da sessão? | Afeta RN-06 (Relógio de SLA) e respostas sobre incidente crítico | AC-08, AC-09 | Antes do Sprint 1 |
| OQ-06 | O campo `tier_hint` no request será preenchido pelo bot do Teams automaticamente (via integração CRM) ou digitado pelo atendente em texto livre? | Define se o campo é estruturado (enum confiável) ou requer parsing de NLP com risco de erro de extração | AC-08, AC-09, RN-06 | Antes do Sprint 1 |
