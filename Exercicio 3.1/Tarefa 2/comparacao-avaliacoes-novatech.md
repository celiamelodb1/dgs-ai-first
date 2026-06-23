# Comparação de Avaliações — Respostas do Assistente NovaTech

**Avaliador A:** Analista de Negócios Sênior (avaliação anterior)  
**Avaliador B:** Claude (avaliação independente, baseada em releitura integral do Anexo A)  
**Data:** 22/06/2026  
**Metodologia:** Avaliação B foi produzida sem consulta à avaliação A. Divergências identificadas a posteriori.

---

## Quadro comparativo geral

| # | Pergunta (resumo) | Veredicto — Analista | Veredicto — Claude | Alinhamento |
|---|---|---|---|---|
| 1 | Prazo de devolução standard | ⚠️ Parcialmente correta | ⚠️ Parcialmente correta | ✅ Alinhados — divergência de ênfase |
| 2 | SLA Silver | ⚠️ Parcialmente correta | ⚠️ Parcialmente correta | ✅ Alinhados — divergência de ênfase |
| 3 | Devolução carga perigosa cl. 3 | ⚠️ Parcialmente correta | ⚠️ Parcialmente correta | ✅ Alinhados — divergência de ênfase |
| 4 | Política carga danificada | ❌ Incorreta | ❌ Incorreta | ✅ Alinhados — divergência de profundidade |
| 5 | SLA Enterprise | ✅ Correta | ⚠️ Parcialmente correta | ⚠️ **Divergência de veredicto** |
| 6 | Frete expresso carga perigosa | ❌ Incorreta | ❌ Incorreta — com ressalva | ⚠️ **Divergência de fundamentação** |

---

## Análise detalhada por resposta

---

### Resposta 1 — Prazo de devolução para produtos standard

**Resposta do assistente:** "O prazo é de 7 dias úteis após o recebimento. O cliente deve abrir chamado no portal e anexar fotos."

#### Avaliador A — Analista
- Veredicto: ⚠️ Parcialmente correta
- Foco: omissão do CT-e e especificação das 3 fotos; citação de seção errada (3.2 em vez de 3.1)

#### Avaliador B — Claude
- Veredicto: ⚠️ Parcialmente correta
- Foco: mesmo diagnóstico de omissão do CT-e e seção errada

**Ponto adicional identificado por Claude não coberto pelo analista:**

A POL-001 seção 3.1 especifica que os 7 dias úteis são contados a partir da **data de recebimento confirmada no sistema de tracking** — não da data que o cliente alega ter recebido. Essa distinção é relevante em disputas de prazo. A resposta do assistente diz apenas "após o recebimento", sem mencionar que a referência é o tracking, não a declaração do cliente. Em contexto de atendimento, essa omissão pode gerar conflito.

A POL-001 seção 3.3 também menciona que o chamado deve ser aberto em portal.novatech.com.br, selecionando a categoria "Devolução de Mercadoria" — detalhe de navegação omitido pelo assistente que pode gerar dificuldade para o cliente.

**Conclusão:** Alinhamento no veredicto. Claude identificou duas omissões adicionais de menor criticidade.

---

### Resposta 2 — SLA Silver

**Resposta do assistente:** "O cliente Silver tem resolução em até 48h."

#### Avaliador A — Analista
- Veredicto: ⚠️ Parcialmente correta
- Foco: omissão do regime de incidentes críticos (8h para Silver); menção ao gap da seção 5 sobre pausa do relógio

#### Avaliador B — Claude
- Veredicto: ⚠️ Parcialmente correta
- Foco: mesmo diagnóstico sobre incidentes críticos

**Ponto adicional identificado por Claude:**

A SLA-2024 seção 5 é explícita: o relógio pausa fora do horário comercial **para chamados gerais**, mas **não pausa para incidentes críticos de clientes Gold**. O analista identificou corretamente que a seção 5 não menciona Silver nessa regra — o que é um gap do documento. Claude concorda que isso é um gap real, mas vai além: a ausência de menção ao Silver na regra de não-pausa pode ser interpretada como que o relógio *pausa* para incidentes críticos Silver. Isso é mais favorável ao cliente Silver do que ao Gold e pode ser contestado contratualmente. O assistente não sinalizou nenhuma dessas nuances.

**Ponto de discordância menor:** O analista afirmou que o impacto é alto "nos casos críticos onde o atendente pode comunicar 48h quando o correto seria 8h". Claude avalia que a pergunta foi genérica ("qual o prazo de resolução?") e não mencionou incidente crítico — portanto, a resposta de 48h seria correta para o contexto perguntado, mas incompleta por não qualificar os cenários. A criticidade do problema é real, mas sua origem está na ausência de qualificação, não numa resposta errada para a pergunta exata.

**Conclusão:** Alinhamento no veredicto e diagnóstico principal. Claude identifica nuance adicional sobre a regra de pausa do relógio para Silver em incidentes críticos.

---

### Resposta 3 — Devolução de carga perigosa classe 3

**Resposta do assistente:** "Não. Cargas perigosas (classes 1 a 6 da ANTT) não podem ser devolvidas pelo processo padrão. Recomendo escalar para o supervisor."

#### Avaliador A — Analista
- Veredicto: ⚠️ Parcialmente correta
- Foco: encaminhamento para "supervisor" em vez de Gestão de Riscos ramal 4500

#### Avaliador B — Claude
- Veredicto: ⚠️ Parcialmente correta
- Foco: mesmo diagnóstico

**Ponto adicional identificado por Claude:**

O analista abordou o encaminhamento incorreto e mencionou brevemente o FAQ item 3 sobre possibilidade de exceções. Claude vai além nesse ponto: a afirmação "não pode" do assistente é tecnicamente correta para o processo padrão, mas cria expectativa zero para o cliente. A POL-001 seção 3.2 instrui encaminhamento ao Gestão de Riscos "para tratamento individual" — o que implica que há um processo alternativo. Dizer "não pode" sem mencionar que existe um caminho não padrão é, na prática, desinformação por omissão.

Adicionalmente: o assistente listou corretamente "classes 1 a 6", mas a classe específica perguntada (classe 3 — líquidos inflamáveis) também está explicitamente nomeada na POL-001. Confirmar a classe específica aumentaria a precisão e a confiança do atendente.

**Conclusão:** Alinhamento no veredicto. Claude reforça a natureza da omissão como desinformação ativa, não apenas encaminhamento incorreto.

---

### Resposta 4 — Política para carga danificada em transporte

**Resposta do assistente:** "A política de danos prevê reembolso integral quando comprovada negligência da transportadora, mediante laudo técnico e fotos." (Fonte: Nenhuma | Confiança: Alta)

#### Avaliador A — Analista
- Veredicto: ❌ Incorreta
- Foco: gap documental + conteúdo fabricado + confiança Alta indevida + omissão das instruções do FAQ

#### Avaliador B — Claude
- Veredicto: ❌ Incorreta
- Foco: mesmo diagnóstico, com uma diferença de enquadramento

**Diferença de enquadramento identificada por Claude:**

O analista tratou o problema principal como "conteúdo fabricado sem fonte". Claude concorda, mas adiciona um ângulo jurídico-semântico que o analista não abordou: o assistente usou a palavra **"negligência"**, que é um termo jurídico com implicação legal específica no direito civil brasileiro. A POL-001 seção 3.5 usa a expressão "defeito ou erro da NovaTech" — que é mais ampla e não implica necessariamente negligência. Um cliente que leu essa resposta poderia invocar o termo "negligência" numa disputa, e a NovaTech não teria como se defender com base em seus próprios documentos, pois o termo não consta em nenhum deles.

Esse é o único ponto onde Claude considera que o problema transcende a ausência de fonte: o assistente não apenas fabricou uma resposta, mas introduziu terminologia juridicamente sensível que não existe na documentação.

**Conclusão:** Alinhamento de veredicto. Claude acrescenta dimensão jurídica do uso da palavra "negligência".

---

### Resposta 5 — SLA do cliente Enterprise

**Resposta do assistente:** "O tier Enterprise não foi encontrado na documentação. Os tiers documentados são Gold, Silver e Standard. Sugiro confirmar a classificação ou escalar ao supervisor."

#### Avaliador A — Analista
- Veredicto: ✅ Correta
- Justificativa: comportamento exemplar para ausência de dado; único ponto de melhoria é o encaminhamento ao Comercial em vez de "supervisor"

#### Avaliador B — Claude
- Veredicto: ⚠️ Parcialmente correta

**Fundamentação da divergência:**

Claude discorda do veredicto "Correta" por uma razão que o analista identificou mas não considerou suficiente para rebaixar o veredicto: o encaminhamento.

A SLA-2024 seção 1 é prescritiva: *"Solicitações de SLA diferenciado fora desses tiers devem ser encaminhadas ao **Comercial** para análise de viabilidade."* O assistente disse "escalar ao supervisor" — que é uma instrução diferente, com destinatário diferente, e que não consta no documento.

Do ponto de vista do design de um assistente RAG, há dois comportamentos distintos quando uma informação não é encontrada:

1. **Correto:** não encontrar a informação e informar que não existe + encaminhar para o destino correto definido na documentação
2. **Incorreto:** não encontrar a informação e inventar um encaminhamento não documentado

O assistente fez o segundo. O fato de ter acertado que "Enterprise não existe" não compensa ter fabricado o encaminhamento. Em termos de critério de aceite, uma resposta correta exige que **tanto o conteúdo quanto o encaminhamento** sejam rastreáveis à fonte.

Portanto, Claude avalia como parcialmente correta: acertou o diagnóstico (tier inexistente), errou o encaminhamento (Comercial, não supervisor).

**Conclusão:** **Divergência de veredicto.** Analista: ✅ Correta. Claude: ⚠️ Parcialmente correta. A diferença está no peso atribuído ao encaminhamento incorreto.

---

### Resposta 6 — Envio de carga perigosa com frete expresso

**Resposta do assistente:** "Sim, cargas perigosas podem ser enviadas via frete expresso mediante autorização prévia do compliance e documentação ANTT atualizada." (Fonte: FAQ item 32 | Confiança: Alta)

#### Avaliador A — Analista
- Veredicto: ❌ Incorreta
- Foco: fonte informal com confiança Alta; ausência de documento formal; risco regulatório ANTT

#### Avaliador B — Claude
- Veredicto: ❌ Incorreta — com ressalva sobre o conteúdo factual

**Divergência de fundamentação:**

Ambos concordam no veredicto final (incorreta), mas divergem na análise do conteúdo factual.

O analista tratou o conteúdo da resposta como simplesmente não confiável por vir do FAQ. Claude concorda com isso, mas adiciona uma ressalva: o FAQ item 32 pode estar factualmente correto como prática operacional real — a equipe de atendimento descreve algo que acontece na prática. O problema não é necessariamente que o conteúdo seja falso; é que:

(a) não há como verificar, pois não existe documento formal  
(b) a confiança Alta sinaliza ao atendente que a informação é tão confiável quanto uma POL ou PROC, quando não é  
(c) o FAQ omite o detalhe operacional mais crítico para o cliente: *"Na prática, demora uns 2 dias para conseguir a autorização, então o 'expresso' acaba não sendo tão expresso."* — informação que estava no FAQ item 32 e foi descartada pelo assistente

Esse último ponto é relevante: o assistente selecionou apenas a parte positiva do FAQ (sim, é possível) e descartou a ressalva prática (mas não é realmente expresso). Isso configura uma omissão que distorce a expectativa do cliente independente da questão da fonte.

**Conclusão:** Alinhamento de veredicto. Divergência relevante na fundamentação: analista foca na qualidade da fonte; Claude foca adicionalmente na omissão seletiva de conteúdo que distorce a expectativa do cliente.

---

## Síntese das divergências

### Divergência 1 — Resposta 5: veredicto diferente

| Dimensão | Analista | Claude |
|---|---|---|
| Veredicto | ✅ Correta | ⚠️ Parcialmente correta |
| Critério | Acertou o conteúdo principal; encaminhamento é melhoria menor | Encaminhamento incorreto também é falha rastreável à fonte |
| Argumento central | O comportamento de não fabricar conteúdo é exemplar e deve ser reconhecido | Uma resposta correta exige que conteúdo *e* encaminhamento sejam rastreáveis à documentação |

**Posição de Claude:** Uma resposta de atendimento tem dois componentes — a informação e a ação seguinte. Se o assistente acerta a informação mas erra a ação, o atendente pode executar o passo errado. No caso do tier Enterprise, direcionar ao "supervisor" em vez do Comercial pode resultar em nenhuma análise de viabilidade sendo feita. O veredicto "parcialmente correta" preserva o reconhecimento do acerto principal sem ignorar a falha no encaminhamento.

---

### Divergência 2 — Resposta 6: fundamentação diferente

| Dimensão | Analista | Claude |
|---|---|---|
| Veredicto | ❌ Incorreta | ❌ Incorreta |
| Ênfase | Qualidade da fonte (FAQ informal com confiança Alta) | Qualidade da fonte + omissão seletiva de conteúdo que distorce expectativa |
| Ponto adicional | — | O assistente descartou a ressalva prática do próprio FAQ ("não é realmente expresso") |

---

### Pontos adicionais identificados por Claude não cobertos pelo analista

| Resposta | Ponto adicional |
|---|---|
| 1 | A referência de prazo é a data de recebimento no tracking, não a declaração do cliente (POL-001 seção 3.1) |
| 1 | Omissão da URL do portal e da categoria de chamado (POL-001 seção 3.3) |
| 2 | Ambiguidade da regra de pausa do relógio para Silver em incidentes críticos (SLA-2024 seção 5) |
| 3 | "Não pode" sem mencionar o caminho alternativo equivale a desinformação por omissão |
| 4 | Uso da palavra "negligência" introduz terminologia jurídica não presente em nenhum documento NovaTech |
| 6 | Omissão seletiva da ressalva prática do FAQ ("não é realmente expresso") distorce expectativa do cliente |

---

## Concordâncias fundamentais entre os avaliadores

Apesar das divergências pontuais, ambas as avaliações chegaram às mesmas conclusões estruturais:

1. **Resposta 4 é a falha mais crítica** — gap documental + confiança Alta + sem fonte = padrão de falha mais perigoso
2. **Resposta 6 é crítica por risco regulatório** — fonte informal tratada como normativa para assunto com implicação ANTT
3. **O padrão sistêmico de falha é o mesmo:** confiança Alta atribuída à existência de match semântico, não à qualidade da fonte
4. **Resposta 5 tem o melhor comportamento relativo** — mesmo com divergência de veredicto, ambos reconhecem que o assistente acertou o diagnóstico principal
5. **A regra de confiança por tipo de fonte está ausente** no design do assistente e precisa ser implementada

---

## Recomendação conjunta

A divergência mais significativa (Resposta 5) aponta para uma necessidade de definição de critério de aceite explícito no projeto:

> **Critério de aceite proposto:** Uma resposta é considerada "Correta" somente quando tanto o conteúdo informativo quanto o encaminhamento de ação são rastreáveis a pelo menos um documento formal da base. Respostas que acertam o conteúdo mas fabricam o encaminhamento devem ser classificadas como "Parcialmente corretas".

Esse critério, se adotado, elimina a ambiguidade de avaliação e serve como base para os testes de regressão do assistente.

---

## Referências documentais

| Documento | Versão | Responsável |
|---|---|---|
| POL-001 — Política de Devolução de Mercadorias | 3.1 | Diretoria de Operações |
| PROC-042 — Cálculo de Frete Especial | 1.0 | Diretoria Comercial |
| PROC-042-v2 — Cálculo de Frete Especial (Revisado) | 2.0 | Diretoria Comercial |
| SLA-2024 — Tabela de SLA por Tipo de Cliente | 2024.1 | Comercial + Operações |
| FAQ-Atendimento — Perguntas Frequentes | Não controlada | Nenhum (informal) |
