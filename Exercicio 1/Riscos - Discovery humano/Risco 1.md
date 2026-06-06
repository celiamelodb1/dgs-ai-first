**Risco 1 — Clientes sendo negados desconto ao qual têm direito (item 45 do FAQ × PROC-042 v2)**

O FAQ instrui o atendente a aplicar desconto apenas para clientes com mais de 10 fretes especiais por mês. A PROC-042 v2, vigente desde novembro de 2023, reduziu esse threshold para 8 fretes e definiu percentuais explícitos: 5% a partir de 8 fretes, 10% acima de 15. O FAQ nunca foi atualizado.

Isso significa que hoje, independentemente do RAG, qualquer cliente com 8 ou 9 fretes especiais mensais está sendo negado um benefício previsto no procedimento mais recente. O problema não está esperando o assistente ser construído — já está acontecendo.

**Como levaria ao discovery humano**

Não apresentaria isso como "encontramos uma inconsistência documental". Apresentaria como uma pergunta de negócio com impacto financeiro estimável:

Solicitaria ao Comercial uma extração simples: quantos clientes ativos têm entre 8 e 10 fretes especiais por mês nos últimos 6 meses. Com esse número, multiplicaria pelo valor médio de frete e pelos 5% de desconto não aplicado. O resultado é o custo mensal da inconsistência — valor que torna a conversa concreta e urgente.

A pergunta para o Comercial seria direta: "A regra vigente para desconto por volume é a da PROC-042 v2, com threshold em 8 fretes e percentuais de 5% e 10%?" Se a resposta for sim, o FAQ precisa ser corrigido imediatamente e o time de atendimento precisa ser comunicado — antes de qualquer linha de código do RAG ser escrita. Se a resposta for não, existe uma decisão não documentada que precisa ser formalizada antes da ingestão.

O ponto crítico a deixar claro na conversa: o RAG vai perpetuar e escalar essa inconsistência. Hoje ela afeta os atendimentos que passam pelo FAQ. Com o assistente, afetará todos os atendimentos sobre desconto, com aparência de resposta oficial.
