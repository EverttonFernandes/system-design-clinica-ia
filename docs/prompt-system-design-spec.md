Quero evoluir o System Design atual desta aplicação agêntica para a VERSÃO 2.

IMPORTANTE:
- Não crie uma arquitetura completamente nova ignorando o desenho existente.
- Analise primeiro o System Design V1 disponível no projeto.
- Preserve tudo que ainda fizer sentido.
- Refatore, complemente e amadureça a arquitetura.
- O objetivo da V2 é representar uma aplicação agêntica preparada para produção, segura, escalável, observável, resiliente e economicamente controlada.
- Justifique decisões arquiteturais e explicite trade-offs.
- Evite overengineering: componentes devem existir porque resolvem um problema concreto.
- Quando houver mais de uma solução possível, apresente a escolha recomendada e a alternativa.

==================================================
CONTEXTO DO SISTEMA
==================================================

Estamos modelando uma plataforma de clínicas médicas.

O usuário/paciente pode interagir principalmente através de:

1. Chat Web;
2. WhatsApp.

A aplicação possui um assistente agêntico capaz de:

- responder dúvidas sobre a clínica;
- consultar documentos através de RAG;
- consultar horários;
- identificar especialidades;
- consultar informações autorizadas do paciente;
- realizar agendamentos;
- cancelar ou alterar agendamentos;
- eventualmente executar outras ações através de tools.

A plataforma é multi-tenant.

Existem diversas clínicas usando a mesma plataforma, portanto:

- nenhum dado de uma clínica pode vazar para outra;
- toda operação deve respeitar tenantId/clinicId;
- autorização deve ocorrer antes do acesso aos dados;
- princípio do menor privilégio deve ser aplicado em todas as camadas.

==================================================
1. CANAIS DE ENTRADA
==================================================

Representar:

Web Chat
e
WhatsApp.

Para WhatsApp:

WhatsApp Provider
→ Webhook
→ API Gateway
→ serviço responsável por mensagens/conversas.

O webhook deve:

- validar autenticidade da origem;
- validar assinatura quando aplicável;
- identificar tenant/clínica;
- identificar usuário/conversa;
- possuir proteção contra eventos duplicados;
- responder rapidamente ao provider;
- evitar processamento pesado dentro da própria chamada HTTP.

Caso necessário:

Webhook
→ publicação assíncrona
→ fila/event bus
→ processamento posterior.

Não aprofundar detalhes específicos da API da Meta.
O objetivo é demonstrar arquitetura.

==================================================
2. API GATEWAY
==================================================

Adicionar API Gateway como ponto de entrada para APIs determinísticas.

Responsabilidades:

- autenticação;
- autorização inicial;
- TLS;
- rate limiting;
- quotas;
- logs;
- métricas;
- tracing/correlation ID;
- versionamento de APIs;
- roteamento;
- proteção contra abuso;
- integração com mecanismos de segurança.

As tools do agente NÃO devem acessar bancos diretamente.

Fluxo desejado:

Agent
→ Tool
→ API Gateway
→ API determinística
→ regra de negócio
→ banco.

Exemplos:

searchClinicKnowledge()
→ Knowledge API
→ Vector Store.

getAvailableSlots()
→ Scheduling API
→ PostgreSQL.

createAppointment()
→ Scheduling API
→ PostgreSQL.

getPatientData()
→ Patient API
→ PostgreSQL.

IMPORTANTE:

API Gateway não deve conter regra de negócio complexa.

Ele roteia, protege e aplica políticas transversais.

As regras de negócio continuam nos respectivos serviços.

==================================================
3. APIS DETERMINÍSTICAS
==================================================

Toda operação sensível deve passar por código determinístico.

A LLM pode solicitar uma ação.

A aplicação determinística decide se ela pode acontecer.

Exemplo:

LLM:
"quero criar agendamento"

↓

Tool:
createAppointment(...)

↓

Scheduling API:

- autentica;
- autoriza;
- valida tenant;
- valida paciente;
- valida médico;
- valida horário;
- verifica disponibilidade atual;
- verifica idempotência;
- aplica regra de negócio;
- realiza transação;
- persiste;
- audita.

Somente então retorna sucesso.

Nunca confiar apenas na decisão da LLM.

==================================================
4. REDIS
==================================================

Adicionar Redis explicitamente como componente estratégico.

Não utilizar apenas como cache genérico.

Separar responsabilidades através de diferentes namespaces/chaves.

Exemplo:

session:{sessionId}

Responsável por:

- contexto recente;
- últimas mensagens;
- intenção atual;
- especialidade atual;
- período desejado;
- informações temporárias necessárias para a conversa.

TTL relativamente curto.

Exemplo conceitual:

30 minutos
2 horas
ou valor configurável.

----------------------------------

answer-cache:{patientId}:{questionHash}

Responsável por:

- respostas anteriormente fornecidas ao mesmo paciente;
- evitar executar novamente LLM/RAG quando possível.

TTL pode ser diferente da sessão.

----------------------------------

shared-answer-cache:{clinicId}:{questionHash}

Responsável por respostas reutilizáveis entre pacientes da mesma clínica.

Somente utilizar quando a resposta NÃO possuir dados pessoais.

Exemplos:

- política de cancelamento;
- como entregar um exame;
- endereço da clínica;
- funcionamento da clínica;
- regras administrativas.

Nunca compartilhar cache contendo:

- resultados de exames;
- prontuário;
- dados médicos;
- informações pessoais;
- informações sensíveis individualizadas.

----------------------------------

usage:{patientId}

ou estruturas equivalentes.

Manter informações como:

- requests/minuto;
- requests/dia;
- chamadas LLM;
- chamadas RAG;
- chamadas de tools;
- tokens de entrada;
- tokens de saída;
- custo aproximado;
- tentativas de operações sensíveis.

Utilizar essas informações para:

- rate limit;
- quota;
- FinOps;
- detecção de abuso;
- proteção contra Denial of Wallet.

----------------------------------

Cada tipo de cache deve possuir TTL próprio.

Exemplo:

sessão
→ curto.

política da clínica
→ longo.

agenda
→ curto.

informação altamente dinâmica
→ muito curto ou sem cache.

rate limit
→ segundos/minutos.

quota
→ horas/dias.

==================================================
5. DENIAL OF WALLET / FINOPS
==================================================

A aplicação estará exposta a usuários potencialmente maliciosos.

Projetar proteção contra:

- scripts Python em loop;
- milhares de perguntas repetidas;
- prompts gigantes;
- chamadas repetidas de RAG;
- chamadas excessivas de MCP;
- loops entre tools;
- loops entre agentes;
- geração excessiva de tokens;
- pedidos absurdamente grandes;
- tentativa de usar modelos caros;
- tentativas de gerar centenas ou milhares de operações.

Adicionar:

Rate Limit

Exemplo:

máximo X requests/minuto.

Quota

Exemplo:

máximo X requests/dia.

Concurrency Limit

Exemplo:

máximo X requisições simultâneas por usuário/tenant.

Agent Execution Budget:

- maxSteps;
- maxToolCalls;
- maxLLMCalls;
- maxInputTokens;
- maxOutputTokens;
- maxCost;
- timeout;
- maxRetries.

IMPORTANTE:

Esses limites devem ser determinísticos.

A LLM não decide quanto pode gastar.

O runtime decide.

Adicionar também:

- alertas de custo;
- métricas por usuário;
- métricas por tenant;
- custo por feature;
- custo por agente;
- possibilidade de throttling;
- possibilidade de bloqueio;
- possibilidade de degradar para modelo mais barato.

==================================================
6. AI GATEWAY
==================================================

Adicionar AI Gateway entre Agent Runtime e providers de LLM.

O agente não deve ficar diretamente acoplado a:

OpenAI,
Anthropic,
Google,
ou qualquer modelo específico.

Fluxo:

Agent
→ AI Gateway
→ Model Router
→ Provider / LLM.

O Agent pode solicitar uma rota lógica, por exemplo:

general-chat

simple-extraction

summarization

complex-reasoning

document-analysis.

O AI Gateway decide qual modelo utilizar.

Exemplo:

simple-extraction
→ modelo pequeno/barato.

general-chat
→ modelo intermediário.

complex-reasoning
→ modelo mais poderoso.

----------------------------------

Responsabilidades do AI Gateway:

- model routing;
- fallback;
- retry controlado;
- timeout;
- circuit breaker;
- observabilidade;
- token accounting;
- cost accounting;
- rate limiting de modelos;
- autenticação com providers;
- abstração das credenciais;
- model versioning;
- controle de budget;
- possibilidade de trocar provider sem alterar o agente.

----------------------------------

Fallback:

Primary Provider
→ falhou.

↓

Secondary Provider.

IMPORTANTE:

Não assumir que modelos diferentes são completamente intercambiáveis.

Modelos de fallback precisam passar pelos mesmos Evals relevantes.

----------------------------------

Retry:

Utilizar:

- retry limitado;
- exponential backoff;
- jitter;
- timeout.

Evitar retry amplification.

Por exemplo:

Agent faz retry
+
AI Gateway faz retry
+
Provider faz retry

não pode causar multiplicação descontrolada de chamadas.

Centralizar política de retry.

==================================================
7. TOOLS
==================================================

Representar tools explicitamente.

Exemplos:

searchClinicKnowledge()

getAvailableSlots()

getPatientData()

createAppointment()

cancelAppointment()

updateAppointment()

Tools devem representar capacidades de negócio.

Evitar tools genéricas como:

executeSQL(sql)

callAnyAPI(url)

executeShell(command)

genericHttpRequest(...)

----------------------------------

PRINCÍPIO:

Agent conhece capacidade.

Não conhece implementação.

Exemplo:

Agent:

createAppointment()

não sabe:

- tabela;
- SQL;
- banco;
- endpoint interno;
- provider;
- detalhes de persistência.

==================================================
8. IDEMPOTÊNCIA
==================================================

Adicionar idempotência para operações com side effect.

Exemplos:

createAppointment()

processPayment()

cancelAppointment()

sendDocument()

createPrescription()

Não é obrigatório utilizar idempotency key em operações somente de leitura.

Exemplo:

getAvailableSlots()

normalmente não precisa.

----------------------------------

Exemplo:

Idempotency-Key:
appointment:{patientId}:{doctorId}:{slotId}

Se a mesma operação for repetida por:

- usuário;
- agente;
- retry;
- timeout;
- webhook duplicado;

o backend deve reconhecer a operação e evitar duplicidade.

==================================================
9. RAG / VECTOR STORE
==================================================

Representar corretamente:

RAG não é o banco.

RAG é o processo de recuperação.

O armazenamento vetorial pode ser:

PostgreSQL + pgvector.

Fluxo:

documento
→ validação
→ segurança
→ chunking
→ embeddings
→ pgVector.

Consulta:

pergunta
→ embedding
→ metadata filtering
→ vector similarity search
→ Top K
→ contexto
→ LLM.

----------------------------------

Adicionar filtros obrigatórios.

Exemplo:

tenantId = clinicABC.

clinicId = clinicABC.

documentVisibility = PATIENT.

specialty = dermatology.

Quando aplicável:

patientId.

----------------------------------

IMPORTANTE:

Autorização acontece antes ou durante a recuperação.

Nunca permitir:

searchEntireVectorDatabase(query)

sem filtros de tenant.

----------------------------------

Adicionar índices adequados.

Avaliar:

HNSW

ou

IVFFlat.

Justificar escolha.

Também avaliar índices tradicionais para campos usados em filtros:

tenantId;

clinicId;

documentType;

visibility;

patientId;

etc.

Objetivo:

reduzir espaço de busca antes da similaridade.

----------------------------------

TOP K:

Definir limite máximo determinístico.

Exemplo:

Top K = 5.

A LLM não pode solicitar:

Top K = 5000.

A aplicação define limites.

==================================================
10. SEGURANÇA DE DOCUMENTOS / PROMPT INJECTION
==================================================

Todo documento enviado pelo usuário é UNTRUSTED.

Pipeline:

Upload
→ validação de formato
→ tamanho
→ malware scan
→ extração segura
→ análise
→ sanitização
→ classificação
→ chunking
→ embedding.

Documentos podem conter indirect prompt injection.

Conteúdo recuperado deve ser tratado como:

DATA

e nunca:

INSTRUCTION.

----------------------------------

Mesmo que prompt injection passe:

tools devem possuir mínimo privilégio.

A autorização continua no backend.

A arquitetura deve assumir:

"a LLM pode ser enganada".

Mesmo assim:

"ela não deve conseguir causar dano".

==================================================
11. DADOS DO PACIENTE
==================================================

Dados oficiais e transacionais devem permanecer em banco relacional.

Preferência:

PostgreSQL.

Exemplos:

paciente;

consultas;

agendamentos;

alergias;

prontuário;

pagamentos;

informações administrativas.

----------------------------------

IMPORTANTE:

Não confundir HASH com criptografia.

HASH:

usar quando NÃO é necessário recuperar o valor original.

Exemplos:

password;

comparações específicas;

fingerprints;

identificadores derivados.

Hash NÃO pode ser descriptografado.

----------------------------------

CRIPTOGRAFIA:

usar para dados que precisam ser recuperados posteriormente.

Exemplos:

dados clínicos altamente sensíveis.

Utilizar:

encryption at rest;

encryption in transit;

KMS/Key Management;

key rotation;

controle de acesso;

segregação de responsabilidades.

Não armazenar chaves criptográficas no código.

==================================================
12. AUDITORIA
==================================================

Criar trilha de auditoria persistente para ações relevantes.

Exemplo:

quem fez;

quando;

tenant;

paciente;

ação;

resourceId;

resultado;

tool utilizada;

correlationId;

IP/contexto quando apropriado;

antes/depois quando necessário.

Não necessariamente armazenar a conversa inteira.

Guardar eventos relevantes.

Exemplos:

appointment.created

appointment.cancelled

patient.updated

medicalDocument.accessed

tool.executed

authorization.denied.

==================================================
13. COMPLIANCE / LGPD
==================================================

Aplicar:

data minimization;

least privilege;

purpose limitation;

retention policy;

access control;

auditability;

encryption;

consentimento quando aplicável;

right to deletion quando legalmente permitido;

segregação de dados por tenant.

Não utilizar "compliance" como justificativa para armazenar tudo.

Guardar apenas o necessário.

Definir:

o que guardar;

por quê;

por quanto tempo;

quem pode acessar;

quando deletar.

==================================================
14. CONCORRÊNCIA DE AGENDAMENTO
==================================================

Este ponto é CRÍTICO.

Cenário:

Paciente A consulta:

14:00 está disponível.

Paciente B consulta quase simultaneamente:

14:00 está disponível.

Paciente A agenda.

Paciente B tenta agendar logo depois.

Não confiar no cache.

Não confiar na informação mostrada anteriormente.

No momento da criação:

Scheduling API deve consultar novamente o estado oficial.

----------------------------------

Utilizar estratégia de concorrência.

Avaliar:

optimistic locking;

pessimistic locking;

SELECT FOR UPDATE;

unique constraint;

atomic operation.

Uma solução simples pode ser:

UNIQUE(
    doctor_id,
    appointment_date,
    appointment_time
)

Assim:

Paciente A:

INSERT
→ sucesso.

Paciente B:

INSERT
→ constraint violation.

Retornar:

SLOT_ALREADY_TAKEN.

----------------------------------

A confirmação de agendamento exige consistência forte.

Não utilizar consistência eventual para decidir:

"quem ficou com o horário".

Consistência eventual pode existir em:

- atualização de cache;
- analytics;
- notificações;
- índices secundários;
- processamento assíncrono.

Mas não na confirmação final do slot.

==================================================
15. CACHE DE AGENDA
==================================================

Agenda pode ser cacheada para leitura.

Porém:

TTL curto.

Quando houver:

createAppointment()

cancelAppointment()

rescheduleAppointment()

invalidar cache relevante.

Mesmo assim:

antes de persistir:

consultar/validar banco transacional.

Cache melhora leitura.

Banco decide a verdade.

==================================================
16. LOAD BALANCER
==================================================

O System Design V2 deve nascer preparado para escala horizontal.

Representar:

Internet
→ Load Balancer
→ múltiplas instâncias.

Exemplos:

Webhook Service
x N.

Conversation Service
x N.

Agent Runtime
x N.

Scheduling Service
x N.

Knowledge Service
x N.

Evitar estado local nas instâncias.

Estado compartilhado deve estar em:

Redis;

PostgreSQL;

Object Storage;

Vector Store;

outros componentes adequados.

----------------------------------

Se o volume de agendamentos crescer:

Load Balancer
→ Scheduling Service 1
→ Scheduling Service 2
→ Scheduling Service 3.

O banco continua garantindo concorrência e consistência.

==================================================
17. FILAS / MENSAGERIA
==================================================

Avaliar uso de:

Kafka;

RabbitMQ;

SQS;

ou equivalente

quando processamento síncrono não for necessário.

Exemplos:

audit events;

notifications;

WhatsApp outbound;

email;

analytics;

document ingestion;

embedding generation;

background processing.

----------------------------------

Evitar colocar no fluxo síncrono operações que podem ser processadas posteriormente.

==================================================
18. RESILIÊNCIA
==================================================

Adicionar:

timeout;

retry;

backoff;

jitter;

circuit breaker;

fallback;

bulkhead quando aplicável;

DLQ para processamento assíncrono.

----------------------------------

IMPORTANTE:

Retries em operações de escrita devem existir somente quando a operação for idempotente ou possuir mecanismo equivalente.

Nunca executar retry cego de:

createAppointment()

processPayment()

sem idempotência.

==================================================
19. OBSERVABILIDADE
==================================================

Adicionar observabilidade tradicional:

logs;

metrics;

traces;

correlationId;

latency;

error rate;

throughput;

database metrics;

cache hit ratio.

----------------------------------

Adicionar observabilidade de IA:

model;

provider;

prompt version;

tokens input;

tokens output;

cost;

latency;

tool calls;

RAG chunks;

retrieval score;

fallback;

retry;

agent steps;

agent loops;

evaluation results.

Permitir reconstruir:

User Request
→ Agent
→ LLM
→ Tool
→ API
→ Database
→ Response.

==================================================
20. EVALS
==================================================

Adicionar pipeline de Evals.

Avaliar:

- tool selection;
- tool parameters;
- hallucination;
- groundedness;
- retrieval relevance;
- final answer;
- agent trajectory;
- security;
- prompt injection resistance;
- cost;
- latency;
- fallback models.

Usar combinação de:

deterministic evals;

LLM-as-a-Judge;

human evaluation quando necessário.

Executar Evals:

antes de colocar modelo novo;

antes de trocar prompt;

antes de trocar provider;

antes de habilitar fallback.

==================================================
21. TESTES
==================================================

Separar:

Unit Tests

Integration Tests

Functional Tests

Security Tests

Load Tests

Concurrency Tests

Evals.

Exemplo de teste crítico:

100 pacientes tentam agendar o mesmo horário.

Resultado esperado:

1 agendamento criado.

99 recebem:

SLOT_ALREADY_TAKEN.

Nenhum agendamento duplicado.

==================================================
22. MULTI-TENANCY
==================================================

Representar explicitamente.

Toda requisição deve possuir contexto confiável:

tenantId.

Nunca confiar em tenantId inventado pela LLM.

Tenant deve vir de:

JWT;

sessão autenticada;

credencial;

mapeamento confiável do canal;

outro mecanismo seguro.

----------------------------------

Todas as queries devem respeitar tenant.

PostgreSQL:

WHERE tenant_id = ?

Vector Store:

filter tenantId = ?

Redis:

namespace por tenant.

Auditoria:

tenantId obrigatório.

FinOps:

custos por tenant.

==================================================
23. FLUXOS QUE O SYSTEM DESIGN V2 DEVE MOSTRAR
==================================================

Representar pelo menos estes fluxos:

FLUXO A:

Paciente pergunta política da clínica.

Web/WhatsApp
→ API Gateway
→ Redis Cache
→ cache miss
→ Agent
→ Tool
→ Knowledge API
→ Authorization
→ pgVector
→ Top K
→ Agent
→ AI Gateway
→ LLM
→ resposta
→ Redis Cache
→ paciente.

----------------------------------

FLUXO B:

Paciente pergunta novamente a mesma informação.

Web/WhatsApp
→ API Gateway
→ Redis
→ CACHE HIT
→ resposta.

Evitar:

RAG;

LLM;

pgVector.

----------------------------------

FLUXO C:

Paciente consulta agenda.

Paciente
→ Agent
→ getAvailableSlots()
→ API Gateway
→ Scheduling API
→ PostgreSQL
→ horários.

----------------------------------

FLUXO D:

Paciente confirma horário.

Agent
→ createAppointment()
→ Idempotency Key
→ API Gateway
→ Scheduling API
→ validações
→ concorrência
→ transação
→ PostgreSQL
→ audit event
→ resposta.

----------------------------------

FLUXO E:

LLM principal indisponível.

Agent
→ AI Gateway
→ Primary Provider
→ timeout
→ circuit breaker
→ fallback
→ Secondary Provider.

----------------------------------

FLUXO F:

Usuário malicioso executando script em loop.

Request
→ API Gateway
→ Rate Limit / Quota
→ BLOCK

antes de:

LLM;

RAG;

Tools.

==================================================
24. SAÍDA ESPERADA
==================================================

Produza a V2 em etapas.

Primeiro:

analise o System Design V1.

Depois apresente:

1. problemas/riscos identificados na V1;

2. melhorias necessárias;

3. System Design V2;

4. diagrama completo;

5. responsabilidades de cada componente;

6. fluxo das principais requisições;

7. fluxo do agente;

8. fluxo de RAG;

9. fluxo de agendamento;

10. fluxo de segurança;

11. fluxo de cache;

12. fluxo de fallback;

13. estratégia de escalabilidade;

14. estratégia de observabilidade;

15. estratégia de FinOps;

16. estratégia de concorrência;

17. estratégia de multi-tenancy;

18. principais trade-offs;

19. possíveis gargalos;

20. riscos ainda existentes;

21. diferenças entre V1 e V2.

==================================================
25. DIAGRAMA
==================================================

Gerar um diagrama arquitetural visualmente claro.

Preferencialmente utilizar Mermaid.

Separar visualmente:

CLIENT CHANNELS

EDGE

APPLICATION

AGENT / AI

TOOLS

DETERMINISTIC SERVICES

DATA

ASYNC PROCESSING

OBSERVABILITY / SECURITY.

Não criar um diagrama impossível de ler.

Se necessário:

criar primeiro um diagrama macro.

Depois:

diagramas específicos por fluxo.

==================================================
26. PRINCÍPIO ARQUITETURAL CENTRAL
==================================================

Toda a arquitetura deve obedecer esta ideia:

LLM:

interpreta;

raciocina;

escolhe capacidades;

gera linguagem.

Agent:

orquestra.

Tools:

expõem capacidades limitadas.

AI Gateway:

governa acesso às LLMs.

API Gateway:

governa acesso às APIs.

APIs determinísticas:

validam e executam regras.

Redis:

sessão, cache, rate limit, quota e dados temporários.

PostgreSQL:

source of truth transacional.

pgVector:

recuperação semântica.

Mensageria:

processamento assíncrono.

Observabilidade:

permite explicar o que aconteceu.

Auditoria:

permite provar o que aconteceu.

==================================================
27. FILOSOFIA DE SEGURANÇA
==================================================

Nunca confiar:

no usuário;

no documento;

na LLM;

no resultado externo;

na repetição de chamadas;

em dados vindos do prompt.

Validar em fronteiras determinísticas.

O objetivo NÃO é:

"garantir que a LLM nunca erre."

O objetivo é:

"mesmo que a LLM erre ou seja manipulada, ela não tenha poder suficiente para comprometer dados, causar efeitos indevidos ou gerar custos ilimitados."

==================================================
28. RESTRIÇÃO FINAL
==================================================

Não adicione tecnologias apenas por moda.

Sempre responda:

Qual problema este componente resolve?

Se Redis + PostgreSQL + pgVector resolverem o cenário, não introduzir MongoDB, outro Vector DB ou banco adicional sem necessidade concreta.

O System Design V2 deve demonstrar:

simplicidade;

segurança;

escalabilidade;

resiliência;

manutenibilidade;

observabilidade;

FinOps;

boa experiência do usuário;

e evolução futura sem overengineering.