Vou fazer dois diagramas: o primeiro mostra os 5 documentos e seus temas principais; o segundo destaca os pontos de conflito e lacunas — que são a parte mais crítica para o projeto RAG.Agora o segundo diagrama, focado exclusivamente nos pontos de atrito — conflitos entre documentos e lacunas sem cobertura formal:Os diagramas são clicáveis — cada bloco dispara uma pergunta relacionada ao tema para continuar a conversa.

Alguns pontos que valem destaque para o projeto:

**Para o pipeline RAG:** os quatro conflitos em vermelho são casos em que o assistente **não pode simplesmente recuperar e apresentar** — ele precisa sinalizar ativamente a divergência. Sem isso, vai ora dar o valor da v1, ora o da v2, dependendo de qual chunk for recuperado, gerando exatamente o tipo de inconsistência que o projeto quer eliminar.

**Para a governança da NovaTech:** as quatro lacunas em coral são oportunidades de formalização. O FAQ do item 38 (carga danificada) e o item 22 (seguro) descrevem processos que claramente existem na prática — só não têm documento normativo. O assistente pode ser o catalisador para a NovaTech perceber isso e endereçar antes do go-live.

**Recomendação prática:** antes de iniciar a ingestão, entregar esse mapa para as áreas de Operações, Compliance e Comercial com uma pergunta simples — "qual é a versão correta de cada item em conflito?" — e registrar as respostas como decisão formal. Isso resolve os conflitos antes que o RAG os herde.