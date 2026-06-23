# Classificação de Erros e Ajustes de Produto — Assistente NovaTech

**Escopo:** Respostas com problema identificado nas avaliações anteriores (respostas 1, 2, 3, 4, 5 e 6)  
**Taxonomia de erros adotada:**

| Tipo | Definição |
|---|---|
| **Alucinação** | O assistente gerou conteúdo factual não rastreável a nenhuma fonte da base — inventado |
| **Fonte não confiável** | O assistente recuperou conteúdo real de uma fonte, mas essa fonte não tem autoridade suficiente para o contexto (ex: FAQ informal tratado como normativo) |
| **Informação incompleta** | O assistente recuperou conteúdo correto, mas omitiu partes obrigatórias ou qualificações que mudam o significado prático da resposta |
| **Encaminhamento incorreto** | O assistente acertou o diagnóstico, mas direcionou para ação ou destinatário não documentado |

> Uma resposta pode ter mais de um tipo de erro simultaneamente.

---

## Resposta 1 — Prazo de devolução para produtos standard

**Resposta do assistente:** "O prazo é de 7 dias úteis após o recebimento. O cliente deve abrir chamado no portal e anexar fotos."

### Tipos de erro

| Tipo | Instância |
|---|---|
| Informação incompleta | Omissão do CT-e como campo obrigatório do chamado (POL-001, seção 3.3, item 2) |
| Informação incompleta | "Anexar fotos" sem especificar: mínimo 3, sendo embalagem externa, etiqueta e conteúdo (POL-001, seção 3.3, item 2) |
| Informação incompleta | Prazo contado "após o recebimento" sem mencionar que a referência é o tracking, não a declaração do cliente (POL-001, seção 3.1) |
| Informação incompleta | Omissão da URL portal.novatech.com.br e da categoria de chamado "Devolução de Mercadoria" (POL-001, seção 3.3, item 1) |

> Não há alucinação: tudo que o assistente disse está correto. O problema é o que foi omitido.

### Causa raiz

O chunking do pipeline RAG provavelmente separou a seção 3.1 (prazo) da seção 3.3 (procedimento). O chunk recuperado continha o prazo mas não o detalhamento do procedimento completo — ou o LLM sumarizou a seção 3.3 de forma excessivamente compacta, perdendo campos obrigatórios.

### Ajuste de produto

**Pipeline — Chunking e recuperação:**
Criar chunks que preservem a unidade semântica "prazo + procedimento" como um par inseparável. Regra: seções de política que referenciam explicitamente uma seção de procedimento (ex: "siga o procedimento da seção 3.3") devem ser ingeridas no mesmo chunk ou ter metadado de dependência que force o retriever a buscar ambos.

**Prompt do sistema — Completude de procedimento:**
Adicionar instrução explícita:

```
Quando a resposta envolver um procedimento com etapas (abertura de chamado, solicitação, devolução),
inclua TODOS os campos e requisitos listados na fonte, sem sumarizar.
Se a fonte listar campos obrigatórios, reproduza a lista completa.
```

**Interface — Checklist de procedimento:**
Para perguntas do tipo "como fazer X", exibir a resposta em formato de checklist ao invés de prosa, com cada campo obrigatório como item verificável. Reduz o risco de o atendente "pular" campos ao repassar para o cliente.

---

## Resposta 2 — SLA Silver

**Resposta do assistente:** "O cliente Silver tem resolução em até 48h."

### Tipos de erro

| Tipo | Instância |
|---|---|
| Informação incompleta | Omissão do regime de incidentes críticos: resolução Silver em até 8h (SLA-2024, seção 2) |
| Informação incompleta | Omissão dos critérios que definem incidente crítico (SLA-2024, seção 3) |
| Informação incompleta | Omissão da regra de pausa do relógio de SLA (SLA-2024, seção 5) |

> Não há alucinação. A informação "48h" está correta para chamados gerais. O problema é a ausência de qualificação de contexto.

### Causa raiz

A SLA-2024 tem uma estrutura de tabela com duas linhas para o mesmo tier (chamados gerais e incidentes críticos). Queries genéricas como "qual o SLA do Silver" tendem a recuperar o valor mais simples e direto — o de chamados gerais — porque o texto "resolução Silver 48h" tem maior densidade semântica para essa pergunta do que "resolução Silver incidente crítico 8h". O LLM não foi instruído a verificar se existem regimes alternativos para o mesmo tier.

### Ajuste de produto

**Prompt do sistema — Verificação de regimes alternativos:**
```
Quando a resposta envolver SLA, prazo ou tempo de atendimento, verifique SEMPRE se existem
regimes diferenciados para o mesmo tier (ex: chamados gerais vs. incidentes críticos).
Se existirem, inclua ambos na resposta com a distinção explícita.
Nunca responda com um único valor de SLA sem qualificar o tipo de chamado.
```

**Pipeline — Metadados de documento:**
Marcar chunks da SLA-2024 com metadado `tipo: contratual` e `alerta: contém_regimes_multiplos`. O retriever deve, para documentos com esse metadado, buscar obrigatoriamente todos os chunks do documento, não apenas o de maior similaridade.

**Interface — Alerta de regime:**
Quando a resposta contiver valores de SLA, exibir um badge fixo na UI: `⚠️ Verifique se é incidente crítico — SLAs diferem`. Isso funciona como safety net mesmo que o assistente omita a qualificação.

---

## Resposta 3 — Devolução de carga perigosa classe 3

**Resposta do assistente:** "Não. Cargas perigosas (classes 1 a 6 da ANTT) não podem ser devolvidas pelo processo padrão. Recomendo escalar para o supervisor."

### Tipos de erro

| Tipo | Instância |
|---|---|
| Encaminhamento incorreto | Direcionamento para "supervisor" em vez de Gestão de Riscos, ramal 4500 (POL-001, seção 3.2) |
| Informação incompleta | Omissão de que existe processo alternativo ("tratamento individual"), criando expectativa zero para o cliente |

> Não há alucinação no conteúdo informativo. O erro está exclusivamente no encaminhamento.

### Causa raiz

O LLM recuperou corretamente a regra de inelegibilidade (seção 3.2), mas o chunk provavelmente não incluía o final do parágrafo com a instrução de encaminhamento ao ramal 4500. Alternativamente, o modelo interpretou "Gestão de Riscos" como um destinatário interno e generalizou para "supervisor" — comportamento de alucinação parcial no campo de encaminhamento.

### Ajuste de produto

**Pipeline — Chunk de exceção sempre completo:**
Regra de chunking: seções que definem inelegibilidade ou restrição devem obrigatoriamente incluir o parágrafo de encaminhamento que as segue. Nunca separar "não é elegível" de "para essas categorias, contatar X".

**Prompt do sistema — Encaminhamentos documentados:**
```
Quando a resposta indicar que algo não é possível pelo processo padrão, SEMPRE inclua
o encaminhamento alternativo documentado na fonte. Se não houver encaminhamento documentado,
informe explicitamente que não há instrução formal e oriente o atendente a verificar com
seu gestor imediato.
Nunca invente encaminhamentos (supervisor, gerente, etc.) que não constem na fonte.
```

**Interface — Campo de encaminhamento destacado:**
Para respostas que contenham a palavra "não elegível", "não é possível" ou equivalentes, a interface deve exibir o campo de encaminhamento em destaque visual separado do corpo da resposta. Isso força o atendente a ler o próximo passo antes de responder ao cliente.

---

## Resposta 4 — Política para carga danificada em transporte

**Resposta do assistente:** "A política de danos prevê reembolso integral quando comprovada negligência da transportadora, mediante laudo técnico e fotos." (Fonte: Nenhuma | Confiança: Alta)

### Tipos de erro

| Tipo | Instância | Severidade |
|---|---|---|
| Alucinação | "Reembolso integral quando comprovada negligência" — não consta em nenhum documento da base | 🔴 Crítica |
| Alucinação | Uso do termo jurídico "negligência" — ausente em toda a documentação NovaTech | 🔴 Crítica |
| Fonte não confiável | Conteúdo parcialmente derivado do FAQ item 38 (informal, não validado), sem declaração da fonte | 🔴 Crítica |
| Informação incompleta | Omissão do prazo de 48h para registro (FAQ item 38) | 🟡 Alta |
| Informação incompleta | Omissão do encaminhamento para sinistros@novatech.com.br via Jurídico (FAQ item 38) | 🟡 Alta |

> Este é o único caso com alucinação confirmada: o assistente gerou conteúdo factual novo ("negligência", "reembolso integral") que não é derivável de nenhuma fonte da base — nem mesmo do FAQ informal.

### Causa raiz

Gap documental: não há POL ou PROC formal sobre carga danificada. O LLM, ao não encontrar um documento normativo específico, combinou fragmentos do FAQ item 38 com conhecimento paramétrico próprio sobre "políticas de dano em transporte" — preenchendo o gap com informação plausível mas não verificável. Esse é o comportamento padrão de modelos não instruídos explicitamente sobre como agir em gaps documentais.

### Ajuste de produto

**Prompt do sistema — Comportamento em gap documental (regra crítica):**
```
Se não houver documento formal (POL, PROC ou SLA) que responda à pergunta:
1. Informe explicitamente: "Não encontrei documentação formal sobre este tema na base."
2. Se houver referência no FAQ, apresente-a com a ressalva: "O FAQ interno menciona [X],
   mas este documento não foi validado por Compliance ou Operações."
3. Oriente escalação: "Recomendo verificar com [área responsável] antes de responder ao cliente."
4. NÃO complemente com conhecimento próprio. NÃO use termos jurídicos não presentes na fonte.
5. Mantenha confiança como Baixa ou Nula neste cenário.
```

**Pipeline — Detecção de gap e bloqueio de confiança Alta:**
Implementar verificação pré-resposta: se o retriever não retornar nenhum chunk com `tipo: normativo` (POL/PROC/SLA) para a query, bloquear automaticamente a atribuição de confiança Alta. A confiança máxima permitida sem documento normativo é Baixa.

**Pipeline — Catálogo de gaps conhecido:**
Manter um arquivo de configuração `gaps-documentais.json` com tópicos sem cobertura formal (carga danificada, seguro de carga, frete padrão, escalação Gestão de Riscos). Para queries que mapeiem para esses tópicos, o assistente deve acionar o fluxo de gap antes mesmo do retriever.

```json
{
  "gaps": [
    {
      "topico": "carga danificada em transporte",
      "status": "sem_documento_formal",
      "referencia_informal": "FAQ-Atendimento, item 38",
      "escalacao": "sinistros@novatech.com.br (Jurídico)",
      "responsavel_formalizacao": "Operações"
    }
  ]
}
```

---

## Resposta 5 — SLA do cliente Enterprise

**Resposta do assistente:** "O tier Enterprise não foi encontrado na documentação. Os tiers documentados são Gold, Silver e Standard. Sugiro confirmar a classificação ou escalar ao supervisor."

### Tipos de erro

| Tipo | Instância |
|---|---|
| Encaminhamento incorreto | Direcionamento para "supervisor" em vez de "Comercial para análise de viabilidade" (SLA-2024, seção 1) |

> O conteúdo informativo está correto. O único erro é o encaminhamento não documentado.

### Causa raiz

O chunk que contém "Não existem outros tiers além dos três listados acima" (SLA-2024, seção 1) provavelmente estava no contexto, mas a instrução de encaminhamento ao Comercial pode ter sido truncada ou estar em posição de baixa relevância no chunk. O LLM preencheu o encaminhamento com um valor genérico ("supervisor") — comportamento de alucinação de baixo risco, mas rastreável.

### Ajuste de produto

**Prompt do sistema — Proibição de encaminhamentos genéricos:**
```
Nunca use termos genéricos de encaminhamento como "supervisor", "gerente" ou "sua equipe"
sem que esses destinatários constem explicitamente na fonte recuperada.
Se o documento especifica um destinatário (ex: "encaminhar ao Comercial"), use exatamente esse.
Se não há destinatário documentado, diga: "Não há instrução de encaminhamento na documentação.
Consulte seu gestor imediato."
```

**Pipeline — Chunk de encaminhamento como par obrigatório:**
Toda instrução do tipo "encaminhar para análise de viabilidade" deve ser ingerida no mesmo chunk da regra que a dispara. Nunca separar regra de negócio de seu encaminhamento correspondente.

---

## Resposta 6 — Envio de carga perigosa com frete expresso

**Resposta do assistente:** "Sim, cargas perigosas podem ser enviadas via frete expresso mediante autorização prévia do compliance e documentação ANTT atualizada." (Fonte: FAQ item 32 | Confiança: Alta)

### Tipos de erro

| Tipo | Instância | Severidade |
|---|---|---|
| Fonte não confiável | FAQ informal (não validado por Compliance) tratado com confiança Alta — equivalente a documento normativo | 🔴 Crítica |
| Informação incompleta | Omissão seletiva: descartou a ressalva prática do próprio FAQ ("demora ~2 dias, não é realmente expresso") | 🟡 Alta |
| Informação incompleta | Omissão de que não existe documento formal (PROC/POL) que suporte essa prática | 🟡 Alta |

> Não há alucinação estrita: o conteúdo veio do FAQ. O problema é a combinação de fonte não confiável + confiança indevida + omissão seletiva de ressalva crítica para o cliente.

### Causa raiz

Dois fatores combinados: (1) o pipeline não distingue a confiabilidade das fontes ao montar o contexto — chunks do FAQ têm o mesmo peso semântico que chunks da POL-001; (2) o LLM, ao sumarizar o FAQ item 32, selecionou a afirmação central positiva ("sim, é possível") e descartou o qualificador prático ("mas não é realmente expresso"), que é exatamente a informação mais útil para o atendente gerenciar a expectativa do cliente.

### Ajuste de produto

**Pipeline — Metadado de confiabilidade de fonte:**
Classificar cada documento ingerido com um nível de confiabilidade no momento da ingestão:

```
POL, PROC, SLA vigente e sem conflito  → confiabilidade: ALTA
PROC com conflito de versão             → confiabilidade: MÉDIA
FAQ informal                            → confiabilidade: BAIXA
Ausência de documento (gap)             → confiabilidade: NULA
```

Esse metadado deve ser passado ao LLM junto com o chunk, e o LLM deve ser instruído a declarar a confiança da resposta com base na fonte de menor confiabilidade recuperada.

**Prompt do sistema — Regra de confiança derivada da fonte:**
```
A confiança da sua resposta deve refletir a fonte de menor confiabilidade entre as fontes recuperadas.
- Fonte normativa vigente sem conflito → declare confiança: Alta
- Fonte normativa com conflito de versão → declare confiança: Média. Sinalize o conflito.
- Fonte informal (FAQ) → declare confiança: Baixa. Inclua a ressalva:
  "Esta informação consta no FAQ interno, que não foi validado por Compliance ou Operações."
- Nenhuma fonte normativa → declare confiança: Nula. Informe o gap e oriente escalação.

Nunca eleve a confiança acima do nível permitido pela fonte.
```

**Prompt do sistema — Preservação de ressalvas:**
```
Ao sumarizar conteúdo de uma fonte, NÃO descarte qualificadores, ressalvas ou exceções presentes
na fonte original. Se a fonte diz "sim, mas com a ressalva X", a resposta deve incluir X.
Omitir ressalvas distorce a expectativa do usuário mesmo quando o conteúdo principal está correto.
```

**Interface — Badge de fonte não confiável:**
Quando a resposta for fundamentada total ou parcialmente em FAQ ou documento sem responsável formal, exibir badge permanente na resposta: `⚠️ Baseado em documento informal — confirme com Compliance antes de responder ao cliente`. Não deixar isso como responsabilidade apenas do LLM declarar; tornar visível na camada de interface.

---

## Consolidado: mapa de erros × ajuste de produto

| # | Tipo(s) de erro | Camada do ajuste | Ajuste principal |
|---|---|---|---|
| 1 | Informação incompleta | Pipeline + Prompt | Chunk semântico preserva par "prazo + procedimento"; prompt exige completude de campos obrigatórios |
| 2 | Informação incompleta | Prompt + Pipeline + Interface | Prompt força verificação de regimes múltiplos; badge de alerta de regime crítico na UI |
| 3 | Encaminhamento incorreto | Pipeline + Prompt | Chunk preserva par "restrição + encaminhamento"; prompt proíbe encaminhamentos genéricos |
| 4 | Alucinação + Fonte não confiável + Informação incompleta | Prompt + Pipeline | Regra de gap documental no prompt; bloqueio de confiança Alta sem fonte normativa; catálogo de gaps |
| 5 | Encaminhamento incorreto | Prompt + Pipeline | Proibição de destinatários genéricos; chunk preserva par "regra + encaminhamento" |
| 6 | Fonte não confiável + Informação incompleta | Pipeline + Prompt + Interface | Metadado de confiabilidade por fonte; regra de confiança derivada; preservação de ressalvas; badge de UI |

---

## Priorização dos ajustes

### P0 — Bloqueio de alucinação em gap documental (resposta 4)
**Impacto:** Crítico — único caso com geração de conteúdo jurídico inexistente  
**Esforço:** Médio — requer regra de prompt + metadado no pipeline + catálogo de gaps  
**Ação:** Implementar antes do go-live. Testar com queries sobre carga danificada, seguro de carga e frete padrão.

### P0 — Metadado de confiabilidade de fonte (resposta 6)
**Impacto:** Crítico — FAQ informal com confiança Alta é o padrão de falha mais recorrente  
**Esforço:** Médio — requer classificação dos documentos na ingestão e instrução de prompt  
**Ação:** Implementar na ingestão do pipeline. Todos os 5 documentos devem ser classificados antes do primeiro deploy.

### P1 — Regra de completude de procedimento (respostas 1 e 3)
**Impacto:** Alto — campos obrigatórios omitidos geram retrabalho operacional  
**Esforço:** Baixo — ajuste de prompt e revisão da estratégia de chunking  
**Ação:** Implementar nas primeiras iterações de testes.

### P1 — Proibição de encaminhamentos genéricos (respostas 3 e 5)
**Impacto:** Alto — atendente executa ação errada mesmo tendo recebido informação correta  
**Esforço:** Baixo — ajuste de prompt  
**Ação:** Implementar junto com P1 de completude.

### P2 — Verificação de regimes múltiplos para SLA (resposta 2)
**Impacto:** Médio-alto — erro potencial em incidentes críticos, mas não ocorre em chamados comuns  
**Esforço:** Baixo — ajuste de prompt + badge de UI  
**Ação:** Implementar no primeiro ciclo de refinamento pós-go-live.

---

## Nota sobre a relação entre os ajustes

Os ajustes não são independentes. A cadeia de confiabilidade precisa ser implementada de forma coerente entre as três camadas:

```
Pipeline (classifica a fonte ao ingerir)
    → Prompt (instrui o LLM a derivar confiança da fonte)
        → Interface (exibe o nível de confiança e ressalvas ao atendente)
```

Implementar apenas o ajuste de prompt sem o metadado no pipeline é insuficiente: o LLM não tem como saber que o FAQ é menos confiável que a POL-001 se essa informação não chegar junto com o chunk. Implementar apenas o metadado sem a instrução de prompt é igualmente insuficiente: o modelo pode receber o metadado e não saber o que fazer com ele. Os três precisam funcionar em conjunto.
