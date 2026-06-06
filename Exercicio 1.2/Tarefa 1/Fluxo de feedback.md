## Jornada do atendente — Sinalização de resposta errada, desatualizada ou incompleta ao assistente de IA

---

### Contexto de ativação

Este fluxo é acionado quando o atendente, após receber uma resposta do assistente, identifica que ela contém um erro factual, referencia uma versão de documento que foi substituída, ou deixa de cobrir informação relevante que o atendente sabe existir. Difere do fluxo de baixa confiança porque aqui a IA não emitiu sinal de incerteza — ela respondeu com aparente segurança, e o atendente é quem detecta o problema.

A sinalização cumpre dois propósitos simultâneos: proteger o cliente de receber uma informação incorreta no chamado imediato, e alimentar o ciclo de melhoria da base documental para que o erro não se repita nos chamados seguintes.

---

### Passo 1 — Caracterizar o tipo de problema identificado

Antes de qualquer ação, o atendente nomeia o que encontrou. Os três tipos têm consequências e caminhos distintos:

**Resposta errada:** a IA afirmou algo factualmente incorreto em relação ao documento que ela própria citou, ou citou um documento que não trata do tema perguntado. Exemplo: a IA indica que o prazo de devolução é 10 dias úteis, mas a POL-001 versão 3.1 — o documento citado — estabelece 7 dias úteis.

**Resposta desatualizada:** a IA retornou uma resposta tecnicamente correta para uma versão anterior do documento, mas existe uma versão mais recente que altera a informação. Exemplo: a IA usou os multiplicadores regionais da PROC-042 v1 (Norte: 1,6) quando o chamado se enquadra no período de vigência da PROC-042 v2 (Norte: 1,8).

**Resposta incompleta:** a IA retornou parte da informação correta, mas omitiu regras, exceções ou condições relevantes para o caso específico. Exemplo: a IA explicou o procedimento de devolução padrão da POL-001, mas não mencionou que cargas com lacre violado seguem tratamento diferenciado descrito na seção 3.2.

**Saída:** tipo de problema identificado e descrito pelo atendente antes de prosseguir.

---

### Passo 2 — Não usar a resposta e não responder ao cliente

A resposta problemática não é transmitida ao cliente em nenhuma hipótese. O atendente interrompe o fluxo principal no Passo 4 (avaliação da resposta) e registra no campo de notas internas do chamado que a resposta do assistente foi identificada como problemática, com descrição sucinta do problema.

**Saída:** chamado em estado de pendência; cliente protegido de informação incorreta.

---

### Passo 3 — Localizar a fonte correta

O atendente busca a informação correta diretamente na documentação oficial, sem intermediação do assistente neste momento. As fontes primárias são, em ordem de prioridade:

Documentos normativos com versão e responsável formal (POL, PROC, SLA) acessados diretamente no SharePoint. Tabelas de referência mensais na pasta de rede, verificando se a data do arquivo corresponde ao mês vigente. Confluence apenas para procedimentos operacionais não cobertos pelos documentos normativos.

O FAQ informal não é fonte válida para correção — ele pode ser a origem do próprio erro que está sendo corrigido.

Se o atendente não conseguir localizar a fonte correta de forma independente, aciona a fonte humana competente conforme o tema: Operações para prazos e procedimentos, Comercial para frete e SLA, Compliance para regras normativas.

**Saída:** informação correta localizada com identificação precisa da fonte — nome do documento, versão e seção.

---

### Passo 4 — Registrar o feedback no assistente

Com a informação correta em mãos, o atendente registra o feedback diretamente na interface do assistente, no mesmo contexto da resposta que originou o problema. O registro deve conter campos estruturados, não texto livre, para permitir triagem automatizada pela equipe de projeto:

**Tipo de problema:** seleção entre errado, desatualizado ou incompleto.

**Descrição objetiva:** o que a IA disse versus o que o documento correto estabelece. Exemplo: *"IA retornou multiplicador Norte = 1,6 com base na PROC-042 v1. O correto para chamados a partir de 01/12/2023 é 1,8, conforme PROC-042 v2, seção 2.1."*

**Documento correto:** nome, versão e seção que contém a informação correta.

**Documento problemático:** nome e versão do documento que a IA usou como base, quando identificável.

**Impacto no chamado:** se o erro foi detectado antes ou depois de qualquer comunicação com o cliente.

O registro é vinculado ao ID do chamado para rastreabilidade cruzada.

**Saída:** feedback estruturado registrado no assistente, disponível para a equipe de projeto e para as áreas documentais.

---

### Passo 5 — Responder ao cliente com a informação correta

Com a fonte correta identificada no Passo 3, o atendente retoma o fluxo principal a partir do Passo 5 (resposta ao cliente). A resposta usa a informação validada diretamente da documentação oficial. O documento correto e sua versão são registrados no chamado como fonte da resposta.

O atendente não menciona ao cliente que houve erro do assistente. Internamente, o chamado contém o registro completo do ocorrido para fins de auditoria.

**Saída:** cliente respondido com informação correta e rastreável; fonte registrada no chamado.

---

### Passo 6 — Acompanhar a resolução do problema na base

O feedback registrado no Passo 4 entra na fila de revisão gerenciada pela equipe de projeto em conjunto com as áreas documentais. O atendente não precisa acompanhar ativamente, mas deve estar ciente de que feedbacks recorrentes sobre o mesmo documento ou tema aceleram a priorização da correção.

As ações possíveis a partir do feedback são:

**Reindexação imediata:** quando o documento correto já existe na base mas não foi indexado ou foi indexado em versão errada. A equipe de projeto reprocessa o documento e a correção entra em produção sem necessidade de atualização documental.

**Arquivamento de versão obsoleta:** quando o problema é causado pela coexistência de duas versões ativas, como PROC-042 v1 e v2. A área responsável marca formalmente a versão antiga como obsoleta no SharePoint, e a equipe de projeto remove ou desclassifica o documento na base do assistente.

**Criação de documento formal:** quando o problema é uma resposta incompleta baseada no FAQ informal sobre tema sem cobertura normativa — como seguro de carga ou carga danificada. A área responsável cria o documento oficial, que é então ingerido na base.

**Correção de chunking ou metadado:** quando o documento está correto na base mas o assistente não está recuperando o trecho certo por problema técnico de segmentação ou indexação. A equipe de projeto ajusta o pipeline sem intervenção documental.

O atendente recebe notificação quando o problema que ele reportou for resolvido na base, fechando o ciclo de forma visível.

**Saída:** problema rastreado até resolução; atendente informado do fechamento.

---

### Condições de desvio dentro deste fluxo

| Situação | Ação |
|---|---|
| Atendente não consegue localizar a fonte correta sozinho | Aciona fonte humana competente antes de registrar o feedback — o feedback deve conter a fonte correta, não apenas o erro |
| O mesmo erro já foi reportado por outro atendente | Registrar mesmo assim — ocorrências múltiplas do mesmo problema elevam a prioridade de correção na fila de revisão |
| Erro ocorreu e o cliente já foi informado com dado incorreto | Registrar impacto no chamado como "cliente já comunicado"; supervisor avalia necessidade de contato corretivo com o cliente |
| Atendente não tem certeza se é erro da IA ou da documentação | Registrar como "incerto" com descrição do conflito percebido — a equipe de projeto investiga a origem |
| Problema envolve documento em revisão pelo Compliance | Registrar e sinalizar que o documento-fonte está em revisão ativa; equipe de projeto aguarda versão final antes de reindexar |

---

### Princípios que governam este fluxo

O feedback do atendente é a principal fonte de inteligência sobre a qualidade da base documental. Nenhum processo de revisão automatizado detecta com a mesma precisão os casos em que a informação retornada não corresponde à realidade operacional — porque essa correspondência só é verificável por quem conhece o contexto do chamado.

O registro deve ser objetivo e baseado em fonte, não em percepção. *"Acho que está errado"* não tem valor de ação. *"A IA retornou X com base no documento Y versão Z; o correto é W conforme documento A versão B, seção C"* tem.

A sinalização não é crítica ao assistente — é uma contribuição ao sistema. O atendente que registra feedback consistente está diretamente melhorando a ferramenta que reduz seu próprio tempo de busca. Isso deve ser comunicado claramente durante o treinamento de onboarding do assistente.