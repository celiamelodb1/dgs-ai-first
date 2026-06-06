## PBI 04 — Regras de comportamento do assistente quando não há resposta segura

### 1. Contexto

A base de documentos do assistente da NovaTech contém situações que exigem comportamento específico: há documentos em versões conflitantes sem hierarquia formal definida, temas relevantes para o atendimento sem cobertura em documento oficial, e um FAQ interno do time que não foi validado pelo Compliance. Nesse cenário, a forma como o assistente se comporta quando não possui resposta segura é tão importante quanto a qualidade das respostas que fornece.

Três regras de comportamento devem ser implementadas para garantir que o atendente nunca receba uma resposta apresentada com segurança quando a informação de base é insuficiente, conflitante ou não validada: proibição de combinar fontes de qualidade diferente sem declarar isso; declaração obrigatória quando o tema não possui cobertura em documento oficial; e diferenciação clara entre respostas de graus de confiança distintos. Estas regras se aplicam a toda resposta retornada pelo assistente e têm precedência sobre qualquer outra lógica de geração de resposta.

### 2. Premissas e Riscos

**2.1 Premissas**
- O FAQ interno está identificado na base com o status de "não validado pelo Compliance" e não deve ser tratado como equivalente aos documentos oficiais em nenhuma circunstância.
- Os conflitos documentais ativos da base NovaTech estão mapeados: PROC-042 v1 e v2 coexistem com valores distintos de multiplicadores regionais, fatores de peso e prazo adicional de entrega. O critério de aplicação por período está definido: PROC-042 v1 para chamados abertos antes de 01/12/2023 ainda em processamento; PROC-042 v2 para chamados novos a partir dessa data, conforme seção 5 da PROC-042 v2.
- Os temas sem cobertura em documento formal estão catalogados: seguro de carga, política de carga danificada em trânsito, frete abaixo de 500kg e procedimento formal da Gestão de Riscos para cargas perigosas.
- A equipe responsável pela manutenção do catálogo de temas sem cobertura é a equipe de projeto da DB1, com participação das áreas de Operações, Compliance e Comercial. Apenas membros dessa equipe podem incluir, editar ou remover temas do catálogo manualmente. A inclusão automática ocorre exclusivamente via confirmação de feedback conforme requisito 4.5.
- O critério para classificar uma resposta no grau Atenção é: presença de ao menos uma das condições — conflito com outra versão ativa do mesmo documento, documento em processo de revisão pela área responsável, ou cobertura parcial em que o documento recuperado não responde a todas as dimensões da pergunta. O critério para grau Sem cobertura segura é: ausência de documento formal na base sobre o tema ou cobertura exclusiva pelo FAQ informal não validado.

**2.2 Riscos**
- Sinalizar incerteza com excesso de frequência em respostas que poderiam ser classificadas como Confirmado pode gerar desconfiança do atendente na ferramenta.
- Se novos gaps documentais surgirem sem serem incorporados ao catálogo, o assistente passará a não reconhecê-los e poderá responder com base no FAQ informal sem sinalização.
- A diferenciação visual entre os graus de confiança pode ser ignorada em situações de pressão de SLA se não estiver suficientemente destacada na interface.
- A ausência de atualização do catálogo após resolução de um gap — quando o documento formal for criado — pode fazer o assistente continuar sinalizando ausência de cobertura para tema que já possui documento válido.

### 3. Diretório

Assistente NovaTech → Configuração → Regras de comportamento da resposta

*(Ajustar conforme o menu real do sistema)*

### 4. Requisitos

**4.1 Regra 1 — Proibição de combinar fontes de qualidade diferente sem declaração**

O assistente não deve combinar trechos de documentos oficiais com conteúdo do FAQ informal em uma mesma resposta sem informar explicitamente ao atendente quais fontes foram utilizadas e qual é o status de cada uma.

Quando a resposta utilizar o FAQ como fonte, o assistente deve exibir a seguinte declaração antes de qualquer conteúdo informativo, substituindo os termos entre colchetes pelos valores correspondentes ao caso:

*"Atenção: esta resposta inclui informação do FAQ interno do time de atendimento, que não foi validado pelo Compliance e pode estar desatualizado. A parte sobre [tema] vem exclusivamente do FAQ. Recomendo confirmar com [área responsável conforme mapeamento da premissa 2.1 do PBI 02] antes de responder ao cliente."*

Quando houver documentos em versões conflitantes para o mesmo tema e o critério de aplicação por período não for resolvível automaticamente a partir da data de abertura do chamado, o assistente deve apresentar as informações das duas versões com os respectivos dados de identificação e o critério de uso de cada versão, sem selecionar silenciosamente apenas uma delas. Esta resposta deve ser classificada como grau Atenção.

Quando o critério de aplicação por período for resolvível automaticamente — como no caso da PROC-042, cuja data de corte é 01/12/2023 — o assistente deve aplicar a versão correta diretamente e classificar a resposta como grau Confirmado, informando ao atendente qual versão foi aplicada e o motivo.

**4.2 Regra 2 — Declaração obrigatória de ausência de cobertura documental**

O assistente deve verificar o catálogo de temas sem cobertura em documento formal e reconhecer automaticamente quando uma pergunta se enquadra em um desses temas, mesmo que a pergunta seja formulada de forma diferente das que originaram o mapeamento. O critério de enquadramento é a proximidade semântica entre a pergunta e o tema catalogado, avaliada pelo mesmo mecanismo de busca utilizado para recuperar trechos de documentos.

Quando a pergunta recair sobre tema catalogado como sem cobertura documental formal, o assistente deve retornar a seguinte declaração, substituindo os termos entre colchetes pelos valores correspondentes:

*"Não há documento oficial na base sobre [tema]. A única referência disponível é o FAQ interno [item e descrição resumida], que não foi validado pelo Compliance. Para responder ao cliente com segurança, acione [área responsável conforme mapeamento]. Registre a ausência no formulário de feedback para priorização pela equipe responsável."*

O assistente não deve retornar ausência de resposta sem explicar o motivo — informar ao atendente que o tema não possui cobertura documental é uma resposta operacionalmente útil e obrigatória.

**4.3 Regra 3 — Diferenciação obrigatória entre graus de confiança**

O assistente deve classificar e sinalizar cada resposta em um dos três graus, com diferenciação visual e textual clara, conforme os critérios definidos na premissa 2.1:

**Confirmado:** a resposta é baseada exclusivamente em documento oficial com versão controlada, responsável identificado e data de atualização, sem conflito com outros documentos da base. O assistente apresenta a resposta diretamente, seguida da identificação completa da fonte.

**Atenção:** a resposta é baseada em documento oficial, mas há ao menos uma das seguintes condições: conflito com outra versão ativa do mesmo documento sem critério de período resolvível automaticamente, documento em processo de revisão pela área responsável, ou cobertura parcial que não abrange todas as dimensões da pergunta. O assistente exibe sinalização visual de atenção e descreve a condição que reduz a confiança antes de apresentar o conteúdo.

**Sem cobertura segura:** ausência de documento formal na base sobre o tema, ou cobertura exclusiva pelo FAQ informal não validado. O assistente apresenta o diagnóstico da ausência, o que existe na base sobre o tema e a indicação da área responsável para o atendente acionar — sem apresentar conteúdo informativo como se fosse resposta confirmada.

**4.4 Catálogo de temas sem cobertura documental**

O sistema deve manter catálogo de temas reconhecidos como sem cobertura em documento formal. A manutenção manual do catálogo — inclusão, edição e remoção de temas — é restrita à equipe de projeto da DB1 com participação das áreas responsáveis. Os temas iniciais a serem cadastrados são:
- Seguro de carga: percentuais e condições — cobertura apenas no FAQ item 22
- Carga danificada em trânsito: procedimento de registro e reembolso — cobertura apenas no FAQ item 38
- Frete padrão para cargas abaixo de 500kg — sem cobertura na base
- Procedimento formal da Gestão de Riscos para devoluções de carga perigosa — referência apenas ao ramal 4500 na POL-001 seção 3.2, sem procedimento detalhado

Quando um tema do catálogo receber documento formal criado pela área responsável e esse documento for indexado na base, o tema deve ser removido do catálogo pela equipe responsável para que o assistente passe a respondê-lo com base no novo documento.

**4.5 Atualização automática do catálogo por feedback confirmado**

Quando um registro de feedback do tipo "tema sem cobertura" for confirmado pela equipe de projeto da DB1, o tema deve ser incorporado automaticamente ao catálogo para que o assistente passe a reconhecê-lo nas consultas seguintes. A confirmação deve ser registrada com identificação do membro da equipe que a realizou e a data de confirmação.

### 5. Critérios de Aceite

**CA01 – Declaração de fonte antes do conteúdo quando FAQ for utilizado**
Garantir que o assistente exiba a declaração de status da fonte antes de qualquer conteúdo informativo sempre que o FAQ interno for utilizado na composição da resposta, sem exceção.

**CA02 – Apresentação das duas versões em conflito sem escolha silenciosa**
Garantir que o assistente apresente os valores das duas versões conflitantes com seus dados de identificação e o critério de uso por período quando o conflito não for resolvível automaticamente, sem selecionar uma versão sem informar o atendente.

**CA03 – Aplicação automática da versão correta quando critério de período for resolvível**
Garantir que o assistente aplique a versão correta do documento automaticamente quando o critério de período for resolvível a partir da data de abertura do chamado, informando ao atendente qual versão foi aplicada e o motivo, classificando a resposta como grau Confirmado.

**CA04 – Declaração de ausência de cobertura para temas catalogados**
Garantir que o assistente retorne a declaração de ausência no formato definido, com indicação da área responsável e orientação de registro de feedback, sempre que a pergunta envolver tema catalogado como sem cobertura documental, independentemente da forma como a pergunta for formulada.

**CA05 – Toda resposta classificada em um dos três graus de confiança**
Garantir que toda resposta retornada pelo assistente esteja classificada em um dos três graus — Confirmado, Atenção ou Sem cobertura segura — com sinalização visual distinta entre eles.

**CA06 – Grau sem cobertura segura sem conteúdo informativo apresentado como resposta**
Garantir que o assistente não apresente conteúdo informativo como resposta quando a classificação for Sem cobertura segura, exibindo apenas o diagnóstico da ausência, o que existe na base sobre o tema e a área a acionar.

**CA07 – Manutenção do catálogo restrita à equipe autorizada**
Garantir que a inclusão, edição e remoção manual de temas do catálogo de ausências documentais seja restrita aos membros da equipe de projeto da DB1 com participação das áreas responsáveis, impedindo alterações por outros perfis de usuário.

**CA08 – Atualização automática do catálogo após confirmação de feedback**
Garantir que o catálogo seja atualizado automaticamente após a confirmação de feedback do tipo "tema sem cobertura" pela equipe de projeto, com registro da data e do responsável pela confirmação, fazendo o assistente reconhecer o novo tema nas consultas seguintes.

**CA09 – Remoção de tema do catálogo após criação de documento formal**
Garantir que quando um tema catalogado receber documento formal indexado na base, a equipe responsável consiga removê-lo do catálogo para que o assistente passe a respondê-lo com base no novo documento, sem continuar sinalizando ausência de cobertura.

### 6. Cenários

**Cenário 1 – Pergunta sobre seguro de carga**

Dado que o atendente pergunte "qual o percentual de seguro de carga para contrato padrão?"
Quando o assistente identificar que o tema está catalogado como sem cobertura documental formal
Então deve retornar a declaração de ausência indicando que a única referência é o FAQ interno item 22, sem validação do Compliance, orientar o atendente a acionar o Comercial e classificar a resposta como grau Sem cobertura segura

**Cenário 2 – Pergunta sobre frete com versões conflitantes sem critério resolvível automaticamente**

Dado que o atendente pergunte "qual o multiplicador de frete para o Nordeste?" sem informar a data do chamado
Quando o assistente não conseguir resolver o critério de período automaticamente por falta da data de abertura do chamado
Então deve classificar a resposta como grau Atenção, apresentar os dois valores com identificação de cada versão — v1: 1,4 e v2: 1,5 — e orientar o atendente a informar a data de abertura do chamado para aplicação do critério correto

**Cenário 3 – Pergunta sobre frete com critério de período resolvível automaticamente**

Dado que o atendente pergunte "qual o multiplicador de frete para o Nordeste em um chamado aberto hoje?"
Quando o assistente identificar que a data de abertura é posterior a 01/12/2023
Então deve aplicar a PROC-042 v2 automaticamente, retornar o multiplicador 1,5, classificar a resposta como grau Confirmado e informar ao atendente que a versão v2 foi aplicada por ser a vigente para chamados a partir de 01/12/2023

**Cenário 4 – Pergunta com resposta confirmada em documento oficial**

Dado que o atendente pergunte "qual o prazo para o cliente solicitar devolução de mercadoria padrão?"
Quando o assistente encontrar a informação na POL-001 v3.1 seção 3.1 sem conflito com outros trechos da base
Então deve classificar a resposta como grau Confirmado e retornar que o prazo é de 7 dias úteis, com indicação do documento, versão, seção e data de atualização

**Cenário 5 – Novo tema sem cobertura identificado via feedback e posterior criação de documento**

Dado que três atendentes registrem feedback indicando ausência de documento formal sobre o mesmo tema
Quando a equipe de projeto confirmar a ausência e o tema for incorporado ao catálogo automaticamente
E posteriormente a área responsável criar o documento formal e ele for indexado na base
Então a equipe responsável deve remover o tema do catálogo para que o assistente passe a respondê-lo com base no novo documento, sem continuar sinalizando ausência de cobertura