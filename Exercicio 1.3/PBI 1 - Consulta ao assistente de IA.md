## PBI 01 — Consulta ao assistente de IA durante atendimento de chamado

### 1. Contexto

O time de atendimento da NovaTech possui 45 atendentes que respondem em média 320 chamados por dia, dos quais 60% exigem consulta à documentação interna para serem respondidos. Hoje, essa consulta é feita manualmente em três fontes separadas — SharePoint, Confluence e pasta de rede — consumindo em média 12 minutos por chamado e gerando respostas inconsistentes entre atendentes.

O assistente de IA deve permitir que o atendente faça perguntas em linguagem natural diretamente no Microsoft Teams e receba respostas fundamentadas na documentação oficial da NovaTech, com indicação do documento de origem, versão e data de atualização. A expectativa é reduzir o tempo de busca de 12 minutos para menos de 2 minutos por chamado.

O assistente responde apenas com base nos trechos de documentos que encontrou como relevantes para a pergunta feita. Se a informação não estiver presente nos documentos indexados, o assistente não deve inventar uma resposta — deve declarar a ausência conforme as regras definidas no PBI 04.

### 2. Premissas e Riscos

**2.1 Premissas**
- O assistente está integrado ao Microsoft Teams e acessível durante o atendimento sem troca de aplicação.
- Um documento é considerado oficial e elegível para indexação quando publicado por uma das três áreas responsáveis pela documentação da NovaTech — Operações, Compliance ou Comercial — nas fontes oficiais: SharePoint, Confluence ou pasta de rede, após aprovação formal da área publicadora.
- O FAQ interno do time de atendimento não é documento oficial e não compõe a base de documentos do assistente por não ter sido validado pelo Compliance.
- O atendente é sempre o responsável pela decisão final — o assistente é uma ferramenta de apoio à consulta, não uma autoridade.
- O chamado está registrado no Azure DevOps com o tier do cliente e o timestamp de abertura identificados.
- A equipe responsável pela gestão da base documental do assistente é formada pelos representantes das áreas de Operações, Compliance e Comercial, coordenada pela equipe de projeto da DB1.

**2.2 Riscos**
- Documentos com versões conflitantes coexistindo na base — como PROC-042 v1 e v2 — podem gerar respostas com informações divergentes entre atendentes se o assistente não sinalizar o conflito conforme definido no PBI 04.
- O atendente sob pressão de prazo de atendimento pode usar a resposta sem verificar o documento de origem citado.
- Perguntas sobre temas sem cobertura documental formal — como frete abaixo de 500kg — não possuem trechos na base para embasar resposta. O comportamento do assistente nesses casos é definido no PBI 04.
- A publicação de um documento por área sem aprovação formal pode introduzir conteúdo não validado na base se o processo de elegibilidade não for respeitado.

### 3. Diretório

Assistente NovaTech → Painel de atendimento → Consulta à documentação

*(Ajustar conforme o menu real do sistema)*

### 4. Requisitos

**4.1 Acesso ao assistente durante o atendimento**

O assistente deve estar disponível para o atendente diretamente no Microsoft Teams, sem necessidade de abrir outro sistema durante o atendimento ao cliente.

**4.2 Consulta em linguagem natural**

O atendente deve poder digitar perguntas sobre prazos, regras de frete, políticas de devolução, SLA por tipo de cliente e procedimentos de reclamação usando suas próprias palavras, sem necessidade de usar termos técnicos ou códigos de documento.

**4.3 Resposta fundamentada em documentação oficial**

O assistente deve responder apenas com base nos trechos dos documentos oficiais indexados. O assistente não deve inventar informações que não estejam presentes nesses documentos. Quando não houver informação disponível, o assistente deve declarar a ausência conforme as regras de comportamento definidas no PBI 04.

**4.4 Exibição obrigatória da fonte**

Toda resposta deve indicar obrigatoriamente:
- Nome do documento utilizado
- Versão do documento
- Data da última atualização
- Seção ou trecho de referência
- Área responsável pelo documento

**4.5 Tempo de resposta**

O assistente deve retornar a resposta ao atendente em até 30 segundos após o envio da pergunta.

**4.6 Refinamento da pergunta**

O atendente deve poder reformular ou complementar a pergunta na mesma sessão para obter maior precisão, sem perder o histórico da conversa.

**4.7 Registro obrigatório da fonte no chamado**

Ao utilizar uma resposta do assistente para responder ao cliente, o atendente deve registrar no chamado (Azure DevOps) qual documento foi utilizado como fonte. O sistema deve exigir esse registro como condição para o encerramento do chamado, impedindo o fechamento sem que a fonte esteja informada.

**4.8 Atualização da base documental com novos documentos**

Quando um novo documento for publicado ou uma nova versão de documento existente for disponibilizada nas fontes oficiais — SharePoint, Confluence ou pasta de rede — o sistema deve processar e disponibilizar o conteúdo atualizado para consulta no assistente em até 24 horas corridas após a publicação, independentemente do dia da semana ou horário.

Durante o período de processamento, o assistente deve continuar operando normalmente com a versão anterior do documento, sem interrupção do atendimento.

Quando o processamento de um novo documento for concluído, o sistema deve notificar a equipe responsável pela gestão da base documental confirmando a disponibilização do conteúdo para consulta.

Documentos publicados com marcação formal de substituição de versão anterior devem, após o processamento, manter ambas as versões disponíveis na base quando houver critério de aplicação por período — como é o caso da PROC-042 v1 e v2 — sinalizando ao assistente qual versão aplicar conforme a data de abertura do chamado. Versões sem critério de aplicação definido devem ter a versão anterior marcada como indisponível para novas consultas, mantendo apenas a versão vigente como fonte de resposta.

### 5. Critérios de Aceite

**CA01 – Resposta com identificação da fonte**
Garantir que toda resposta do assistente exiba o nome do documento utilizado, a versão, a data de atualização e a seção de referência antes de ser apresentada ao atendente.

**CA02 – Resposta dentro do prazo**
Garantir que o assistente retorne a resposta ao atendente em até 30 segundos após o envio da pergunta.

**CA03 – Resposta baseada apenas em documentos oficiais**
Garantir que o assistente utilize exclusivamente os documentos oficiais indexados como base para suas respostas, sem usar o FAQ informal como fonte principal.

**CA04 – Declaração de ausência de informação**
Garantir que o assistente informe explicitamente ao atendente quando não encontrar informação sobre o tema consultado, sem apresentar uma resposta inventada no lugar, seguindo o formato definido no PBI 04.

**CA05 – Refinamento na mesma sessão**
Garantir que o atendente possa reformular a pergunta após receber uma resposta e que o assistente processe a nova consulta mantendo o contexto da conversa em andamento.

**CA06 – Acesso sem troca de aplicação**
Garantir que o assistente esteja acessível ao atendente diretamente no Microsoft Teams, sem necessidade de abrir outro sistema durante o atendimento.

**CA07 – Registro de fonte como condição de encerramento do chamado**
Garantir que o sistema impeça o encerramento do chamado no Azure DevOps quando o campo de fonte da resposta não estiver preenchido pelo atendente.

**CA08 – Disponibilização de novo documento em até 24 horas corridas**
Garantir que todo documento publicado ou atualizado nas fontes oficiais esteja disponível para consulta no assistente em até 24 horas corridas após a publicação, sem necessidade de intervenção manual da equipe de TI.

**CA09 – Continuidade do atendimento durante o processamento**
Garantir que o assistente continue respondendo normalmente com a versão anterior do documento durante o período de processamento de uma nova versão, sem interrupção do serviço de atendimento.

**CA10 – Notificação de conclusão do processamento**
Garantir que a equipe responsável pela gestão da base documental receba uma notificação quando o processamento de um novo documento for concluído e o conteúdo estiver disponível para consulta no assistente.

**CA11 – Manutenção de versões com critério de aplicação por período**
Garantir que quando uma nova versão de documento for publicada com critério de aplicação por período — como a PROC-042 v1 e v2 — ambas as versões permaneçam disponíveis na base, com o assistente aplicando a versão correta conforme a data de abertura do chamado consultado.

**CA12 – Indisponibilização de versão substituída sem critério de período**
Garantir que quando uma nova versão de documento for publicada sem critério de aplicação por período, a versão anterior seja marcada como indisponível para novas consultas, mantendo apenas a versão vigente como fonte de resposta.

### 6. Cenários

**Cenário 1 – Pergunta com resposta em documento oficial**

Dado que o atendente pergunte "qual o prazo para o cliente solicitar devolução?"
Quando o assistente localizar a informação na POL-001 v3.1, seção 3.1
Então deve retornar que o prazo é de 7 dias úteis após o recebimento confirmado, com indicação do documento, versão e seção, classificada como grau Confirmado conforme PBI 04

**Cenário 2 – Pergunta sobre SLA de cliente Gold**

Dado que o atendente pergunte "qual o SLA de resposta para cliente Gold?"
Quando o assistente localizar a informação na SLA-2024, seção 2
Então deve retornar que o prazo de primeira resposta é de até 2 horas úteis para chamados gerais e 30 minutos para incidentes críticos, com indicação da fonte, classificada como grau Confirmado

**Cenário 3 – Pergunta sem cobertura documental**

Dado que o atendente pergunte "qual o frete para 300kg para Salvador?"
Quando o assistente não encontrar trechos que cubram frete abaixo de 500kg
Então deve retornar a declaração de ausência conforme formato definido no PBI 04, orientando o atendente a acionar o Comercial

**Cenário 4 – Refinamento após resposta parcial**

Dado que o atendente pergunte "como funciona a devolução?" e receba uma resposta sobre o prazo geral
Quando reformular para "e no caso de carga perigosa?"
Então o assistente deve complementar a resposta com as exceções da seção 3.2 da POL-001, mantendo o contexto da conversa

**Cenário 5 – Publicação de novo documento fora do horário comercial**

Dado que a área de Compliance publique uma versão atualizada da POL-001 às 22h de uma sexta-feira
Quando o sistema processar o documento
Então o conteúdo atualizado deve estar disponível para consulta no assistente até as 22h do sábado, com notificação enviada à equipe responsável ao término do processamento

---