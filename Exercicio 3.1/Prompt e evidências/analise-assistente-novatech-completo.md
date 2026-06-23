# Análise do Assistente de IA — NovaTech

**Projeto:** Assistente RAG de Atendimento ao Cliente  
**Empresa:** NovaTech × DB1 Group  
**Data:** 22/06/2026  
**Conteúdo:** Prompt do sistema · Avaliação do analista · Avaliação independente (Claude) · Comparação · Classificação de erros · Ajustes de produto · Prompts propostos

---

## Sumário

1. [Prompt do sistema e contexto do projeto](#i-prompt-do-sistema-e-contexto-do-projeto)
2. [Avaliação das 6 respostas — Analista de Negócios](#ii-avaliação-das-6-respostas--analista-de-negócios)
3. [Avaliação independente — Claude](#iii-avaliação-independente--claude)
4. [Comparação entre avaliadores e divergências](#iv-comparação-entre-avaliadores-e-divergências)
5. [Classificação de erros e ajustes de produto](#v-classificação-de-erros-e-ajustes-de-produto)
6. [Prompts de sistema propostos](#vi-prompts-de-sistema-propostos)

---

## I. Prompt do sistema e contexto do projeto

### 1.1 Perfil do analista configurado

```
Atue como um Analista de Negócios Sênior especializado em sistemas corporativos
de Educação e integrações com fornecedores.

Ao analisar requisitos:
- Identifique ambiguidades, inconsistências e riscos.
- Questione cenários não cobertos.
- Considere casos de borda (edge cases).
- Avalie impactos funcionais e técnicos.
- Sugira melhorias de regra de negócio quando apropriado.
- Não faça suposições sem explicitar as premissas.

Sempre que possível apresente:
Contexto | Regras de negócio | Fluxo de processamento | Casos de exceção |
Critérios de aceite | Cenários de teste | Escrita de PBIs

Estilo: objetivo e didático. Explique o raciocínio passo a passo.
Utilize tabelas quando facilitar o entendimento.
Para regras complexas, forneça exemplos práticos.
```

### 1.2 Cenário NovaTech

A NovaTech é uma empresa de médio porte do setor de logística com 1.200 funcionários. Documentação interna distribuída em três fontes:

- SharePoint corporativo com ~800 documentos (PDFs e Word)
- Wiki interna no Confluence com ~400 páginas
- Pasta de rede com planilhas de referência atualizadas mensalmente

A equipe de atendimento (45 pessoas) gastava em média **12 minutos por chamado** buscando informações para responder dúvidas de clientes. A NovaTech contratou a DB1 para construir um assistente de IA integrado ao ambiente Microsoft (Teams + SharePoint).

> **Meta:** Reduzir o tempo médio de busca de 12 para menos de 2 minutos por chamado.

**Volume:** 320 chamados/dia, dos quais ~60% envolvem consulta a documentação → ~192 chamados/dia impactados.

### 1.3 Base documental ingerida — Anexo A

| Documento | Descrição | Status | Confiabilidade |
|---|---|---|---|
| POL-001 v3.1 | Política de Devolução de Mercadorias | ✅ Vigente | Alta |
| SLA-2024 | Tabela de SLA por Tipo de Cliente | ✅ Vigente | Alta |
| PROC-042 v1 | Cálculo de Frete Especial (original) | ⚠️ Conflito | Baixa |
| PROC-042 v2 | Cálculo de Frete Especial (revisado) | ⚠️ Conflito | Média |
| FAQ-Atendimento | Perguntas Frequentes (informal) | ⚠️ Não validado | Baixa |

### 1.4 Respostas avaliadas

| # | Pergunta | Resposta do assistente | Fonte citada | Confiança declarada |
|---|---|---|---|---|
| 1 | "Qual o prazo de devolução para produtos standard?" | "O prazo é de 7 dias úteis após o recebimento. O cliente deve abrir chamado no portal e anexar fotos." | POL-001, seção 3.2 | Alta |
| 2 | "Meu cliente é Silver. Qual o prazo de resolução?" | "O cliente Silver tem resolução em até 48h." | SLA-2024 | Alta |
| 3 | "Posso devolver carga perigosa classe 3?" | "Não. Cargas perigosas (classes 1 a 6 da ANTT) não podem ser devolvidas pelo processo padrão. Recomendo escalar para o supervisor." | POL-001, seção 3.2 | Alta |
| 4 | "Qual a política para carga danificada durante transporte?" | "A política de danos prevê reembolso integral quando comprovada negligência da transportadora, mediante laudo técnico e fotos." | Nenhuma | Alta |
| 5 | "Qual o SLA do cliente Enterprise?" | "O tier Enterprise não foi encontrado na documentação. Os tiers documentados são Gold, Silver e Standard. Sugiro confirmar a classificação ou escalar ao supervisor." | — | Baixa |
| 6 | "Posso enviar carga perigosa com frete expresso?" | "Sim, cargas perigosas podem ser enviadas via frete expresso mediante autorização prévia do compliance e documentação ANTT atualizada." | FAQ-Atendimento, item 32 | Alta |

---

## II. Avaliação das 6 respostas — Analista de Negócios

**Avaliador:** Analista de Negócios Sênior  
**Fonte de verdade:** Anexo A — Documentação Simulada NovaTech

### Sumário

| # | Pergunta (resumo) | Veredicto | Severidade |
|---|---|---|---|
| 1 | Prazo de devolução standard | ⚠️ Parcialmente correta | Média |
| 2 | SLA Silver | ⚠️ Parcialmente correta | Alta |
| 3 | Devolução carga perigosa cl. 3 | ⚠️ Parcialmente correta | Média |
| 4 | Política carga danificada | ❌ Incorreta | **Crítica** |
| 5 | SLA Enterprise | ✅ Correta | — |
| 6 | Frete expresso carga perigosa | ❌ Incorreta | **Crítica** |

---

### Resposta 1 — Prazo de devolução para produtos standard

**Pergunta:** "Qual o prazo de devolução para produtos standard?"  
**Resposta:** "O prazo é de 7 dias úteis após o recebimento. O cliente deve abrir chamado no portal e anexar fotos."  
**Fonte citada:** POL-001, seção 3.2 | **Confiança declarada:** Alta

**Veredicto: ⚠️ Parcialmente correta**

**O que estava correto:**
- Prazo de 7 dias úteis (POL-001, seção 3.1 ✓)
- Orientação de abrir chamado no portal (POL-001, seção 3.3, item 1 ✓)

**Problemas identificados:**
- Citação de seção errada: o prazo está na **seção 3.1** (Prazo geral), não na 3.2 (que trata de exceções)
- Omissão do número do CT-e como campo obrigatório do chamado (POL-001, seção 3.3)
- "Anexar fotos" sem especificar: mínimo 3 fotos — embalagem externa, etiqueta de identificação e conteúdo
- Omissão da URL `portal.novatech.com.br` e da categoria de chamado "Devolução de Mercadoria"

**Impacto operacional:** Cliente abre chamado sem CT-e → chamado rejeitado na triagem → retrabalho e atraso.

---

### Resposta 2 — SLA Silver

**Pergunta:** "Meu cliente é Silver. Qual o prazo de resolução?"  
**Resposta:** "O cliente Silver tem resolução em até 48h."  
**Fonte citada:** SLA-2024 | **Confiança declarada:** Alta

**Veredicto: ⚠️ Parcialmente correta**

**O que estava correto:**
- 48h úteis para resolução de chamados gerais (SLA-2024, seção 2 ✓)

**Problemas identificados:**
- Omissão crítica: incidentes críticos Silver têm resolução em até **8h** (SLA-2024, seção 2)
- Omissão dos critérios que definem incidente crítico (SLA-2024, seção 3)
- Omissão da regra de pausa do relógio de SLA (SLA-2024, seção 5)

**Impacto operacional:** Atendente comunica 48h em situação crítica Silver quando o SLA correto é 8h → descumprimento contratual + penalidade.

---

### Resposta 3 — Devolução de carga perigosa classe 3

**Pergunta:** "Posso devolver carga perigosa classe 3?"  
**Resposta:** "Não. Cargas perigosas (classes 1 a 6 da ANTT) não podem ser devolvidas pelo processo padrão. Recomendo escalar para o supervisor."  
**Fonte citada:** POL-001, seção 3.2 | **Confiança declarada:** Alta

**Veredicto: ⚠️ Parcialmente correta**

**O que estava correto:**
- Carga perigosa classe 3 não é elegível pelo processo padrão (POL-001, seção 3.2 ✓)
- Classes 1 a 6 da ANTT listadas corretamente

**Problemas identificados:**
- Encaminhamento para "supervisor" não consta em nenhum documento
- O correto é: **Gestão de Riscos, ramal 4500**, para tratamento individual (POL-001, seção 3.2)
- Afirmar "não pode" sem mencionar o caminho alternativo equivale a desinformação por omissão

**Impacto operacional:** Cliente encaminhado à pessoa errada → atraso e possível tratamento inadequado de carga regulada pela ANTT.

---

### Resposta 4 — Política para carga danificada em transporte

**Pergunta:** "Qual a política para carga danificada durante transporte?"  
**Resposta:** "A política de danos prevê reembolso integral quando comprovada negligência da transportadora, mediante laudo técnico e fotos."  
**Fonte citada:** Nenhuma | **Confiança declarada:** Alta

**Veredicto: ❌ Incorreta — Severidade Crítica**

**Diagnóstico:**

Não existe documento formal (POL ou PROC) sobre carga danificada na base documental da NovaTech. A única referência disponível é o FAQ item 38, classificado explicitamente como informal e não validado por Compliance.

O assistente:
1. Fabricou conteúdo sem fonte rastreável
2. Introduziu o termo "negligência" — ausente em qualquer documento NovaTech
3. "Reembolso integral" não rastreável a nenhuma fonte
4. Declarou Confiança Alta sem nenhuma fonte normativa
5. Omitiu o prazo de 48h para registro da ocorrência (FAQ item 38)
6. Omitiu o encaminhamento para `sinistros@novatech.com.br` via Jurídico (FAQ item 38)

**Comportamento esperado:** Informar que não há política formalizada disponível; apresentar o FAQ com ressalva explícita; orientar escalação ao Jurídico.

**Impacto operacional:** Crítico. Informação incorreta sobre reembolso em transporte pode gerar implicação contratual e jurídica.

---

### Resposta 5 — SLA do cliente Enterprise

**Pergunta:** "Qual o SLA do cliente Enterprise?"  
**Resposta:** "O tier Enterprise não foi encontrado na documentação. Os tiers documentados são Gold, Silver e Standard. Sugiro confirmar a classificação ou escalar ao supervisor."  
**Fonte citada:** — | **Confiança declarada:** Baixa

**Veredicto: ✅ Correta** *(veredicto do analista — ver divergência na Seção IV)*

**Por que está correta:**
- Tier Enterprise não existe (SLA-2024, seção 1: apenas Gold, Silver e Standard)
- Recusou fabricar resposta
- Informou corretamente os tiers existentes
- Orientou confirmação ou escalação
- Confiança Baixa declarada é apropriada

**Ponto de melhoria:** O encaminhamento ideal seria ao **Comercial** para análise de viabilidade (SLA-2024, seção 1), não genericamente ao "supervisor".

---

### Resposta 6 — Envio de carga perigosa com frete expresso

**Pergunta:** "Posso enviar carga perigosa com frete expresso?"  
**Resposta:** "Sim, cargas perigosas podem ser enviadas via frete expresso mediante autorização prévia do compliance e documentação ANTT atualizada."  
**Fonte citada:** FAQ-Atendimento, item 32 | **Confiança declarada:** Alta

**Veredicto: ❌ Incorreta — Severidade Crítica**

**Problemas identificados:**

**Problema 1 — Fonte inadequada com confiança indevida:** O FAQ-Atendimento é classificado explicitamente como informal, não validado por Compliance ou Operações. Atribuir Confiança Alta contradiz a própria classificação da fonte.

**Problema 2 — Ausência de documento formal:** Não existe nenhum PROC ou POL que autorize frete expresso para carga perigosa. A PROC-043 (referenciada pela PROC-042 v2, seção 4) está em processo de revisão pelo Compliance.

**Problema 3 — Omissão seletiva:** O FAQ item 32 contém a ressalva "demora uns 2 dias para conseguir a autorização, então o 'expresso' acaba não sendo tão expresso" — informação descartada pelo assistente.

**Impacto operacional:** Crítico. Atendente cria expectativa que a operação pode não cumprir → risco regulatório ANTT.

---

### Padrão de falha identificado

As respostas 4 e 6 revelam o mesmo anti-padrão sistêmico: o assistente atribui Confiança Alta sempre que encontra conteúdo semanticamente relacionado, independente da qualidade da fonte.

**Regra de negócio ausente no design do assistente:**

| Tipo de fonte | Confiança máxima permitida |
|---|---|
| POL / PROC / SLA vigente e sem conflito | Alta |
| PROC com conflito de versão (ex: PROC-042) | Média — com sinalização do conflito |
| FAQ informal não validado | Baixa — com ressalva obrigatória |
| Ausência de documento formal (gap) | Nula — com orientação de escalação |

---

## III. Avaliação independente — Claude

**Metodologia:** Avaliação produzida após releitura integral e independente do Anexo A, sem consulta à avaliação anterior. Divergências identificadas a posteriori.

### Sumário

| # | Pergunta (resumo) | Veredicto Claude |
|---|---|---|
| 1 | Prazo de devolução standard | ⚠️ Parcialmente correta |
| 2 | SLA Silver | ⚠️ Parcialmente correta |
| 3 | Devolução carga perigosa cl. 3 | ⚠️ Parcialmente correta |
| 4 | Política carga danificada | ❌ Incorreta |
| 5 | SLA Enterprise | ⚠️ Parcialmente correta *(divergência)* |
| 6 | Frete expresso carga perigosa | ❌ Incorreta — com ressalva de fundamentação |

---

### Pontos adicionais identificados por Claude

#### Resposta 1 — ponto extra
A POL-001 seção 3.1 especifica que os 7 dias úteis são contados a partir da **data de recebimento confirmada no sistema de tracking** — não da data declarada pelo cliente. Em disputas de prazo, essa distinção é decisiva. O assistente diz apenas "após o recebimento", sem mencionar a referência ao tracking.

#### Resposta 2 — ponto extra
A SLA-2024 seção 5 não menciona Silver explicitamente na regra de não-pausa do relógio para incidentes críticos (mencionada apenas para Gold). Isso é um gap do próprio documento: a ausência de menção ao Silver pode ser interpretada como que o relógio *pausa* para incidentes críticos Silver — mais favorável ao cliente Silver e potencialmente contestável.

#### Resposta 3 — ponto extra
Afirmar "não pode" sem mencionar o caminho alternativo é **desinformação ativa por omissão** — não apenas encaminhamento incorreto. A POL-001 instrui encaminhamento ao Gestão de Riscos para "tratamento individual", o que implica que existe um processo alternativo.

#### Resposta 4 — ponto extra
O termo **"negligência"** tem implicação jurídica específica no direito civil brasileiro. A NovaTech usa "defeito ou erro da NovaTech" (POL-001, seção 3.5) — expressão mais ampla e sem carga jurídica. A substituição pode ser usada contra a empresa em litígio.

#### Resposta 5 — divergência de veredicto
Claude avalia como **⚠️ Parcialmente correta** (não Correta). O encaminhamento para "supervisor" em vez de "Comercial para análise de viabilidade" é uma falha rastreável: a SLA-2024 seção 1 é prescritiva sobre o destinatário. Uma resposta correta exige que tanto o conteúdo quanto o encaminhamento sejam rastreáveis à fonte.

#### Resposta 6 — ponto extra
O assistente fez **omissão seletiva**: recuperou a afirmação positiva do FAQ item 32 ("sim, é possível") e descartou a ressalva mais útil para o atendente ("demora ~2 dias para conseguir a autorização, então o 'expresso' acaba não sendo tão expresso"). Isso distorce a expectativa do cliente independente da questão da fonte.

---

## IV. Comparação entre avaliadores e divergências

### Quadro comparativo geral

| # | Pergunta (resumo) | Analista | Claude | Alinhamento |
|---|---|---|---|---|
| 1 | Prazo de devolução standard | ⚠️ Parcialmente correta | ⚠️ Parcialmente correta | ✅ Alinhados — divergência de ênfase |
| 2 | SLA Silver | ⚠️ Parcialmente correta | ⚠️ Parcialmente correta | ✅ Alinhados — divergência de ênfase |
| 3 | Devolução carga perigosa cl. 3 | ⚠️ Parcialmente correta | ⚠️ Parcialmente correta | ✅ Alinhados — divergência de ênfase |
| 4 | Política carga danificada | ❌ Incorreta | ❌ Incorreta | ✅ Alinhados — divergência de profundidade |
| 5 | SLA Enterprise | ✅ Correta | ⚠️ Parcialmente correta | ⚠️ **Divergência de veredicto** |
| 6 | Frete expresso carga perigosa | ❌ Incorreta | ❌ Incorreta | ⚠️ **Divergência de fundamentação** |

---

### Divergência 1 — Resposta 5: veredicto diferente

| Dimensão | Analista | Claude |
|---|---|---|
| Veredicto | ✅ Correta | ⚠️ Parcialmente correta |
| Critério | Acertou o conteúdo principal; encaminhamento é melhoria menor | Encaminhamento incorreto também é falha rastreável à fonte |
| Argumento | Comportamento de não fabricar conteúdo é exemplar e deve ser reconhecido | Resposta correta exige que conteúdo E encaminhamento sejam rastreáveis à documentação |

**Posição de Claude:** Uma resposta de atendimento tem dois componentes — a informação e a ação seguinte. Direcionar ao "supervisor" em vez do Comercial pode resultar em nenhuma análise de viabilidade sendo feita. O veredicto "parcialmente correta" preserva o reconhecimento do acerto principal sem ignorar a falha no encaminhamento.

---

### Divergência 2 — Resposta 6: fundamentação diferente

| Dimensão | Analista | Claude |
|---|---|---|
| Veredicto | ❌ Incorreta | ❌ Incorreta |
| Ênfase | Qualidade da fonte (FAQ informal com Confiança Alta) | Qualidade da fonte + omissão seletiva que distorce expectativa |
| Ponto adicional | — | Assistente descartou a ressalva prática do próprio FAQ ("não é realmente expresso") |

---

### Pontos adicionais de Claude não cobertos pelo analista

| Resposta | Ponto adicional |
|---|---|
| 1 | Referência de prazo é o tracking, não a declaração do cliente (POL-001 seção 3.1) |
| 1 | Omissão da URL do portal e da categoria de chamado (POL-001 seção 3.3) |
| 2 | Ambiguidade da regra de pausa do relógio para Silver em incidentes críticos (SLA-2024 seção 5) |
| 3 | "Não pode" sem caminho alternativo é desinformação ativa por omissão |
| 4 | Uso da palavra "negligência" introduz terminologia jurídica não presente em nenhum documento NovaTech |
| 6 | Omissão seletiva da ressalva prática do FAQ distorce expectativa do cliente |

---

### Concordâncias fundamentais

Apesar das divergências pontuais, ambas as avaliações chegaram às mesmas conclusões estruturais:

1. **Resposta 4 é a falha mais crítica** — gap documental + Confiança Alta + sem fonte
2. **Resposta 6 é crítica por risco regulatório** — fonte informal tratada como normativa para assunto com implicação ANTT
3. **O padrão sistêmico de falha é o mesmo:** Confiança Alta atribuída à existência de match semântico, não à qualidade da fonte
4. **Resposta 5 tem o melhor comportamento relativo** — mesmo com divergência de veredicto
5. **A regra de confiança por tipo de fonte está ausente** no design do assistente

---

### Recomendação conjunta — Critério de aceite proposto

> **Uma resposta é considerada "Correta" somente quando tanto o conteúdo informativo quanto o encaminhamento de ação são rastreáveis a pelo menos um documento formal da base. Respostas que acertam o conteúdo mas fabricam o encaminhamento devem ser classificadas como "Parcialmente corretas".**

---

## V. Classificação de erros e ajustes de produto

### 5.1 Taxonomia adotada

| Tipo de erro | Definição |
|---|---|
| **Alucinação** | O assistente gerou conteúdo factual não rastreável a nenhuma fonte da base — inventado |
| **Fonte não confiável** | O assistente recuperou conteúdo real de uma fonte, mas essa fonte não tem autoridade suficiente para o contexto (ex: FAQ informal tratado como normativo) |
| **Informação incompleta** | O assistente recuperou conteúdo correto, mas omitiu partes obrigatórias ou qualificações que mudam o significado prático da resposta |
| **Encaminhamento incorreto** | O assistente acertou o diagnóstico, mas direcionou para ação ou destinatário não documentado |

> Uma resposta pode ter mais de um tipo de erro simultaneamente.

---

### 5.2 Resposta 1 — Prazo de devolução standard

**Tipos de erro:**

| Tipo | Instância |
|---|---|
| Informação incompleta | Omissão do CT-e como campo obrigatório (POL-001, seção 3.3, item 2) |
| Informação incompleta | "Anexar fotos" sem especificar: mínimo 3, sendo embalagem externa, etiqueta e conteúdo |
| Informação incompleta | Prazo sem mencionar que a referência é o tracking, não a declaração do cliente |
| Informação incompleta | Omissão da URL do portal e da categoria de chamado |

**Causa raiz:** O chunking provavelmente separou a seção 3.1 (prazo) da seção 3.3 (procedimento). O modelo sumarizou de forma excessivamente compacta, perdendo campos obrigatórios.

**Ajustes de produto:**

**Pipeline — Chunking semântico:**
Criar chunks que preservem a unidade semântica "prazo + procedimento" como um par inseparável. Seções que referenciam explicitamente uma seção de procedimento devem ser ingeridas no mesmo chunk ou ter metadado de dependência.

**Prompt do sistema:**
```
Quando a resposta envolver um procedimento com etapas (abertura de chamado,
solicitação, devolução), inclua TODOS os campos e requisitos listados na fonte,
sem sumarizar. Se a fonte listar campos obrigatórios, reproduza a lista completa.
```

**Interface — Checklist:** Para perguntas do tipo "como fazer X", exibir a resposta em formato de checklist ao invés de prosa, com cada campo obrigatório como item verificável.

---

### 5.3 Resposta 2 — SLA Silver

**Tipos de erro:**

| Tipo | Instância |
|---|---|
| Informação incompleta | Omissão do regime de incidentes críticos: resolução Silver em até 8h (SLA-2024, seção 2) |
| Informação incompleta | Omissão dos critérios que definem incidente crítico (SLA-2024, seção 3) |
| Informação incompleta | Omissão da regra de pausa do relógio de SLA (SLA-2024, seção 5) |

**Causa raiz:** Queries genéricas como "qual o SLA do Silver" tendem a recuperar o valor mais simples — chamados gerais — porque tem maior densidade semântica para essa pergunta. O LLM não foi instruído a verificar se existem regimes alternativos.

**Ajustes de produto:**

**Prompt do sistema:**
```
Quando a resposta envolver SLA, prazo ou tempo de atendimento, verifique SEMPRE
se existem regimes diferenciados para o mesmo tier (ex: chamados gerais vs.
incidentes críticos). Se existirem, inclua ambos na resposta com a distinção
explícita. Nunca responda com um único valor de SLA sem qualificar o tipo de chamado.
```

**Pipeline — Metadados:** Marcar chunks da SLA-2024 com metadado `contém_regimes_multiplos`. O retriever deve, para documentos com esse metadado, buscar obrigatoriamente todos os chunks, não apenas o de maior similaridade.

**Interface — Badge:** Quando a resposta contiver valores de SLA, exibir alerta fixo: `⚠️ Verifique se é incidente crítico — SLAs diferem`.

---

### 5.4 Resposta 3 — Devolução carga perigosa classe 3

**Tipos de erro:**

| Tipo | Instância |
|---|---|
| Encaminhamento incorreto | Direcionamento para "supervisor" em vez de Gestão de Riscos, ramal 4500 (POL-001, seção 3.2) |
| Informação incompleta | Omissão de que existe processo alternativo ("tratamento individual") |

**Causa raiz:** O chunk provavelmente não incluía o final do parágrafo com a instrução de encaminhamento ao ramal 4500, ou o modelo generalizou "Gestão de Riscos" para "supervisor".

**Ajustes de produto:**

**Pipeline — Chunk de exceção sempre completo:**
Seções que definem inelegibilidade ou restrição devem obrigatoriamente incluir o parágrafo de encaminhamento que as segue. Nunca separar "não é elegível" de "para essas categorias, contatar X".

**Prompt do sistema:**
```
Quando a resposta indicar que algo não é possível pelo processo padrão, SEMPRE inclua
o encaminhamento alternativo documentado na fonte. Se não houver encaminhamento
documentado, informe explicitamente. Nunca invente encaminhamentos (supervisor,
gerente, etc.) que não constem na fonte.
```

**Interface — Campo de encaminhamento destacado:** Para respostas que contenham "não elegível" ou equivalentes, exibir o campo de encaminhamento em destaque visual separado do corpo da resposta.

---

### 5.5 Resposta 4 — Política carga danificada

**Tipos de erro:**

| Tipo | Instância | Severidade |
|---|---|---|
| Alucinação | "Reembolso integral quando comprovada negligência" — não consta em nenhum documento | 🔴 Crítica |
| Alucinação | Uso do termo jurídico "negligência" — ausente em toda a documentação NovaTech | 🔴 Crítica |
| Fonte não confiável | Conteúdo derivado do FAQ item 38 (informal) sem declaração da fonte | 🔴 Crítica |
| Informação incompleta | Omissão do prazo de 48h para registro (FAQ item 38) | 🟡 Alta |
| Informação incompleta | Omissão do encaminhamento sinistros@novatech.com.br via Jurídico (FAQ item 38) | 🟡 Alta |

**Causa raiz:** Gap documental. O LLM, ao não encontrar documento normativo específico, combinou fragmentos do FAQ com conhecimento paramétrico sobre "políticas de dano em transporte" — preenchendo o gap com informação plausível mas não verificável.

**Ajustes de produto:**

**Prompt do sistema — Comportamento em gap documental (regra crítica):**
```
Se não houver documento formal (POL, PROC ou SLA) que responda à pergunta:
1. Informe explicitamente: "Não encontrei documentação formal sobre este tema na base."
2. Se houver referência no FAQ, apresente com ressalva: "O FAQ interno menciona [X],
   mas este documento não foi validado por Compliance ou Operações."
3. Oriente escalação: "Recomendo verificar com [área responsável] antes de responder."
4. NÃO complemente com conhecimento próprio.
5. NÃO use termos jurídicos não presentes na fonte.
6. Mantenha confiança como Baixa ou Nula neste cenário.
```

**Pipeline — Bloqueio de Confiança Alta sem fonte normativa:**
Se o retriever não retornar nenhum chunk com `tipo: normativo` para a query, bloquear automaticamente a atribuição de Confiança Alta. A confiança máxima sem documento normativo é Baixa.

**Pipeline — Catálogo de gaps:**
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

### 5.6 Resposta 5 — SLA Enterprise

**Tipos de erro:**

| Tipo | Instância |
|---|---|
| Encaminhamento incorreto | Direcionamento para "supervisor" em vez de "Comercial para análise de viabilidade" (SLA-2024, seção 1) |

**Causa raiz:** A instrução de encaminhamento ao Comercial pode ter sido truncada ou estar em posição de baixa relevância no chunk. O LLM preencheu com valor genérico.

**Ajustes de produto:**

**Prompt do sistema:**
```
Nunca use termos genéricos de encaminhamento como "supervisor", "gerente" ou "sua equipe"
sem que esses destinatários constem explicitamente na fonte recuperada.
Se o documento especifica um destinatário (ex: "encaminhar ao Comercial"), use exatamente esse.
```

**Pipeline:** Toda instrução de encaminhamento deve ser ingerida no mesmo chunk da regra que a dispara.

---

### 5.7 Resposta 6 — Frete expresso carga perigosa

**Tipos de erro:**

| Tipo | Instância | Severidade |
|---|---|---|
| Fonte não confiável | FAQ informal tratado com Confiança Alta | 🔴 Crítica |
| Informação incompleta | Omissão da ressalva: "demora ~2 dias, não é realmente expresso" (FAQ item 32) | 🟡 Alta |
| Informação incompleta | Omissão de que não existe documento formal que suporte a prática | 🟡 Alta |

**Causa raiz:** O pipeline não distingue confiabilidade das fontes ao montar o contexto — chunks do FAQ têm o mesmo peso semântico que chunks da POL-001. O LLM selecionou a afirmação positiva e descartou o qualificador prático.

**Ajustes de produto:**

**Pipeline — Metadado de confiabilidade de fonte:**
```
POL, PROC, SLA vigente e sem conflito  → confiabilidade: ALTA
PROC com conflito de versão             → confiabilidade: MÉDIA
FAQ informal                            → confiabilidade: BAIXA
Ausência de documento (gap)             → confiabilidade: NULA
```

**Prompt do sistema:**
```
A confiança da sua resposta deve refletir a fonte de menor confiabilidade recuperada.
- Fonte normativa vigente sem conflito      → confiança: Alta
- Fonte normativa com conflito de versão   → confiança: Média. Sinalize o conflito.
- Fonte informal (FAQ)                     → confiança: Baixa.
  Inclua a ressalva: "Esta informação consta no FAQ interno, que não foi validado
  por Compliance ou Operações."
- Nenhuma fonte normativa                  → confiança: Nula. Informe o gap.

Nunca eleve a confiança acima do nível permitido pela fonte.
```

**Prompt do sistema — Preservação de ressalvas:**
```
Ao sumarizar conteúdo de uma fonte, NÃO descarte qualificadores, ressalvas ou
exceções presentes na fonte original. Se a fonte diz "sim, mas com a ressalva X",
a resposta deve incluir X. Omitir ressalvas distorce a expectativa do usuário
mesmo quando o conteúdo principal está correto.
```

**Interface — Badge:** Quando a resposta for fundamentada em FAQ ou documento sem responsável formal, exibir permanentemente: `⚠️ Baseado em documento informal — confirme com Compliance antes de responder ao cliente`.

---

### 5.8 Consolidado — mapa de erros × ajuste × prioridade

| # | Tipo(s) de erro | Camada | Ajuste principal | Prioridade |
|---|---|---|---|---|
| 1 | Informação incompleta | Pipeline + Prompt | Chunk semântico par prazo+procedimento; prompt exige completude de campos obrigatórios | P1 |
| 2 | Informação incompleta | Prompt + Pipeline + Interface | Prompt força verificação de regimes múltiplos; badge de alerta de regime crítico na UI | P2 |
| 3 | Encaminhamento incorreto | Pipeline + Prompt | Chunk preserva par restrição+encaminhamento; prompt proíbe encaminhamentos genéricos | P1 |
| 4 | Alucinação + Fonte não confiável + Incompleta | Prompt + Pipeline | Regra de gap documental; bloqueio de Confiança Alta sem fonte normativa; catálogo de gaps | **P0** |
| 5 | Encaminhamento incorreto | Prompt + Pipeline | Proibição de destinatários genéricos; chunk preserva par regra+encaminhamento | P1 |
| 6 | Fonte não confiável + Incompleta | Pipeline + Prompt + Interface | Metadado de confiabilidade; regra de confiança derivada; preservação de ressalvas; badge UI | **P0** |

---

### 5.9 Nota sobre interdependência dos ajustes

Os ajustes não são independentes. A cadeia de confiabilidade precisa funcionar de forma coerente entre as três camadas:

```
Pipeline (classifica a fonte ao ingerir)
    → Prompt (instrui o LLM a derivar confiança da fonte)
        → Interface (exibe o nível de confiança e ressalvas ao atendente)
```

Implementar apenas o ajuste de prompt sem o metadado no pipeline é insuficiente: o LLM não tem como saber que o FAQ é menos confiável que a POL-001 se essa informação não chegar junto com o chunk. Os três precisam funcionar em conjunto.

---

## VI. Prompts de sistema propostos

### Prompt 1 — Confiança derivada da fonte

```
A confiança da sua resposta deve refletir a fonte de menor confiabilidade
entre as fontes recuperadas:

- Fonte normativa vigente sem conflito      → confiança: Alta
- Fonte normativa com conflito de versão   → confiança: Média. Sinalize o conflito.
- Fonte informal (FAQ)                     → confiança: Baixa.
  Inclua a ressalva: "Esta informação consta no FAQ interno, que não foi
  validado por Compliance ou Operações."
- Nenhuma fonte normativa (gap documental) → confiança: Nula.
  Informe o gap e oriente escalação.

Nunca eleve a confiança acima do nível permitido pela fonte.
```

---

### Prompt 2 — Comportamento em gap documental

```
Se não houver documento formal (POL, PROC ou SLA) que responda à pergunta:

1. Informe explicitamente: "Não encontrei documentação formal sobre este
   tema na base."
2. Se houver referência no FAQ, apresente com ressalva: "O FAQ interno
   menciona [X], mas este documento não foi validado por Compliance ou
   Operações."
3. Oriente escalação: "Recomendo verificar com [área responsável] antes
   de responder ao cliente."
4. NÃO complemente com conhecimento próprio.
5. NÃO use termos jurídicos não presentes na fonte.
6. Mantenha confiança como Baixa ou Nula neste cenário.
```

---

### Prompt 3 — Completude de procedimento

```
Quando a resposta envolver um procedimento com etapas (abertura de chamado,
solicitação, devolução), inclua TODOS os campos e requisitos listados na fonte,
sem sumarizar.

Se a fonte listar campos obrigatórios, reproduza a lista completa.
Nunca simplifique "mínimo 3 fotos: embalagem, etiqueta, conteúdo" para
"anexar fotos".
```

---

### Prompt 4 — Encaminhamentos documentados

```
Quando a resposta indicar que algo não é possível pelo processo padrão,
SEMPRE inclua o encaminhamento alternativo documentado na fonte.

Se não houver encaminhamento documentado, informe explicitamente e oriente
o atendente a verificar com seu gestor.

Nunca invente encaminhamentos (supervisor, gerente, etc.) que não
constem na fonte recuperada.

Se a fonte especifica um destinatário exato (ex: "Comercial para análise
de viabilidade"), use exatamente esse — não generalize.
```

---

### Prompt 5 — Verificação de regimes múltiplos

```
Quando a resposta envolver SLA, prazo ou tempo de atendimento, verifique
SEMPRE se existem regimes diferenciados para o mesmo tier
(ex: chamados gerais vs. incidentes críticos).

Se existirem, inclua ambos na resposta com a distinção explícita.

Nunca responda com um único valor de SLA sem qualificar o tipo de chamado.
```

---

### Prompt 6 — Preservação de ressalvas

```
Ao sumarizar conteúdo de uma fonte, NÃO descarte qualificadores, ressalvas
ou exceções presentes na fonte original.

Se a fonte diz "sim, mas com a ressalva X", a resposta DEVE incluir X.

Omitir ressalvas distorce a expectativa do usuário mesmo quando o conteúdo
principal está correto.
```

---

## Referências documentais

| Documento | Versão | Responsável |
|---|---|---|
| POL-001 — Política de Devolução de Mercadorias | 3.1 | Diretoria de Operações |
| PROC-042 — Cálculo de Frete Especial | 1.0 | Diretoria Comercial |
| PROC-042-v2 — Cálculo de Frete Especial (Revisado) | 2.0 | Diretoria Comercial |
| SLA-2024 — Tabela de SLA por Tipo de Cliente | 2024.1 | Comercial + Operações |
| FAQ-Atendimento — Perguntas Frequentes | Não controlada | Nenhum (informal) |

---

*Documento gerado em 22/06/2026 · NovaTech × DB1 Group · Confidencial*
