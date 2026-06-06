## PBI 02 — Tratamento de respostas de baixa confiança ou discordância do atendente

### 1. Contexto

Durante o atendimento, dois cenários exigem que o chamado não avance para a resposta ao cliente sem validação prévia: o assistente pode indicar que não encontrou documentação suficiente para responder com segurança, ou o atendente pode discordar da resposta recebida com base em sua experiência ou percepção de desatualização.

A discordância do atendente é acionada por um botão específico na interface do assistente, separado da ação de sinalizar erro do PBI 03. A distinção entre os dois fluxos é: no PBI 02 o atendente não consegue identificar o documento correto por conta própria e precisa de orientação — a dúvida é sobre qual informação usar. No PBI 03 o atendente já identificou o documento correto e sabe que a resposta da IA está errada — a certeza é sobre o erro.

Em ambos os casos do PBI 02, o sistema deve oferecer ao atendente um caminho estruturado — reformular a pergunta ou acionar a área responsável — e garantir que a orientação recebida seja registrada antes de a resposta chegar ao cliente.

### 2. Premissas e Riscos

**2.1 Premissas**
- O assistente indica explicitamente ao atendente quando não encontrou cobertura documental suficiente para a pergunta. O critério para essa sinalização é: ausência de trechos de documentos formais recuperados sobre o tema, ou presença de trechos de versões conflitantes sem critério de aplicação resolvível automaticamente pelo sistema.
- O atendente pode acionar discordância de qualquer resposta do assistente, independentemente do grau de confiança exibido, usando o botão "Discordo desta resposta" na interface.
- A orientação recebida da área responsável deve ser registrada por escrito no chamado antes de ser usada como base da resposta ao cliente. Orientações recebidas verbalmente não são aceitas como registro válido.
- O supervisor do atendente é responsável pela decisão final quando a área responsável não retornar dentro do prazo do SLA do chamado.
- A equipe responsável pela gestão da base documental é formada pelos representantes das áreas de Operações, Compliance e Comercial, coordenada pela equipe de projeto da DB1.
- O mapeamento de tema para área responsável está cadastrado no sistema conforme a tabela: prazos e procedimentos operacionais → Operações; frete, SLA e contratos → Comercial; regras normativas e compliance → Compliance. Temas não mapeados são encaminhados ao supervisor imediato do atendente.

**2.2 Riscos**
- O atendente pode acionar a área responsável em casos resolvíveis por reformulação da pergunta ao assistente, gerando sobrecarga desnecessária nas áreas.
- Chamados de incidentes críticos de clientes Gold não pausam o SLA fora do horário comercial — a validação humana não pode ser sequencial à tentativa de reformulação nesses casos.
- Orientações recebidas verbalmente, sem registro escrito no chamado, impedem auditoria posterior e não contribuem para a melhoria da documentação.
- O formulário de feedback ao encerramento é condição obrigatória — sua ausência impede o fechamento do chamado que passou por validação humana.

### 3. Diretório

Assistente NovaTech → Painel de atendimento → Consulta à documentação → Resposta com sinalização de incerteza

*(Ajustar conforme o menu real do sistema)*

### 4. Requisitos

**4.1 Sinalização de incerteza pelo assistente**

Quando o assistente não encontrar cobertura documental suficiente para a pergunta — por ausência de trechos formais recuperados ou por presença de versões conflitantes sem critério de aplicação resolvível automaticamente — deve indicar ao atendente o motivo da incerteza entre as situações possíveis:
- Tema sem documento formal na base
- Documentos com versões em conflito para o mesmo tema sem critério de período aplicável automaticamente
- Documento existente que cobre o tema apenas parcialmente

**4.2 Acionamento de discordância pelo atendente**

O sistema deve disponibilizar o botão "Discordo desta resposta" em toda resposta retornada pelo assistente. Ao acionar esse botão, o atendente deve descrever em campo de texto o motivo da discordância. O sistema registra a discordância e inicia o mesmo fluxo de validação descrito nos requisitos seguintes.

Este fluxo se aplica quando o atendente não consegue identificar por conta própria o documento correto e precisa de orientação. Quando o atendente já identificou o documento correto e sabe que a resposta está errada, deve usar o fluxo de sinalização de erro definido no PBI 03.

**4.3 Bloqueio de avanço sem validação registrada**

Enquanto o chamado estiver com resposta pendente de validação, o sistema deve impedir o registro da resposta ao cliente, liberando o avanço somente após o atendente registrar a fonte utilizada para validar a informação — seja por reformulação bem-sucedida, seja por orientação da área responsável.

**4.4 Reformulação da pergunta antes da escalada**

O sistema deve permitir que o atendente reformule a pergunta ao assistente — especificando o documento, a versão ou o período do chamado — antes de acionar a área responsável.

Se a reformulação retornar resposta com grau Confirmado conforme definido no PBI 04, o chamado deve retornar ao fluxo normal, com o atendente registrando o documento utilizado conforme exigido pelo CA07 do PBI 01.

**4.5 Encaminhamento para área responsável**

O sistema deve indicar ao atendente a área responsável pelo tema conforme o mapeamento cadastrado na premissa 2.1:
- Prazos e procedimentos operacionais → Operações
- Frete, SLA e contratos → Comercial
- Regras normativas e compliance → Compliance
- Temas não mapeados ou discordância com base em prática conhecida → supervisor imediato

**4.6 Registro obrigatório da orientação recebida**

O sistema deve exigir que o atendente registre no chamado a orientação recebida da área responsável — por escrito, via campo de texto no chamado — antes de permitir o avanço para a resposta ao cliente. O sistema deve bloquear o avanço enquanto esse campo estiver vazio.

**4.7 Registro obrigatório para atualização da documentação**

Ao encerrar um chamado que passou por validação humana, o sistema deve exigir o preenchimento do formulário de feedback como condição para o encerramento do chamado, com os campos: pergunta feita ao assistente, motivo da incerteza ou discordância, área acionada, orientação recebida e sugestão de atualização documental. O chamado não pode ser encerrado sem o preenchimento desse formulário.

**4.8 Tratamento de incidentes críticos Gold com incerteza simultânea**

Quando o chamado for classificado como incidente crítico de cliente Gold e o assistente sinalizar incerteza ou o atendente acionar discordância, o sistema deve permitir que o atendente inicie simultaneamente a reformulação da pergunta e a escalada para a área responsável, sem que uma etapa bloqueie a outra. A contagem do SLA não deve ser pausada durante esse processo.

### 5. Critérios de Aceite

**CA01 – Sinalização de incerteza com motivo identificado**
Garantir que o assistente informe ao atendente quando não encontrar cobertura documental suficiente para a pergunta, indicando o motivo antes de qualquer conteúdo informativo.

**CA02 – Botão de discordância disponível em toda resposta**
Garantir que o botão "Discordo desta resposta" esteja disponível em toda resposta retornada pelo assistente, permitindo ao atendente iniciar o fluxo de validação a partir de qualquer resposta recebida.

**CA03 – Impedimento de resposta ao cliente sem validação registrada**
Garantir que o sistema bloqueie o registro da resposta ao cliente enquanto o chamado estiver com validação pendente, liberando o avanço somente após o atendente registrar a fonte de validação utilizada.

**CA04 – Retorno ao fluxo normal após reformulação com resposta confirmada**
Garantir que o chamado retorne ao fluxo normal de atendimento quando o atendente reformular a pergunta e o assistente retornar resposta com grau Confirmado, exigindo o registro do documento utilizado conforme CA07 do PBI 01.

**CA05 – Indicação da área responsável conforme mapeamento cadastrado**
Garantir que o sistema indique ao atendente a área responsável correspondente ao tema do chamado conforme o mapeamento definido na premissa 2.1, e encaminhe ao supervisor imediato quando o tema não estiver mapeado.

**CA06 – Bloqueio de avanço sem orientação registrada por escrito**
Garantir que o sistema exija o registro escrito da orientação recebida no campo do chamado como condição obrigatória para o atendente prosseguir com a resposta ao cliente, impedindo o avanço com campo vazio.

**CA07 – Encerramento bloqueado sem formulário de feedback preenchido**
Garantir que o sistema impeça o encerramento de chamados que passaram por validação humana quando o formulário de feedback não estiver preenchido pelo atendente.

**CA08 – Escalada e reformulação simultâneas em incidentes críticos Gold**
Garantir que em chamados classificados como incidentes críticos de clientes Gold o atendente possa iniciar a escalada para a área responsável e reformular a pergunta ao assistente ao mesmo tempo, sem que o sistema bloqueie uma ação enquanto a outra estiver em andamento e sem pausar a contagem do SLA.

### 6. Cenários

**Cenário 1 – Reformulação resolve o conflito entre versões**

Dado que o atendente pergunte "qual o multiplicador de frete para o Nordeste?"
E o assistente sinalize conflito entre PROC-042 v1 (1,4) e PROC-042 v2 (1,5) sem critério de período resolvível automaticamente
Quando o atendente reformule especificando que o chamado foi aberto após 01/12/2023
Então o assistente deve retornar o multiplicador 1,5 da PROC-042 v2 com grau Confirmado, o atendente deve registrar a fonte e o chamado deve retornar ao fluxo normal sem escalada

**Cenário 2 – Escalada por tema sem documento formal**

Dado que o atendente pergunte "qual o percentual de seguro de carga para contrato padrão?"
E o assistente identifique que o tema não possui documento formal na base
Quando o assistente sinalizar a ausência de cobertura
Então o sistema deve indicar o Comercial como área responsável conforme mapeamento, registrar a pendência no chamado e bloquear a resposta ao cliente até o registro da orientação recebida

**Cenário 3 – Discordância do atendente com prática conhecida**

Dado que o atendente receba uma resposta sobre prazo de frete especial indicando mais 2 dias úteis com base na PROC-042 v1
E o atendente saiba que a prática vigente é de mais 3 dias úteis conforme PROC-042 v2, mas não consiga localizar o documento por conta própria
Quando o atendente acionar o botão "Discordo desta resposta" e descrever o motivo
Então o sistema deve indicar o Comercial como área responsável, bloquear a resposta ao cliente e exigir o registro escrito da orientação recebida antes de prosseguir

**Cenário 4 – Incidente crítico Gold com incerteza simultânea**

Dado que o chamado seja classificado como incidente crítico de cliente Gold com SLA de 30 minutos para primeira resposta
E o assistente sinalize incerteza na resposta sobre o tema consultado
Então o sistema deve permitir que o atendente reformule a pergunta e acione a área responsável ao mesmo tempo, sem pausar o SLA, registrando ambas as ações no chamado

---