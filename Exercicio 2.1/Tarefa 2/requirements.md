# requirements.md — Query Endpoint
**Sistema:** Assistente de IA NovaTech
**Componente:** Query Endpoint (BC-02 — Atendimento e Consulta)
**Versão:** 1.0.0
**Data:** Junho 2026
**Responsável:** DB1 — Equipe de Produto

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
- Trigger: nenhum chunk com score de relevância acima do limiar mínimo recuperado
- Resposta inclui: "Não encontrei informações sobre este tema na documentação oficial." + sugestão de encaminhamento baseada na categoria da pergunta (ex.: para frete padrão → Comercial; para carga danificada → sinistros@novatech.com.br)
- UX: resposta neutra, sem simulação de conteúdo
- O endpoint **nunca extrapola** com base em documentos adjacentes.
- Rastreabilidade: query anonimizada registrada em BC-05 como gap recorrente

---

## 6. Regras de Negócio

### RN-01 — Versão vigente do PROC-042
Para consultas sobre frete especial sem contexto de data explícito, o endpoint utiliza exclusivamente chunks do **PROC-042 v2**. Chunks do PROC-042 v1 não são recuperados para consultas novas.

**Exceção:** se a pergunta contiver referência explícita a chamado aberto antes de 01/12/2023 (ex.: "chamado aberto em novembro de 2023"), o endpoint pode recuperar v1 e deve sinalizar a versão utilizada na resposta.

**Fonte:** PROC-042 v2 §5 (disposição transitória) · ADR-003

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
O contexto de sessão é mantido por **60 minutos de inatividade**. Após esse período, a sessão é encerrada e o histórico não é recuperável. Uma nova pergunta inicia uma nova sessão.

O histórico de sessão **não é armazenado de forma persistente** após o encerramento. BC-05 registra apenas logs anonimizados de consulta, não o histórico completo de sessão.

---

## 7. Acceptance Criteria

Os critérios abaixo são verificáveis em teste. Cada um está rastreado ao outcome que valida.

### AC-01 · UO-01
**Dado** que um atendente envia uma pergunta coberta por documento normativo vigente,
**Quando** o endpoint processa a consulta,
**Então** a resposta é retornada em até **8 segundos** e contém o nome do documento, a versão e a seção de origem.

---

### AC-02 · UO-02
**Dado** que a única fonte relevante para a pergunta é o FAQ-Atendimento,
**Quando** o endpoint retorna a resposta,
**Então** o campo `confidence_level` é `informal` e o campo `disclaimer` contém o texto: *"Fonte: FAQ interno — não validado por Compliance. Confirme com supervisor antes de encaminhar ao cliente."*

---

### AC-03 · UO-03
**Dado** que o sistema recupera chunks do PROC-042 v1 e v2 com valores de multiplicador regional divergentes para a mesma região,
**Quando** o endpoint processa a consulta,
**Então** o estado retornado é `CONTRADIÇÃO_DETECTADA`, nenhum valor de multiplicador é apresentado como correto, e a instrução de escalada está presente na resposta.

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
**Dado** que o atendente fez uma consulta sobre devolução em uma sessão ativa,
**Quando** o atendente envia uma segunda mensagem de follow-up sem repetir o contexto (ex.: "e o prazo para coleta?"),
**Então** o endpoint interpreta a pergunta no contexto da consulta anterior e retorna resposta consistente com o histórico da sessão.

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

## 8. Edge Cases e Cenários de Risco

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

## 9. Dependencies

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

## 10. Out of Scope (explícito)

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

## 11. Open Questions

As questões abaixo **bloqueiam ou afetam** critérios de aceite específicos. Precisam de resposta da NovaTech antes do início do desenvolvimento do endpoint.

| # | Questão | Impacto | AC afetado | Prazo sugerido |
|---|---|---|---|---|
| OQ-01 | O FAQ-Atendimento será ingerido na base vetorial? | Define se estado COM_RESSALVA existe ou todos os temas de FAQ viram GAP_DOCUMENTAL | AC-02, E6 | Antes do Sprint 1 |
| OQ-02 | Qual o comportamento esperado para perguntas sobre seguro de carga? | FAQ Item 22 é a única fonte — risco jurídico de repassar percentuais sem validade contratual | AC-05 (escopo) | Antes do Sprint 1 |
| OQ-03 | A NovaTech formalizará a obsolescência do PROC-042 v1 no SharePoint antes do go-live? | Se não formalizado, ADR-003 depende de controle técnico no índice vetorial sem respaldo documental | AC-03, AC-04 | Antes do Sprint 2 |
| OQ-04 | Haverá encaminhamento automático para supervisor via Teams quando estado for CONTRADIÇÃO_DETECTADA, ou apenas orientação textual? | Define se o endpoint precisa de integração com o sistema de chamados | AC-03 | Antes do Sprint 2 |
| OQ-05 | O atendente precisa informar o tier do cliente na pergunta, ou o endpoint deve inferir do contexto da sessão? | Afeta RN-06 (Relógio de SLA) e respostas sobre incidente crítico | AC-08, AC-09 | Antes do Sprint 1 |
