Quero que você edite e evolua o arquivo Draw.io/XML atualmente aberto.

IMPORTANTE:
- O desenho atual foi produzido por mim durante uma entrevista de System Design para uma vaga de Engenheiro de IA Sênior.
- Quero PRESERVAR a ideia original e os componentes que fazem sentido.
- Não quero simplesmente apagar tudo e criar outro diagrama genérico.
- Quero transformar esse desenho em um case completo de estudo de "System Design com IA".
- Corrija os erros conceituais existentes.
- Adicione componentes faltantes somente quando houver justificativa arquitetural.
- Evite overengineering.
- O resultado precisa continuar sendo um arquivo .drawio válido e totalmente editável.
- Organize visualmente o diagrama para que seja possível estudá-lo posteriormente.
- Use nomes claros e preferencialmente em português, mantendo termos técnicos conhecidos em inglês quando fizer sentido.

==================================================
CONTEXTO DO SISTEMA
==================================================

Estamos projetando uma plataforma SaaS multi-tenant para clínicas.

Pacientes podem interagir através de:

- Web
- WhatsApp

O sistema deve permitir principalmente:

1. Fazer perguntas sobre informações da clínica.
2. Consultar especialidades.
3. Encontrar/recomendar profissionais adequados.
4. Consultar horários disponíveis.
5. Criar agendamentos.
6. Manter contexto de conversação.
7. Utilizar IA Generativa, agentes, RAG e tools.
8. Suportar múltiplas clínicas sem permitir vazamento de informações entre elas.
9. Evoluir futuramente para grande escala.

Cada clínica é um tenant independente.

Considere identificadores como:

tenantId
clinicId

Toda operação sensível a contexto precisa garantir isolamento entre tenants.

==================================================
PRIMEIRA CORREÇÃO CONCEITUAL IMPORTANTE
==================================================

No desenho atual existe uma mistura entre:

- pipeline de ingestão de documentos;
- pipeline de atendimento de perguntas.

CORRIJA ISSO.

A pergunta do usuário NÃO deve passar por chunking.

Existem dois fluxos diferentes.

FLUXO 1 — INGESTÃO SEMÂNTICA

Documentos da clínica
→ armazenamento
→ parsing
→ chunking
→ embeddings
→ indexação no Vector Database.

Cada chunk deve possuir metadados como:

tenantId
clinicId
documentId
documentType
version
createdAt
permissions/status quando necessário.

FLUXO 2 — CONSULTA/RUNTIME

Pergunta do usuário
→ Agent Service
→ geração do embedding da pergunta quando houver RAG
→ busca no Vector Database
→ filtro obrigatório por tenantId/clinicId
→ recuperação dos chunks relevantes
→ contexto enviado ao LLM
→ resposta.

Mostre esses dois pipelines visualmente separados.

==================================================
ARQUITETURA QUE DEVE SER EVOLUÍDA
==================================================

Organize o desenho aproximadamente nas seguintes camadas.

--------------------------------------------------
1. CHANNELS / CLIENTES
--------------------------------------------------

Paciente

→ Web
→ WhatsApp

Os dois canais convergem para a mesma camada de entrada.

--------------------------------------------------
2. API / EDGE LAYER
--------------------------------------------------

Adicionar uma camada como:

API Gateway / BFF

Responsabilidades:

- autenticação;
- autorização inicial;
- identificação do tenant/clinic;
- rate limiting;
- roteamento;
- correlation/trace ID;
- validações básicas.

Não coloque Load Balancer como uma caixa obrigatória se a infraestrutura/API Gateway já puder realizar isso.

Entretanto, represente ou anote que:

Agent Services e Backend Services devem poder escalar horizontalmente.

Adicionar anotação:

"Horizontal Scaling / Multiple Instances"

--------------------------------------------------
3. AGENT / AI ORCHESTRATION LAYER
--------------------------------------------------

Substituir a ideia central de "LLM API" por algo mais arquitetural como:

Agent Service / AI Orchestrator

Esse componente deve controlar:

- system prompt;
- agentes;
- tools;
- state;
- session;
- model selection;
- tool calling;
- RAG;
- limites de execução.

IMPORTANTE:

Não começar obrigatoriamente com multi-agentes.

Representar inicialmente:

Clinic Assistant Agent

com ferramentas especializadas.

Tools possíveis:

searchClinicKnowledge
findProfessionals
getAvailableSlots
createAppointment

Adicionar uma anotação indicando evolução possível:

"Se a complexidade justificar, evoluir para Supervisor + Specialized Agents"

Exemplos futuros:

Supervisor Agent
Knowledge Agent
Scheduling Agent
Professional Recommendation Agent

Não transformar tudo em multi-agent sem necessidade.

Mostrar explicitamente o trade-off:

Mais agentes =
+ especialização
+ isolamento de contexto
- mais tokens
- mais latência
- mais complexidade
- maior dificuldade de observabilidade

--------------------------------------------------
4. SESSION / STATE
--------------------------------------------------

Adicionar:

Session / State Store

Pode ser representado conceitualmente como Redis ou outro storage adequado.

Responsabilidades:

- conversationId/sessionId;
- estado atual do workflow;
- informações temporárias;
- contexto necessário para continuidade da conversa;
- checkpoints quando necessário.

Não assumir que a LLM "lembra" das coisas sozinha.

Mostrar que memória/state são persistidos pela aplicação.

--------------------------------------------------
5. TOOLS
--------------------------------------------------

As tools devem ser tratadas como adapters/capabilities.

Criar visualmente tools como:

Search Clinic Knowledge Tool
Find Professionals Tool
Get Available Slots Tool
Create Appointment Tool

Mostrar que:

Agent
→ Tool
→ Service/API determinística

A LLM decide QUANDO usar uma tool.

O backend/tool define COMO a operação acontece.

Adicionar anotação importante:

"LLM nunca é camada de autorização."

As tools/backend devem continuar verificando:

- autenticação;
- autorização;
- tenant;
- validação;
- regras de negócio.

--------------------------------------------------
6. RAG SERVICE
--------------------------------------------------

Criar um componente claro chamado:

RAG Service / Retrieval Service

Fluxo:

Agent
→ RAG Tool
→ Query Embedding
→ Vector Database
→ Metadata Filter
→ Top-K Relevant Chunks
→ contexto retorna para o agente/LLM.

FILTRO OBRIGATÓRIO:

tenantId / clinicId.

Não deixar que a LLM determine isso apenas através do prompt.

Adicionar uma anotação:

"Tenant isolation enforced by backend/retriever"

Isso é extremamente importante para evitar vazamento de dados entre clínicas.

--------------------------------------------------
7. VECTOR DATABASE
--------------------------------------------------

Representar:

Vector Database

Exemplos conceituais:

Pinecone / Weaviate / pgVector / similar.

Não é necessário escolher obrigatoriamente uma tecnologia.

Mostrar que armazena:

embedding
+
chunk
+
metadata.

Adicionar preocupação:

Versioning / reindexing.

Caso documentos sejam atualizados:

- gerar nova versão;
- atualizar embeddings;
- evitar conteúdo obsoleto.

--------------------------------------------------
8. SEMANTIC INGESTION PIPELINE
--------------------------------------------------

Criar uma área totalmente separada do runtime.

Fluxo:

Clinic Documents
→ Object Storage
→ Queue
→ Ingestion Worker
→ Parser
→ Chunking
→ Embedding Model
→ Vector Database.

A ingestão deve ser assíncrona.

Adicionar estados conceituais:

PENDING
PROCESSING
INDEXED
FAILED

Adicionar:

Retry + DLQ

para falhas no pipeline.

O pipeline deve ser idempotente.

--------------------------------------------------
9. SCHEDULING DOMAIN
--------------------------------------------------

A tool createAppointment NÃO deve escrever diretamente em banco através da LLM.

Fluxo correto:

Agent
→ Create Appointment Tool
→ Scheduling API / Scheduling Service
→ Transactional Database.

Esse serviço deve continuar sendo software determinístico.

Adicionar preocupações:

IDEMPOTENCY

Evitar que um retry/tool call duplicado crie dois agendamentos.

CONCURRENCY

Dois usuários podem tentar reservar o mesmo horário simultaneamente.

Adicionar proteção conceitual como:

- unique constraint;
- optimistic locking;
- pessimistic locking;
- atomic reservation;

Não precisa escolher apenas uma solução, mas mostre que o problema existe.

Adicionar:

idempotencyKey

no fluxo de criação de agendamento.

--------------------------------------------------
10. CACHE
--------------------------------------------------

Adicionar uma camada de cache quando fizer sentido.

Exemplos:

- informações frequentes da clínica;
- configurações;
- respostas altamente repetitivas e estáveis;
- retrieval cache.

IMPORTANTE:

Cache precisa ser multi-tenant.

Exemplo de chave conceitual:

tenantId + key

Nunca permitir cache global que possa devolver resposta de outra clínica.

Adicionar conceitos:

TTL
Cache-aside
Invalidation

Não exagerar em semantic cache inicialmente; pode aparecer como evolução futura.

--------------------------------------------------
11. MODEL LAYER / MODEL ROUTING
--------------------------------------------------

Representar diferentes providers:

Anthropic / Claude
OpenAI
Google Gemini

Não definir um único modelo como solução absoluta.

Adicionar componente conceitual:

Model Router

Responsabilidade:

escolher modelo com base em:

- custo;
- qualidade;
- latência;
- disponibilidade;
- capacidade de reasoning;
- tool calling;
- tamanho de contexto.

Exemplo:

simple task
→ cheap/fast model

complex reasoning
→ stronger model

IMPORTANTE:

AI Gateway deve ser representado como uma possível evolução arquitetural, e não obrigatoriamente como componente inicial.

Pode existir:

Agent Service
→ Model Router
→ Providers

E uma anotação:

"At scale: extract to AI Gateway"

O AI Gateway futuro pode centralizar:

- provider abstraction;
- model routing;
- rate limiting;
- token quotas;
- observability;
- cost;
- retries;
- fallback;
- governance.

--------------------------------------------------
12. RESILIENCE
--------------------------------------------------

Mostrar preocupações de resiliência em chamadas externas:

Timeout
Retry
Exponential Backoff + Jitter
Circuit Breaker
Fallback

IMPORTANTE:

Retry apenas para falhas transitórias.

Não fazer retry de erro de negócio/validação.

Mostrar fallback entre modelos/providers como possibilidade, mas indicar:

"Fallback must be validated because providers/models may behave differently."

--------------------------------------------------
13. TOKEN / COST CONTROL
--------------------------------------------------

Adicionar uma área ou notas de:

AI Cost Control

Monitorar:

input tokens
output tokens
cost per request
cost per agent
cost per tenant
cost per successful task.

Adicionar técnicas:

- model routing;
- context truncation;
- summarization;
- Top-K control;
- caching;
- max steps;
- max token budget;
- avoid agent loops.

Multi-agent workflows devem possuir:

max steps
timeout
token budget
recursion/loop protection.

--------------------------------------------------
14. EVALUATIONS
--------------------------------------------------

Criar uma área separada de:

Offline Evals / Quality Pipeline

Fluxo conceitual:

Evaluation Dataset
→ Agent Version
→ Evaluation
→ Metrics.

Avaliar:

- factuality;
- correct tool usage;
- retrieval quality;
- hallucination;
- policy compliance;
- safety.

Adicionar:

LLM-as-a-Judge

como possível técnica de avaliação.

IMPORTANTE:

Não colocar obrigatoriamente LLM-as-a-Judge em todas as requisições de produção.

Anotar:

"Prefer offline evals; online judge only when justified or sampled"

Para ações altamente críticas:

Human-in-the-Loop.

--------------------------------------------------
15. OBSERVABILITY
--------------------------------------------------

Adicionar uma camada transversal de:

AI Observability

Possíveis ferramentas:

LangSmith
OpenTelemetry
Prometheus/Grafana
ou equivalentes.

Monitorar:

traceId
conversationId
tenantId
agentId
model
tool calls
retrievals
latency
errors
token usage
cost
agent steps.

Mostrar distributed tracing ao longo do fluxo.

IMPORTANTE:

Não logar indiscriminadamente:

prompts completos
dados médicos
PII
dados sensíveis.

--------------------------------------------------
16. SECURITY
--------------------------------------------------

Criar uma camada transversal de segurança.

Incluir:

Authentication
Authorization
Tenant Isolation
Least Privilege
Encryption in Transit
Encryption at Rest
Secrets Management
LGPD
Data Minimization
PII masking/redaction
Prompt Injection Protection
Guardrails
Audit Logs.

Adicionar uma anotação muito importante:

"Retrieved RAG content must be treated as data, not trusted instructions."

Não permitir que documentos recuperados sobrescrevam system prompts ou autorizem tools.

--------------------------------------------------
17. HUMAN-IN-THE-LOOP
--------------------------------------------------

Adicionar apenas para ações de risco elevado.

Exemplo:

alterações clínicas sensíveis
ações irreversíveis
decisões de alto impacto.

Fluxo:

Agent
→ proposed action
→ Human Approval
→ Tool execution.

Não usar em toda operação.

--------------------------------------------------
18. COMMUNICATION PATTERNS
--------------------------------------------------

Mostrar claramente quais caminhos são:

SYNCHRONOUS

Exemplo:

User
→ API
→ Agent
→ LLM
→ response.

ASYNCHRONOUS

Exemplo:

Document Upload
→ Queue
→ Ingestion Worker.

Para respostas longas de LLM:

considerar Streaming / SSE.

WebSocket apenas se comunicação bidirecional persistente realmente for necessária.

--------------------------------------------------
19. SCALABILITY
--------------------------------------------------

Mostrar que os componentes principais podem escalar horizontalmente:

API/BFF
Agent Service
RAG Service
Workers.

Adicionar:

Autoscaling

com base em métricas relevantes.

Para workers:

queue depth

pode ser uma métrica importante.

Para APIs:

CPU / memory / requests / latency.

Não assumir que escalar apenas Agent Service resolve tudo.

Identificar dependências que também podem virar gargalo:

LLM provider
database
vector database
external APIs.

--------------------------------------------------
20. FAILURE SCENARIOS
--------------------------------------------------

Adicionar uma área lateral chamada:

Failure Scenarios / Questions to Ask

Inclua perguntas como:

- E se o LLM provider estiver indisponível?
- E se o Vector DB ficar indisponível?
- E se o RAG recuperar informação errada?
- E se os documentos estiverem desatualizados?
- E se o agente entrar em loop?
- E se uma tool for chamada duas vezes?
- E se dois pacientes tentarem agendar o mesmo horário?
- E se o tráfego aumentar 10x?
- E se uma clínica tentar acessar dados de outra?
- E se um documento contiver prompt injection?
- E se o custo de tokens crescer drasticamente?
- E se uma chamada de LLM ultrapassar timeout?
- E se o pipeline de ingestão falhar pela metade?

Não é necessário ligar todas essas perguntas com setas.

Pode ser uma área de estudo/anotação.

==================================================
ORGANIZAÇÃO VISUAL
==================================================

Quero que o desenho final seja visualmente legível.

Organize aproximadamente da esquerda para a direita:

CHANNELS
↓
EDGE / API
↓
AGENT / ORCHESTRATION
↓
TOOLS / DOMAIN SERVICES
↓
DATA / RAG / EXTERNAL SYSTEMS.

Pipeline de ingestão deve aparecer separado, preferencialmente abaixo.

Providers de LLM podem aparecer à direita.

Observability, Security e Cost Control devem aparecer como preocupações transversais, talvez como faixas laterais ou inferiores.

Evals devem aparecer separados do fluxo principal de runtime.

Utilize cores/grupos diferentes para:

- canais;
- backend tradicional;
- camada de IA;
- dados;
- infraestrutura;
- concerns transversais.

Evite cruzamento excessivo de linhas.

Use setas com labels quando relevante:

Sync
Async
Tool Call
Retrieval
LLM Call
Event
Streaming.

==================================================
PRINCÍPIOS ARQUITETURAIS
==================================================

O desenho deve demonstrar explicitamente:

1. Começar simples.
2. Não usar multi-agent sem necessidade.
3. IA decide; backend determinístico protege regras críticas.
4. LLM não é authorization layer.
5. Multi-tenancy deve ser garantido pela aplicação.
6. RAG não garante factualidade automaticamente.
7. Vector DB não precisa ser source of truth.
8. Evals devem existir antes de produção.
9. Observabilidade de IA inclui tokens, custo e agent traces.
10. Segurança e LGPD são concerns de primeira classe.
11. Escalabilidade não significa apenas adicionar pods.
12. Toda decisão deve considerar:
   qualidade
   custo
   latência
   segurança.
13. Toda escolha arquitetural possui trade-offs.

==================================================
OBJETIVO FINAL
==================================================

O resultado deve ser um diagrama de System Design com IA que eu possa usar para estudar os seguintes assuntos:

- System Design tradicional;
- sistemas distribuídos;
- arquiteturas com LLMs;
- agentes;
- multi-agentes;
- LangChain;
- LangGraph;
- RAG;
- embeddings;
- bancos vetoriais;
- tools;
- model routing;
- AI Gateway;
- caching;
- mensageria;
- segurança;
- observabilidade;
- evals;
- custos;
- escalabilidade;
- resiliência.

Não transforme o desenho em uma arquitetura impossível de ler.

Se algum conceito não precisar obrigatoriamente fazer parte do runtime inicial, represente como:

"Evolution / Optional / At Scale"

em vez de colocá-lo como dependência obrigatória.

Primeiro analise o XML existente.
Depois proponha mentalmente a reorganização.
Em seguida faça as alterações no arquivo XML.

Após editar:

1. valide que o XML continua sendo um arquivo Draw.io válido;
2. garanta que todas as conexões têm source e target coerentes;
3. garanta que nenhum componente importante ficou sobreposto;
4. preserve boa legibilidade;
5. mantenha o arquivo totalmente editável no Draw.io.

Ao final, além de editar o arquivo, me entregue um resumo textual contendo:

- o que existia originalmente;
- o que foi corrigido;
- o que foi adicionado;
- quais componentes são obrigatórios no MVP;
- quais componentes são evoluções futuras;
- principais trade-offs;
- principais failure scenarios que devo estudar.