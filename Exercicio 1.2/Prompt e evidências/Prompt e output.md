### Fornecido Anexo A e Cenário
Com base no cenário abaixo e anexo A, crie um mapa da jornada do atendente utilizando IA para localizar respostas aos chamados

```markdown
O Cenário

A NovaTech é uma empresa de médio porte do setor de logística com 1.200 funcionários. Sua operação depende de um conjunto extenso de documentação interna: manuais de procedimento operacional, políticas de compliance, tabelas de SLA por tipo de cliente, regras de cálculo de frete, e normas de segurança de carga.

Hoje, essa documentação está espalhada em três fontes: um SharePoint corporativo com ~800 documentos (PDFs e Word), uma wiki interna no Confluence com ~400 páginas, e uma pasta de rede com planilhas de referência atualizadas mensalmente.

O problema: a equipe de atendimento ao cliente (45 pessoas) gasta em média 12 minutos por chamado buscando informações nessas fontes para responder dúvidas de clientes sobre prazos, regras de frete, políticas de devolução e procedimentos de reclamação. Isso gera atrasos, respostas inconsistentes e frustração tanto dos atendentes quanto dos clientes.

A NovaTech contratou a DB1 para construir um assistente de IA que permita aos atendentes fazer perguntas em linguagem natural e receber respostas fundamentadas na documentação oficial da empresa, com indicação da fonte. O assistente será integrado ao ambiente Microsoft da NovaTech (Teams + SharePoint).

### Informações adicionais fornecidas pela NovaTech

- O volume médio é de 320 chamados/dia, dos quais ~60% envolvem consulta a documentação.
- A documentação é atualizada mensalmente por 3 áreas diferentes (Operações, Compliance, Comercial), sem processo unificado de revisão.
- Alguns documentos se contradizem entre versões — a equipe de atendimento hoje resolve isso "perguntando para quem sabe".
- A NovaTech já tem licenças Microsoft 365 E3 e está disposta a provisionar Azure AI Services.
- O projeto tem orçamento para 3 meses de discovery + desenvolvimento + go-live.
- A expectativa da diretoria é reduzir o tempo médio de busca de 12 para menos de 2 minutos por chamado.

---

### Solicitado: Em formato de texto estruturado, descreva o fluxo principal do atendente

## Fluxo principal do atendente — NovaTech com Assistente de IA

---

### Pré-condição
Chamado aberto no portal pelo cliente com número de CT-e, descrição do problema e dados de contato. Sistema registra timestamp e inicia contagem de SLA automaticamente conforme tier do cliente (Gold: 2h resposta / Silver: 4h / Standard: 8h).

---

### Passo 1 — Receber e contextualizar o chamado

O atendente acessa o chamado no Azure DevOps integrado ao Teams. Verifica o tier do cliente para saber o SLA vigente e lê a descrição para entender o tipo de dúvida. Nenhuma consulta documental acontece ainda.

**Saída:** atendente sabe quem é o cliente, qual é o SLA e qual é o problema relatado.

---

### Passo 2 — Classificar a dúvida

O atendente identifica a categoria do chamado entre os tipos mais comuns: prazo de entrega, cálculo de frete, devolução de mercadoria, SLA contratual ou procedimento de reclamação. Essa classificação determina qual documentação será relevante na consulta.

**Saída:** categoria definida, atendente sabe que precisa consultar documentação.

---

### Passo 3 — Consultar o assistente de IA

O atendente abre o assistente diretamente no Teams e formula a pergunta em linguagem natural — por exemplo, *"qual o prazo para devolução de carga refrigerada?"* ou *"qual o multiplicador de frete especial para o Nordeste?"*. O assistente realiza busca semântica sobre os documentos indexados (SharePoint, Confluence e planilhas de rede) e retorna uma resposta fundamentada com o trecho relevante, o nome do documento, a versão e a data de atualização. Se necessário, o atendente refina a pergunta para obter mais precisão.

**Saída:** resposta em linguagem natural com fonte identificada, entregue em menos de 30 segundos.

---

### Passo 4 — Avaliar a resposta

O atendente verifica se a resposta cobre o caso específico do cliente. Confere a versão do documento citado e a data de atualização. Se o assistente sinalizar conflito entre versões — como ocorre com PROC-042 v1 e v2 — o atendente observa qual versão está indicada como vigente para o período do chamado e usa essa como referência.

**Saída:** resposta validada e pronta para uso, ou identificação de que o caso exige tratamento de exceção.

---

### Passo 5 — Tratar exceção (quando aplicável)

Acionado quando a resposta da IA não cobre o caso, quando há conflito documental sem resolução clara, ou quando a situação envolve categorias de risco elevado (carga perigosa, incidente crítico, carga com lacre violado). O atendente escala para o supervisor ou contata diretamente a área responsável — Gestão de Riscos (ramal 4500) para cargas perigosas, Comercial para negociações de prazo expirado ou desconto, Jurídico para carga danificada em trânsito. O caso é registrado com flag de exceção no sistema.

**Saída:** decisão tomada com respaldo da área competente e registro formal no chamado.

---

### Passo 6 — Responder o cliente e encerrar o chamado

O atendente redige a resposta ao cliente com base na documentação oficial consultada, sem transcrevê-la literalmente — adaptando a linguagem ao contexto. Registra no chamado qual documento foi usado como fonte. Encerra o chamado ou o encaminha para a fila correta caso dependa de ação de outra área. O sistema recalcula o tempo de resolução e atualiza as métricas de SLA.

**Saída:** cliente respondido com informação precisa e rastreável, chamado encerrado com fonte registrada.

---

### Condições de desvio

| Situação | Desvio |
|---|---|
| Carga ainda em trânsito | Não aplica POL-001 — consultar PROC-088 (Interceptação de Carga) |
| Carga perigosa (classes 1–6 ANTT) | Não usar processo padrão de devolução — acionar Gestão de Riscos (ramal 4500) |
| Solicitação de devolução após 7 dias úteis | Encaminhar ao Comercial para negociação caso a caso |
| Conflito entre PROC-042 v1 e v2 | Usar v2 para chamados a partir de 01/12/2023; v1 para chamados anteriores ainda em processamento |
| Resposta vem apenas do FAQ informal | Não usar como fonte definitiva — confirmar com área responsável antes de responder |
| Incidente crítico de cliente Gold | SLA não pausa fora do horário comercial — tratamento prioritário imediato |
| Cliente menciona tier "Platinum" | Tier inexistente — orientar que os tiers são Gold, Silver e Standard; solicitar número do contrato |

---

### Invariantes do fluxo

O atendente é sempre o responsável pela decisão final — o assistente de IA é uma ferramenta de apoio, não uma autoridade. Toda resposta ao cliente deve ter uma fonte documental rastreável registrada no chamado. Casos sem cobertura documental formal não devem ser respondidos com base no FAQ informal sem validação prévia da área responsável.

---
### Solicitado: Elabore a jornada do atendente em formato de texto estruturado de Qual solução deve ser aplicada quando a assistente de IA não tem confiança na resposta para a pergunta do atendente ou quando o atendente discorda da resposta
## Jornada do atendente — Tratamento de respostas de baixa confiança ou divergência com a IA

---

### Contexto de ativação

Este fluxo é acionado em dois cenários distintos, ambos ocorrendo após o Passo 3 do fluxo principal (consulta ao assistente):

**Cenário A — Baixa confiança da IA:** o assistente retorna uma resposta sinalizando explicitamente incerteza. Isso ocorre quando nenhum documento da base cobre o caso com clareza suficiente, quando há conflito entre versões sem resolução definida, ou quando a pergunta envolve tópicos com gap documental conhecido (seguro de carga, carga danificada, frete padrão abaixo de 500kg, procedimento da Gestão de Riscos).

**Cenário B — Discordância do atendente:** o assistente retorna uma resposta com confiança, mas o atendente a questiona com base em experiência prévia, em informação recebida de colega ou supervisor, ou em percepção de que o documento citado está desatualizado.

Ambos os cenários interrompem o fluxo principal antes do Passo 6 (resposta ao cliente) e seguem o tratamento descrito abaixo.

---

### Passo 1 — Identificar o motivo da incerteza

O atendente lê o sinal emitido pelo assistente ou articula sua própria discordância. Os motivos possíveis se enquadram em quatro categorias:

**Gap documental:** a pergunta envolve um assunto sem documento normativo formal na base. Exemplos do contexto NovaTech: política de carga danificada em trânsito, percentuais de seguro de carga, frete abaixo de 500kg. A IA pode ter retornado uma resposta baseada no FAQ informal, que não é validado pelo Compliance.

**Conflito entre versões:** dois documentos ativos respondem à mesma pergunta de forma diferente. Exemplo direto: PROC-042 v1 e v2 coexistem com multiplicadores regionais, fatores de peso e prazo de entrega distintos, sem marcação formal de obsolescência no SharePoint.

**Cobertura parcial:** o documento citado existe e é válido, mas não cobre a especificidade do caso — por exemplo, a POL-001 cobre devolução padrão, mas o caso envolve carga com lacre violado e ausência do motorista no momento da entrega, situação não detalhada na política.

**Conhecimento tácito divergente:** o atendente tem uma prática consolidada que difere do que a IA retornou. Pode indicar que a documentação está desatualizada, que existe uma instrução verbal não documentada, ou que a prática informal está errada e precisa ser corrigida.

**Saída:** motivo da incerteza categorizado antes de qualquer ação.

---

### Passo 2 — Não responder ao cliente ainda

Independentemente do motivo, o atendente não avança para o Passo 6 do fluxo principal. Responder com informação de confiança baixa ou com fonte em conflito gera inconsistência entre atendentes e pode criar compromissos contratuais incorretos com o cliente — especialmente em casos envolvendo SLA, cálculo de frete e devoluções.

O chamado permanece aberto. O atendente registra no campo de notas internas do chamado que a resposta está pendente de validação, com breve descrição do motivo.

**Saída:** cliente não recebeu resposta incorreta; chamado em estado de pendência documentada.

---

### Passo 3 — Reformular a pergunta ao assistente

Antes de escalar, o atendente tenta resolver internamente refinando a consulta. Reformulações eficazes incluem:

- Especificar o documento que deveria cobrir o caso: *"Com base na POL-001 versão 3.1, qual é o procedimento para devolução de carga com lacre violado?"*
- Restringir o escopo temporal: *"Qual o multiplicador regional para o Norte segundo a PROC-042-v2, vigente a partir de dezembro de 2023?"*
- Separar a pergunta em partes menores quando ela combina múltiplas regras.
- Pedir explicitamente que o assistente liste quais documentos cobrem o tema, sem gerar uma resposta sintetizada.

Se a reformulação retornar uma resposta com fonte clara e sem conflito, o fluxo retorna ao Passo 4 do fluxo principal. Se a incerteza persistir, avança para o próximo passo.

**Saída:** incerteza resolvida internamente (retorna ao fluxo principal) ou confirmada como não resolvível pelo assistente (avança).

---

### Passo 4 — Acionar a fonte humana competente

A escalada segue a categoria de motivo identificada no Passo 1:

**Gap documental** — acionar a área responsável pelo tema ausente. Para carga danificada: encaminhar para o e-mail sinistros@novatech.com.br e aguardar orientação antes de responder. Para seguro de carga: contatar o Comercial para confirmar percentual aplicável ao contrato do cliente. Para frete abaixo de 500kg: solicitar ao Comercial a tabela vigente.

**Conflito entre versões** — acionar a área autora do documento mais recente. Para PROC-042: contatar a Diretoria Comercial para confirmar qual versão aplica ao contrato e ao período do chamado. Registrar a resposta recebida no chamado para rastreabilidade.

**Cobertura parcial** — acionar o supervisor de atendimento ou o especialista da área operacional que conhece o caso análogo. Descrever o caso específico e pedir orientação por escrito (e-mail ou mensagem no Teams), não verbal.

**Conhecimento tácito divergente** — acionar o supervisor imediato. Apresentar lado a lado o que a IA retornou (com fonte) e o que o atendente conhece como prática. Solicitar que o supervisor indique qual está correto e, se a prática for a correta, que abra solicitação de atualização documental.

Em todos os casos, a orientação recebida deve ser registrada no chamado antes de responder ao cliente.

**Saída:** orientação validada por fonte humana competente, registrada formalmente no chamado.

---

### Passo 5 — Responder ao cliente com base na orientação validada

Com a orientação registrada, o atendente responde ao cliente. A resposta cita a informação validada sem expor detalhes internos sobre o conflito ou a incerteza. O atendente não menciona ao cliente que houve divergência ou que a IA não soube responder.

Se a resolução exigir ação de outra área — coleta reversa, análise de sinistro, renegociação comercial — o atendente informa ao cliente o próximo passo e o prazo esperado, sem comprometer prazos que não controlam.

**Saída:** cliente respondido com informação validada e rastreável.

---

### Passo 6 — Registrar o caso para melhoria da base documental

Este passo é obrigatório e ocorre após o encerramento do chamado. O atendente registra o caso em um formulário de feedback do assistente — a ser definido pela equipe de projeto — com os seguintes campos:

- Pergunta feita ao assistente
- Motivo da incerteza (gap, conflito, cobertura parcial, tácito divergente)
- Área acionada para resolução
- Orientação recebida
- Sugestão de ação: criar documento, atualizar versão, arquivar versão obsoleta, validar FAQ

Esses registros alimentam o processo mensal de revisão documental das três áreas responsáveis (Operações, Compliance, Comercial), criando um ciclo de melhoria contínua da base de conhecimento do assistente.

**Saída:** caso documentado, base de conhecimento sinalizando gaps ativos para as equipes responsáveis.

---

### Condições de desvio dentro deste fluxo

| Situação | Ação |
|---|---|
| Fonte humana não responde dentro do SLA do chamado | Supervisor do atendente assume responsabilidade pela decisão e registra o risco formalmente |
| Orientação recebida contradiz o documento da IA | Registrar ambas no chamado; acionar o supervisor para definir qual prevalece antes de responder |
| Caso envolve incidente crítico de cliente Gold | SLA não pausa — a escalada humana deve ocorrer em paralelo à consulta à IA, não em sequência |
| Atendente recebe orientação apenas verbal | Solicitar confirmação por escrito antes de responder ao cliente; orientação verbal não é registrável |
| Gap identificado já foi reportado por outro atendente | Verificar se há resolução pendente no sistema; não duplicar escalada, mas registrar ocorrência adicional para priorização |

---

### Princípios que governam este fluxo

A incerteza da IA não é uma falha — é um sinal de qualidade. Um assistente que sinaliza quando não sabe é mais confiável do que um que responde com falsa segurança. O atendente deve tratar esse sinal como informação útil, não como obstáculo.

A discordância do atendente também é um sinal de qualidade. Quando um atendente experiente percebe divergência entre a IA e a prática real, isso quase sempre indica um de dois problemas: a documentação está desatualizada, ou a prática informal está errada. Ambos precisam ser resolvidos — e o registro no Passo 6 é o mecanismo para isso.

Nenhuma resposta ao cliente deve sair sem fonte rastreável registrada no chamado. Isso vale tanto para respostas geradas com confiança alta quanto para as que passaram por este fluxo de exceção.

---

### Solicitado: Elabore a jornada do atendente em formato de texto estruturado De como o atendente sinaliza para a assistente que uma resposta está errada, desatualizada ou incompleta

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

---

### Solicitado: Elabore no formato de texto estruturado 3 restrições e regras de comportamento que a assistente deve adotar quando não tem a resposta de forma consistente e segura para dar ao atendente

## Restrições e regras de comportamento do assistente — Respostas sem cobertura consistente e segura

---

### Contexto de aplicação

Estas três restrições governam o comportamento do assistente nos momentos em que ele não dispõe de resposta com cobertura documental suficiente para ser considerada confiável. Elas não são degradações de qualidade — são garantias de integridade. Um assistente que sabe quando não sabe é mais valioso operacionalmente do que um que responde sempre, porque o atendente pode calibrar sua confiança na ferramenta e agir de forma apropriada em cada situação.

As regras se aplicam de forma cumulativa: quando uma situação ativa mais de uma delas, todas devem ser observadas simultaneamente.

---

### Restrição 1 — Nunca sintetizar resposta a partir de fontes informais ou conflitantes sem declaração explícita

**O que a restrição proíbe**

O assistente não deve construir uma resposta aparentemente coerente combinando trechos de documentos com níveis de confiabilidade diferentes — por exemplo, misturando a POL-001 com o FAQ informal — sem informar ao atendente que as fontes têm status distintos. Também não deve escolher silenciosamente uma versão de documento quando duas versões conflitantes estão ativas, apresentando o resultado como se fosse a única interpretação possível.

A síntese silenciosa é o comportamento de maior risco no contexto da NovaTech porque a base documental contém contradições conhecidas e gaps com cobertura apenas informal. Uma resposta que parece completa e fundamentada, mas foi construída sobre fontes inadequadas, é mais perigosa do que a ausência de resposta — porque o atendente não tem razão para questionar o que não sabe que está errado.

**O que o assistente deve fazer no lugar**

Quando a resposta exigir combinação de fontes com status diferente, o assistente declara explicitamente quais fontes foram usadas, qual é o status de cada uma e onde está o limite da cobertura confiável. Quando houver conflito ativo entre versões, o assistente apresenta as duas versões lado a lado com seus respectivos metadados — documento, versão, data, área responsável — e indica qual critério de aplicação existe, se houver. No caso da PROC-042, por exemplo, o assistente informa que a v1 aplica a chamados abertos antes de 01/12/2023 ainda em processamento e a v2 aplica a chamados novos a partir dessa data, conforme a seção 5 da própria v2, e deixa a decisão de qual usar ao atendente com base na data do chamado.

**Formato de sinalização obrigatório**

A resposta deve abrir com uma declaração de status antes de qualquer conteúdo informativo, seguindo este padrão:

*"Atenção: esta resposta combina [documento oficial X, versão Y] com [FAQ informal, sem validação de Compliance]. A parte relativa a [tema Z] vem exclusivamente do FAQ e não possui cobertura normativa confirmada. Recomendo validar com [área responsável] antes de responder ao cliente."*

---

### Restrição 2 — Nunca omitir a ausência de cobertura documental quando o tema consultado tem gap conhecido

**O que a restrição proíbe**

O assistente não deve responder a perguntas sobre temas sem cobertura documental formal usando apenas o FAQ informal como se fosse fonte equivalente aos documentos normativos, sem nenhuma distinção. Também não deve simplesmente deixar de responder sem explicar por quê — a ausência de resposta sem explicação força o atendente a concluir erroneamente que o assistente falhou tecnicamente, quando na verdade o problema está na base documental.

**O que o assistente deve fazer no lugar**

Quando a pergunta recai sobre um dos gaps documentais identificados na base da NovaTech — seguro de carga, política de carga danificada em trânsito, frete abaixo de 500kg, procedimento formal da Gestão de Riscos — o assistente declara o gap de forma direta, informa o que existe na base sobre o tema e qual é o status dessa cobertura, e indica a rota de escalada humana adequada para o caso.

A declaração do gap é informação operacional útil, não admissão de falha. O atendente que sabe que não existe documento formal sobre seguro de carga age diferente do atendente que acha que o assistente simplesmente não encontrou o arquivo.

**Formato de sinalização obrigatório**

*"Não existe documento normativo formal na base sobre [tema]. A única referência disponível é o FAQ interno [item X], que não foi validado pelo Compliance e pode conter informações desatualizadas. Para responder ao cliente com segurança, acione [área responsável / contato específico]. Registre o gap no formulário de feedback para priorização de criação documental."*

O assistente deve manter uma lista interna dos gaps documentais conhecidos — inicialmente os quatro identificados no Anexo A — e reconhecê-los proativamente quando a pergunta se aproximar desses temas, mesmo que a pergunta não seja idêntica às perguntas que revelaram o gap originalmente.

---

### Restrição 3 — Nunca apresentar nível de confiança uniforme entre respostas de qualidade documental distinta

**O que a restrição proíbe**

O assistente não deve usar o mesmo formato, tom e estrutura de resposta independentemente da qualidade da cobertura documental encontrada. Tratar com igual aparência uma resposta baseada na POL-001 versão 3.1 — documento normativo com responsável formal, data de atualização e classificação de uso obrigatório — e uma resposta baseada no FAQ informal — documento colaborativo sem responsável, sem versão controlada e com aviso interno de não validação — induz o atendente a calibrar sua confiança de forma incorreta.

A uniformidade visual da resposta é um vetor de risco porque o atendente em situação de pressão de SLA tende a consumir a resposta sem ler os metadados. Se o formato não distingue a qualidade da fonte, o atendente não tem como perceber a diferença sem esforço adicional de leitura que o contexto operacional muitas vezes não permite.

**O que o assistente deve fazer no lugar**

O assistente adota um sistema de três níveis de confiança, aplicado visualmente e verbalmente a cada resposta, de forma que o nível seja perceptível sem leitura dos metadados completos:

**Nível confirmado:** resposta baseada exclusivamente em documento normativo formal com versão controlada, responsável identificado e data de atualização recente. O assistente apresenta a resposta diretamente, seguida da fonte. Nenhum aviso adicional é necessário além da citação. Exemplo de aplicação: resposta sobre prazo de devolução baseada na POL-001 v3.1, seção 3.1.

**Nível atenção:** resposta baseada em documento formal existente, mas com pelo menos uma das seguintes condições — conflito ativo com outra versão, documento cuja área responsável informou revisão em andamento, ou cobertura parcial que não abrange todas as dimensões da pergunta. O assistente apresenta a resposta com marcação visual de atenção e descreve explicitamente a condição que reduz a confiança. O atendente pode usar a informação, mas deve verificar a condição sinalizadas antes de responder ao cliente em casos de alto impacto contratual.

**Nível sem cobertura segura:** resposta não encontrada em documento formal, ou encontrada apenas em fonte informal não validada, ou construída por inferência sobre documentos que não tratam diretamente do tema. O assistente não apresenta conteúdo informativo como se fosse resposta — apresenta o diagnóstico do gap, o que existe na base sobre o tema e a rota de escalada. O atendente não deve usar nenhum conteúdo deste nível para responder ao cliente sem validação humana prévia.

**Formato de sinalização obrigatório por nível**

Nível confirmado: *"[Resposta]. Fonte: [documento], versão [X], seção [Y], atualizado em [data]."*

Nível atenção: *"⚠ Atenção: [condição que reduz a confiança]. Com base em [documento, versão, seção]: [resposta]. Verifique [condição específica] antes de responder ao cliente em casos de impacto contratual."*

Nível sem cobertura segura: *"✗ Sem cobertura documental segura para esta pergunta. [Diagnóstico do gap]. O que existe na base: [descrição do que foi encontrado e seu status]. Próximo passo: [rota de escalada com contato ou área específica]. Registre no formulário de feedback para priorização."*

---

### Interdependência entre as três restrições

As três restrições formam um sistema coerente e não devem ser aplicadas de forma isolada.

A Restrição 1 governa o comportamento quando existem fontes, mas elas são conflitantes ou têm status diferente. A Restrição 2 governa o comportamento quando não existe fonte formal sobre o tema. A Restrição 3 governa a forma como qualquer resultado — de qualquer nível de cobertura — é comunicado ao atendente.

Na prática, toda resposta do assistente passa pelas três verificações em sequência: primeiro, quais fontes foram usadas e qual é o status de cada uma (Restrição 1); segundo, existe gap documental formal sobre o tema (Restrição 2); terceiro, qual nível de confiança deve ser atribuído ao conjunto e como isso deve ser comunicado (Restrição 3).

O resultado é que o atendente nunca recebe uma resposta sem saber exatamente o que ela vale — e isso é a condição mínima para que ele possa agir com responsabilidade no chamado.

--- 

### Claude Cowork para geração da jornada do atendente com  Assistente de IA
Anexo na tarefa 2 o HTML da jornada do atendente conforme os fluxos mapeados.
Também gerado no formato SVG

