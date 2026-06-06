## PBI 03 — Sinalização de resposta incorreta, desatualizada ou incompleta ao assistente

### 1. Contexto

O assistente pode retornar respostas com aparente segurança que, ao serem analisadas pelo atendente, revelam-se incorretas, baseadas em versões desatualizadas de documentos, ou omissas em relação a regras e exceções relevantes para o caso do cliente.

Este fluxo se aplica quando o atendente já identificou o documento correto e tem certeza de que a resposta do assistente está errada. Quando o atendente percebe um problema mas não consegue identificar o documento correto por conta própria, deve usar o botão "Discordo desta resposta" do fluxo definido no PBI 02.

O sistema deve oferecer um caminho estruturado para que o atendente registre o problema com precisão, bloqueie o uso da resposta incorreta e contribua para a correção da base de documentos do assistente. A equipe responsável pelo tratamento dos feedbacks é formada pelos representantes das áreas de Operações, Compliance e Comercial, coordenada pela equipe de projeto da DB1, com prazo de resolução de até 5 dias úteis após o recebimento do registro.

### 2. Premissas e Riscos

**2.1 Premissas**
- O atendente é capaz de identificar divergências entre a resposta do assistente e a documentação oficial que conhece, e consegue identificar o documento correto com nome, versão e seção.
- Todo registro de problema deve identificar o documento correto com nome, versão e seção — registros sem essa referência não são aceitos pelo sistema.
- O FAQ interno do time de atendimento não é fonte válida para embasar a correção de uma resposta do assistente.
- Registros do mesmo tipo de problema para o mesmo documento e versão são consolidados em uma única ocorrência com contagem de incidências, independentemente do tipo de problema reportado — incorreta, desatualizada ou incompleta.
- A equipe responsável tem prazo de até 5 dias úteis para resolver e fechar cada registro de problema recebido.
- A equipe responsável pela gestão da base documental é formada pelos representantes das áreas de Operações, Compliance e Comercial, coordenada pela equipe de projeto da DB1.

**2.2 Riscos**
- Registros de problema sem identificação do documento correto sobrecarregam a equipe responsável sem resultado — por isso o sistema bloqueia o envio sem esse preenchimento.
- Se o atendente identificar o erro somente após já ter comunicado a informação incorreta ao cliente, o chamado exige avaliação do supervisor para definição de contato corretivo. Erros de inversão de regra — como informar que carga perigosa pode ser devolvida pelo processo padrão — devem ser priorizados pela equipe responsável sobre demais tipos de problema.
- A ausência de notificação de resolução ao atendente que reportou o problema desestimula novos registros e enfraquece o ciclo de melhoria.

### 3. Diretório

Assistente NovaTech → Painel de atendimento → Consulta à documentação → Sinalizar problema na resposta

*(Ajustar conforme o menu real do sistema)*

### 4. Requisitos

**4.1 Ação de sinalização disponível em toda resposta**

O assistente deve disponibilizar, em toda resposta retornada ao atendente, o botão "Sinalizar problema nesta resposta" — visível diretamente na interface, sem necessidade de abrir outro sistema. Este botão é distinto do botão "Discordo desta resposta" do PBI 02 e deve ser usado quando o atendente já identificou o documento correto e tem certeza do erro.

**4.2 Formulário de registro do problema**

Ao acionar a sinalização, o sistema deve exibir formulário com os seguintes campos obrigatórios:
- Tipo de problema: incorreta, desatualizada ou incompleta
- O que o assistente disse versus o que o documento correto estabelece
- Documento correto: nome, versão e seção
- Documento utilizado pelo assistente: nome e versão
- Situação do chamado: problema identificado antes ou depois de comunicar o cliente

O sistema deve bloquear o envio do formulário quando o campo "documento correto" não estiver preenchido com nome, versão e seção.

**4.3 Bloqueio da resposta sinalizada**

Após o registro do problema, o sistema deve bloquear o uso da resposta original para o cliente e exigir que o atendente registre a fonte correta antes de prosseguir com o atendimento.

**4.4 Orientação para localização da fonte correta**

O sistema deve orientar o atendente a buscar a informação correta diretamente nos documentos normativos, na seguinte ordem de prioridade: políticas (POL) → procedimentos (PROC) → tabelas de SLA → Confluence. O FAQ interno não deve ser apresentado como opção válida de fonte para correção.

Se o atendente não conseguir localizar a informação por conta própria, o sistema deve oferecer o encaminhamento para a área responsável pelo tema conforme o mapeamento definido na premissa 2.1 do PBI 02.

**4.5 Vinculação do registro ao chamado**

O registro do problema deve ser automaticamente vinculado ao número do chamado em curso para rastreabilidade.

**4.6 Consolidação de registros do mesmo documento e versão**

Quando um problema for reportado para um documento e versão que já possui registro aberto na fila de revisão, o sistema deve consolidar a nova ocorrência no registro existente, incrementando a contagem de incidências, independentemente do tipo de problema reportado pelo novo atendente.

**4.7 Priorização de erros de inversão de regra**

Registros classificados como incorretos em que a resposta do assistente inverte uma regra proibitiva — como informar que carga perigosa pode ser devolvida pelo processo padrão, contrariando a POL-001 seção 3.2 — devem ser automaticamente sinalizados como prioritários na fila de revisão da equipe responsável.

**4.8 Prazo de resolução e notificação ao atendente**

A equipe responsável tem prazo de até 5 dias úteis para resolver cada registro de problema recebido. Ao concluir a resolução, o sistema deve notificar o atendente que originou o registro informando o encerramento e a ação tomada na base de documentos.

**4.9 Tratamento de erro comunicado ao cliente**

Quando o campo "situação do chamado" do formulário indicar que o cliente já foi comunicado com a informação incorreta, o sistema deve notificar automaticamente o supervisor do atendente para avaliação da necessidade de contato corretivo com o cliente.

### 5. Critérios de Aceite

**CA01 – Botão de sinalização disponível e distinto do botão de discordância**
Garantir que o botão "Sinalizar problema nesta resposta" esteja visível em toda resposta retornada pelo assistente, separado e visualmente distinto do botão "Discordo desta resposta" do PBI 02.

**CA02 – Envio bloqueado sem documento correto identificado**
Garantir que o sistema bloqueie o envio do formulário de registro de problema quando o campo "documento correto" não estiver preenchido com nome, versão e seção.

**CA03 – Bloqueio do uso da resposta sinalizada**
Garantir que o sistema impeça o uso da resposta sinalizada para comunicar o cliente, exigindo o registro da fonte correta como condição para prosseguir com o atendimento.

**CA04 – FAQ excluído como opção de fonte para correção**
Garantir que o FAQ interno não seja apresentado pelo sistema como opção válida de fonte ao atendente que estiver localizando o documento correto após sinalizar um problema.

**CA05 – Registro vinculado automaticamente ao chamado**
Garantir que todo registro de problema seja automaticamente vinculado ao número do chamado em curso no momento do envio do formulário.

**CA06 – Consolidação de ocorrências do mesmo documento e versão**
Garantir que o sistema consolide em um único registro os problemas reportados para o mesmo documento e versão, incrementando a contagem de incidências independentemente do tipo de problema reportado, sem criar entradas duplicadas na fila de revisão.

**CA07 – Priorização automática de inversão de regra**
Garantir que registros classificados como incorretos em que a resposta inverte uma regra proibitiva sejam automaticamente marcados como prioritários na fila de revisão da equipe responsável.

**CA08 – Notificação ao atendente dentro do prazo de resolução**
Garantir que o atendente que originou o registro receba uma notificação de encerramento em até 5 dias úteis após o envio do formulário, informando a ação tomada na base de documentos.

**CA09 – Notificação ao supervisor quando cliente já foi comunicado**
Garantir que o sistema notifique automaticamente o supervisor do atendente quando o formulário indicar que o cliente já foi comunicado com a informação incorreta, para avaliação de contato corretivo.

### 6. Cenários

**Cenário 1 – Multiplicador de frete desatualizado**

Dado que o assistente retorne o multiplicador regional Norte como 1,6 com base na PROC-042 v1
E o chamado tenha sido aberto após 01/12/2023, quando o correto é 1,8 conforme PROC-042 v2, seção 2.1
Quando o atendente acionar "Sinalizar problema nesta resposta" e registrar o problema como desatualizado com referência à PROC-042 v2 seção 2.1
Então o sistema deve bloquear o uso do valor 1,6, registrar a ocorrência vinculada ao chamado e permitir o prosseguimento com o valor correto sem comunicar o erro ao cliente

**Cenário 2 – Resposta incompleta sobre exceções de devolução**

Dado que o assistente retorne apenas o prazo de 7 dias úteis da POL-001 seção 3.1 sem mencionar a exceção de cargas perigosas da seção 3.2
Quando o atendente identificar a omissão porque o chamado envolve carga perigosa e registrar o problema como incompleto indicando a POL-001 v3.1 seção 3.2
Então o sistema deve bloquear a resposta parcial, registrar a ocorrência e exigir que o atendente registre a seção 3.2 como complemento antes de prosseguir

**Cenário 3 – Inversão de regra sobre carga perigosa**

Dado que o assistente retorne que carga perigosa pode ser devolvida pelo processo padrão
E a POL-001 v3.1 seção 3.2 estabelece que cargas perigosas não são elegíveis para devolução padrão
Quando o atendente registrar o problema como incorreto com referência à POL-001 v3.1 seção 3.2
Então o sistema deve bloquear a resposta, marcar o registro como prioritário na fila de revisão e notificar a equipe responsável para resolução em caráter urgente

**Cenário 4 – Erro identificado após comunicação com o cliente**

Dado que o atendente identifique o erro somente após já ter comunicado a informação incorreta ao cliente
Quando preencher o formulário com a situação "cliente já comunicado"
Então o sistema deve notificar automaticamente o supervisor do atendente para avaliação da necessidade de contato corretivo com o cliente

---