# Requirements Document

## Introduction

Esta feature implementa o FAQ do SupportBot: um serviço REST que recebe a pergunta de um cliente, identifica a categoria (Prazo de Entrega, Troca e Devolução ou Pagamentos), retorna a resposta mais adequada com um score de confiança e, quando a confiança for baixa, abre automaticamente um ticket com os dados do cliente para follow-up humano.

## Glossary

- **SupportBot**: Sistema de suporte ao cliente que responde FAQs e abre tickets automaticamente.
- **FAQ_Service**: Componente de aplicação responsável por processar perguntas e retornar respostas.
- **Ticket_Service**: Componente de aplicação responsável por criar e persistir tickets de suporte.
- **FAQ_Repository**: Port de saída para persistência e consulta de entradas de FAQ.
- **Ticket_Repository**: Port de saída para persistência e consulta de tickets.
- **Confidence_Score**: Valor numérico entre 0.0 e 1.0 que representa o grau de certeza do SupportBot ao associar uma pergunta a uma resposta de FAQ.
- **Confidence_Threshold**: Valor mínimo de Confidence_Score (0.5) abaixo do qual o SupportBot considera a resposta inadequada e abre um ticket.
- **Ticket**: Registro estruturado contendo os dados do cliente e a pergunta original, criado quando o Confidence_Score está abaixo do Confidence_Threshold.
- **FAQ_Entry**: Entidade de domínio que representa um par pergunta/resposta pertencente a uma categoria de FAQ.
- **Category**: Classificação temática de uma FAQ_Entry. Valores válidos: `DELIVERY`, `RETURNS`, `PAYMENTS`.
- **Client**: Usuário final que envia perguntas ao SupportBot via API REST.

---

## Requirements

### Requirement 1: Responder Perguntas de FAQ

**User Story:** Como cliente, quero enviar uma pergunta ao SupportBot via API REST, para que eu receba uma resposta imediata sobre prazo de entrega, troca/devolução ou pagamentos.

#### Acceptance Criteria

1. WHEN uma requisição POST é recebida em `/api/faq/ask` com um corpo JSON contendo os campos `clientName`, `clientEmail` e `question`, THE FAQ_Service SHALL retornar uma resposta HTTP 200 contendo os campos `answer`, `category` e `confidenceScore`.
2. WHEN a pergunta recebida corresponde a uma FAQ_Entry com Confidence_Score maior ou igual a 0.5, THE FAQ_Service SHALL retornar a resposta da FAQ_Entry correspondente.
3. THE FAQ_Service SHALL calcular o Confidence_Score com base na similaridade textual entre a pergunta recebida e as perguntas cadastradas nas FAQ_Entries.
4. THE FAQ_Service SHALL associar cada resposta retornada à Category da FAQ_Entry correspondente.
5. IF o corpo da requisição não contiver os campos obrigatórios `clientName`, `clientEmail` ou `question`, THEN THE SupportBot SHALL retornar HTTP 400 com uma mensagem de erro descritiva.
6. IF o campo `question` for uma string vazia ou contiver apenas espaços em branco, THEN THE FAQ_Service SHALL retornar HTTP 400 com uma mensagem de erro descritiva.

---

### Requirement 2: Abertura Automática de Ticket por Baixa Confiança

**User Story:** Como agente de suporte, quero receber um ticket quando o SupportBot não souber responder com confiança, para que eu possa fazer o follow-up com o cliente.

#### Acceptance Criteria

1. WHEN o Confidence_Score calculado para uma pergunta for menor que 0.5, THE Ticket_Service SHALL criar um Ticket contendo `clientName`, `clientEmail`, `question` e o timestamp de criação.
2. WHEN um Ticket é criado, THE Ticket_Service SHALL persistir o Ticket via Ticket_Repository e retornar HTTP 200 com os campos `ticketId`, `message` e `confidenceScore` na resposta ao cliente.
3. THE SupportBot SHALL retornar ao cliente uma mensagem informando que a pergunta foi encaminhada para um agente humano, sempre que um Ticket for aberto.
4. IF a persistência do Ticket falhar, THEN THE Ticket_Service SHALL retornar HTTP 500 com uma mensagem de erro descritiva.

---

### Requirement 3: Gerenciamento de Entradas de FAQ

**User Story:** Como administrador do sistema, quero cadastrar, consultar e remover entradas de FAQ, para que o SupportBot tenha uma base de conhecimento atualizada.

#### Acceptance Criteria

1. WHEN uma requisição POST é recebida em `/api/faq` com um corpo JSON contendo `question`, `answer` e `category`, THE FAQ_Service SHALL persistir a FAQ_Entry via FAQ_Repository e retornar HTTP 201 com os dados da entrada criada, incluindo o `id` gerado.
2. WHEN uma requisição GET é recebida em `/api/faq`, THE FAQ_Service SHALL retornar HTTP 200 com a lista de todas as FAQ_Entries persistidas.
3. WHEN uma requisição GET é recebida em `/api/faq?category={category}`, THE FAQ_Service SHALL retornar HTTP 200 com a lista de FAQ_Entries filtradas pela Category informada.
4. WHEN uma requisição DELETE é recebida em `/api/faq/{id}`, THE FAQ_Service SHALL remover a FAQ_Entry correspondente via FAQ_Repository e retornar HTTP 204.
5. IF uma requisição DELETE for recebida em `/api/faq/{id}` e não existir FAQ_Entry com o `id` informado, THEN THE FAQ_Service SHALL retornar HTTP 404 com uma mensagem de erro descritiva.
6. IF o campo `category` de uma requisição POST em `/api/faq` não corresponder a um dos valores válidos (`DELIVERY`, `RETURNS`, `PAYMENTS`), THEN THE FAQ_Service SHALL retornar HTTP 400 com uma mensagem de erro descritiva.

---

### Requirement 4: Consulta de Tickets Abertos

**User Story:** Como agente de suporte, quero consultar os tickets abertos pelo SupportBot, para que eu possa priorizar e realizar o atendimento humano.

#### Acceptance Criteria

1. WHEN uma requisição GET é recebida em `/api/tickets`, THE Ticket_Service SHALL retornar HTTP 200 com a lista de todos os Tickets persistidos, ordenados pelo timestamp de criação em ordem decrescente.
2. WHEN uma requisição GET é recebida em `/api/tickets/{id}`, THE Ticket_Service SHALL retornar HTTP 200 com os dados do Ticket correspondente.
3. IF uma requisição GET for recebida em `/api/tickets/{id}` e não existir Ticket com o `id` informado, THEN THE Ticket_Service SHALL retornar HTTP 404 com uma mensagem de erro descritiva.

---

### Requirement 5: Persistência e Isolamento de Camadas

**User Story:** Como desenvolvedor, quero que a lógica de negócio seja isolada da infraestrutura, para que o sistema seja testável e evolutivo conforme a Arquitetura Hexagonal.

#### Acceptance Criteria

1. THE SupportBot SHALL definir FAQ_Repository e Ticket_Repository como interfaces (ports) na camada de domínio, sem dependências de frameworks de persistência.
2. THE SupportBot SHALL implementar FAQ_Repository e Ticket_Repository como adaptadores na camada de infraestrutura usando Spring Data JPA.
3. THE SupportBot SHALL manter as entidades JPA restritas à camada de infraestrutura, utilizando DTOs para comunicação entre camadas de aplicação e infraestrutura.
4. THE FAQ_Service e THE Ticket_Service SHALL receber suas dependências exclusivamente por injeção de construtor.
5. WHILE o perfil de execução for desenvolvimento, THE SupportBot SHALL utilizar o banco de dados H2 em memória para persistência de FAQ_Entries e Tickets.
