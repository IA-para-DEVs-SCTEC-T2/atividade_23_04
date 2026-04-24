# Implementation Plan: SupportBot FAQ

## Overview

Implementação incremental do SupportBot FAQ seguindo Arquitetura Hexagonal (Ports & Adapters) em Java 17 com Spring Boot. As tarefas progridem da camada de domínio para infraestrutura, garantindo que cada etapa seja integrada antes de avançar.

## Tasks

- [ ] 1. Configurar estrutura do projeto e camada de domínio
  - Criar projeto Spring Boot com dependências: `spring-boot-starter-web`, `spring-boot-starter-data-jpa`, `h2`, `jqwik`
  - Criar estrutura de pacotes: `domain/`, `application/`, `infrastructure/`
  - Implementar enum `Category` com valores `DELIVERY`, `RETURNS`, `PAYMENTS`
  - Implementar entidade de domínio `FaqEntry` (sem anotações JPA) com campos `id`, `question`, `answer`, `category`
  - Implementar entidade de domínio `Ticket` com campos `id`, `clientName`, `clientEmail`, `question`, `createdAt`
  - Definir port `FaqRepository` com métodos `save`, `findAll`, `findByCategory`, `findById`, `deleteById`, `existsById`
  - Definir port `TicketRepository` com métodos `save`, `findAllOrderByCreatedAtDesc`, `findById`
  - Implementar exceções de domínio `FaqEntryNotFoundException` e `TicketNotFoundException`
  - _Requirements: 5.1_

- [ ] 2. Implementar `SimilarityCalculator` e `MatchResult`
  - [ ] 2.1 Implementar `MatchResult` como record com campos `FaqEntry entry` e `double score`
    - _Requirements: 1.3_

  - [ ] 2.2 Implementar `SimilarityCalculator` com método `findBestMatch(String question, List<FaqEntry> entries)`
    - Implementar `jaccardSimilarity(String a, String b)`: tokenizar por lowercase + split em espaço/pontuação, calcular interseção/união dos conjuntos de tokens
    - `findBestMatch` deve iterar todas as entries e retornar o `MatchResult` com maior score; retornar score 0.0 se a lista estiver vazia
    - _Requirements: 1.3_

  - [ ]* 2.3 Escrever testes de unidade para `SimilarityCalculator`
    - Testar `jaccardSimilarity` com strings idênticas (esperado: 1.0), strings sem tokens em comum (esperado: 0.0) e strings parcialmente similares
    - Testar `findBestMatch` com lista vazia e com múltiplas entries
    - _Requirements: 1.3_

  - [ ]* 2.4 Escrever property test para `SimilarityCalculator` — Property 1
    - **Property 1: Score sempre dentro do intervalo válido**
    - Para qualquer par de strings, `jaccardSimilarity` deve retornar valor em [0.0, 1.0]; para strings idênticas não-vazias, deve retornar 1.0
    - Usar `@Property` com `@ForAll String` no `SimilarityCalculatorTest`
    - `// Feature: supportbot-faq, Property 1: Score sempre dentro do intervalo válido`
    - **Validates: Requirements 1.3**

- [ ] 3. Implementar `TicketService` (camada de aplicação)
  - [ ] 3.1 Implementar DTOs: `TicketResponse(Long ticketId, String message, double confidenceScore)` e `TicketListResponse(Long id, String clientName, String clientEmail, String question, LocalDateTime createdAt)`
    - _Requirements: 2.2_

  - [ ] 3.2 Implementar `TicketService` com injeção de `TicketRepository` por construtor
    - Método `createTicket(String clientName, String clientEmail, String question, double score)`: criar `Ticket` com `createdAt = LocalDateTime.now()`, persistir via `TicketRepository`, retornar `TicketResponse` com mensagem fixa informando encaminhamento ao agente humano
    - Método `listAll()`: delegar a `TicketRepository.findAllOrderByCreatedAtDesc()`, mapear para `TicketListResponse`
    - Método `findById(Long id)`: buscar via `TicketRepository`, lançar `TicketNotFoundException` se ausente, retornar `TicketListResponse`
    - _Requirements: 2.1, 2.2, 2.3, 4.1, 4.2, 4.3_

  - [ ]* 3.3 Escrever property test para `TicketService` — Property 4
    - **Property 4: Ticket criado preserva dados do cliente**
    - Para qualquer combinação de `clientName`, `clientEmail` e `question`, o ticket persistido deve conter exatamente os mesmos valores; a resposta deve conter `ticketId`, `message` e `confidenceScore`
    - Usar mock de `TicketRepository` e `@Property` com `@ForAll String` no `TicketServiceTest`
    - `// Feature: supportbot-faq, Property 4: Ticket criado preserva dados do cliente`
    - **Validates: Requirements 2.1, 2.2**

  - [ ]* 3.4 Escrever property test para `TicketService` — Property 8
    - **Property 8: Listagem de tickets ordenada por createdAt decrescente**
    - Para qualquer lista de tickets com timestamps distintos, `listAll()` deve retornar a lista ordenada por `createdAt` decrescente
    - Usar `@Property` com `@ForAll List<Ticket>` no `TicketServiceTest`
    - `// Feature: supportbot-faq, Property 8: Listagem de tickets está ordenada por createdAt decrescente`
    - **Validates: Requirements 4.1**

  - [ ]* 3.5 Escrever property test para `TicketService` — Property 9
    - **Property 9: Round-trip de busca de ticket por id**
    - Para qualquer ticket criado, `findById(id)` deve retornar os mesmos dados; para id inexistente, deve lançar `TicketNotFoundException`
    - `// Feature: supportbot-faq, Property 9: Round-trip de busca de ticket por id`
    - **Validates: Requirements 4.2, 4.3**

- [ ] 4. Implementar `FaqService` (camada de aplicação)
  - [ ] 4.1 Implementar DTOs: `AskRequest(String clientName, String clientEmail, String question)`, `AskResponse(String answer, String category, double confidenceScore)`, `CreateFaqRequest(String question, String answer, Category category)`, `FaqEntryResponse(Long id, String question, String answer, String category)`
    - _Requirements: 1.1, 3.1_

  - [ ] 4.2 Implementar `FaqService` com injeção de `FaqRepository`, `SimilarityCalculator` e `TicketService` por construtor
    - Método `ask(AskRequest request)`: validar campos obrigatórios e `question` não-whitespace (lançar exceção com mensagem descritiva se inválido), buscar todas as entries, calcular `MatchResult`; se `score >= 0.5` retornar `AskResponse`; se `score < 0.5` delegar a `TicketService.createTicket` e retornar `TicketResponse`
    - Método `create(CreateFaqRequest request)`: persistir via `FaqRepository`, retornar `FaqEntryResponse` com id gerado
    - Método `listAll(Optional<Category> category)`: se category presente, usar `findByCategory`; senão `findAll`; mapear para `List<FaqEntryResponse>`
    - Método `delete(Long id)`: verificar existência via `existsById`, lançar `FaqEntryNotFoundException` se ausente, chamar `deleteById`
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.6, 2.1, 3.1, 3.2, 3.3, 3.4, 3.5_

  - [ ]* 4.3 Escrever property test para `FaqService` — Property 2
    - **Property 2: Pergunta idêntica à FAQ_Entry retorna resposta e category corretas**
    - Para qualquer `FaqEntry` cadastrada, `ask` com a pergunta exata deve retornar `confidenceScore >= 0.5`, `answer` e `category` corretos
    - Usar mock de `FaqRepository` e `@Property` com `@ForAll FaqEntry` no `FaqServiceTest`
    - `// Feature: supportbot-faq, Property 2: Pergunta idêntica à FAQ_Entry retorna resposta e category corretas`
    - **Validates: Requirements 1.2, 1.4**

  - [ ]* 4.4 Escrever property test para `FaqService` — Property 5
    - **Property 5: Round-trip de criação de FAQ_Entry**
    - Para qualquer `CreateFaqRequest` válido, a entry retornada deve conter os mesmos `question`, `answer` e `category`, além de `id` não-nulo
    - `// Feature: supportbot-faq, Property 5: Round-trip de criação de FAQ_Entry`
    - **Validates: Requirements 3.1, 3.2**

  - [ ]* 4.5 Escrever property test para `FaqService` — Property 6
    - **Property 6: Filtro por category retorna apenas entries da category solicitada**
    - Para qualquer `Category` e qualquer conjunto de entries, `listAll(Optional.of(category))` deve retornar apenas entries com aquela category
    - `// Feature: supportbot-faq, Property 6: Filtro por category retorna apenas entries da category solicitada`
    - **Validates: Requirements 3.3**

  - [ ]* 4.6 Escrever property test para `FaqService` — Property 7
    - **Property 7: Delete remove a entry e torna id inexistente**
    - Para qualquer entry existente, após `delete(id)` a entry não deve mais ser encontrada; para id inexistente, deve lançar `FaqEntryNotFoundException`
    - `// Feature: supportbot-faq, Property 7: Delete remove a entry e torna id inexistente`
    - **Validates: Requirements 3.4, 3.5**

- [ ] 5. Checkpoint — Verificar lógica de domínio e aplicação
  - Garantir que todos os testes de unidade e property tests das tarefas 2, 3 e 4 passam.
  - Verificar que nenhuma classe de domínio ou aplicação importa classes de `org.springframework` ou `jakarta.persistence`.
  - Perguntar ao usuário se há dúvidas antes de prosseguir para a infraestrutura.

- [ ] 6. Implementar camada de infraestrutura — Persistência
  - [ ] 6.1 Criar entidade JPA `FaqEntryJpaEntity` com `@Entity`, `@Table(name = "faq_entries")` e campos mapeados; implementar métodos de conversão para/de `FaqEntry`
    - _Requirements: 5.2, 5.3_

  - [ ] 6.2 Criar entidade JPA `TicketJpaEntity` com `@Entity`, `@Table(name = "tickets")` e campos mapeados; implementar métodos de conversão para/de `Ticket`
    - _Requirements: 5.2, 5.3_

  - [ ] 6.3 Criar `FaqJpaRepository extends JpaRepository<FaqEntryJpaEntity, Long>` com query para `findByCategory`
    - _Requirements: 5.2_

  - [ ] 6.4 Criar `TicketJpaRepository extends JpaRepository<TicketJpaEntity, Long>` com query para `findAllByOrderByCreatedAtDesc`
    - _Requirements: 5.2_

  - [ ] 6.5 Implementar `FaqJpaAdapter` que implementa `FaqRepository`, injeta `FaqJpaRepository` por construtor e converte entre entidades JPA e de domínio em todos os métodos
    - _Requirements: 5.2, 5.3_

  - [ ] 6.6 Implementar `TicketJpaAdapter` que implementa `TicketRepository`, injeta `TicketJpaRepository` por construtor e converte entre entidades JPA e de domínio em todos os métodos
    - _Requirements: 5.2, 5.3_

  - [ ] 6.7 Configurar `application.properties` com datasource H2, `ddl-auto=create-drop` e console H2 habilitado
    - _Requirements: 5.5_

  - [ ]* 6.8 Escrever testes de unidade para os adapters `FaqJpaAdapter` e `TicketJpaAdapter`
    - Testar conversão entre entidades JPA e de domínio para todos os campos
    - _Requirements: 5.2, 5.3_

- [ ] 7. Implementar camada de infraestrutura — Controllers e tratamento de erros
  - [ ] 7.1 Implementar `FaqController` com `@RestController`, `@RequestMapping("/api/faq")`, injeção de `FaqService` por construtor
    - `POST /api/faq/ask` → `FaqService.ask`, retornar HTTP 200
    - `POST /api/faq` → `FaqService.create`, retornar HTTP 201
    - `GET /api/faq` → `FaqService.listAll(Optional.ofNullable(category))`, retornar HTTP 200
    - `DELETE /api/faq/{id}` → `FaqService.delete`, retornar HTTP 204
    - _Requirements: 1.1, 3.1, 3.2, 3.3, 3.4_

  - [ ] 7.2 Implementar `TicketController` com `@RestController`, `@RequestMapping("/api/tickets")`, injeção de `TicketService` por construtor
    - `GET /api/tickets` → `TicketService.listAll`, retornar HTTP 200
    - `GET /api/tickets/{id}` → `TicketService.findById`, retornar HTTP 200
    - _Requirements: 4.1, 4.2_

  - [ ] 7.3 Implementar `GlobalExceptionHandler` com `@ControllerAdvice`
    - Mapear `FaqEntryNotFoundException` → HTTP 404
    - Mapear `TicketNotFoundException` → HTTP 404
    - Mapear exceções de validação de campos obrigatórios → HTTP 400
    - Mapear falha de persistência de ticket → HTTP 500
    - Todas as respostas de erro no formato `{"error": "<mensagem>"}`
    - _Requirements: 1.5, 1.6, 2.4, 3.5, 3.6, 4.3_

  - [ ]* 7.4 Escrever property test para `FaqController` — Property 3
    - **Property 3: Requisição inválida sempre retorna HTTP 400**
    - Para qualquer requisição `POST /api/faq/ask` com campos obrigatórios nulos ou `question` apenas whitespace, o controller deve retornar HTTP 400
    - Usar `MockMvc` e `@Property` gerando requests com campos nulos/whitespace no `FaqControllerTest`
    - `// Feature: supportbot-faq, Property 3: Requisição inválida sempre retorna HTTP 400`
    - **Validates: Requirements 1.5, 1.6**

- [ ] 8. Implementar testes de integração e smoke tests
  - [ ] 8.1 Escrever smoke test com `@SpringBootTest` verificando que a aplicação sobe e as tabelas `faq_entries` e `tickets` são criadas no H2
    - _Requirements: 5.5_

  - [ ]* 8.2 Escrever testes de integração para o fluxo completo de `POST /api/faq/ask`
    - Cenário com score alto: cadastrar FAQ entry, enviar pergunta idêntica, verificar `AskResponse` com `confidenceScore >= 0.5`
    - Cenário com score baixo: enviar pergunta sem FAQ cadastrada, verificar criação de ticket e `TicketResponse`
    - Usar `@SpringBootTest` com `MockMvc` e H2 em memória
    - _Requirements: 1.1, 1.2, 2.1, 2.2_

  - [ ]* 8.3 Escrever testes de integração para CRUD de FAQ entries
    - Testar `POST /api/faq`, `GET /api/faq`, `GET /api/faq?category=DELIVERY`, `DELETE /api/faq/{id}`
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5_

- [ ] 9. Checkpoint final — Garantir que todos os testes passam
  - Garantir que todos os testes (unidade, property-based e integração) passam.
  - Verificar que nenhuma entidade JPA é exposta diretamente nas respostas HTTP.
  - Perguntar ao usuário se há dúvidas ou ajustes antes de concluir.

## Notes

- Tarefas marcadas com `*` são opcionais e podem ser puladas para um MVP mais rápido
- Cada tarefa referencia os requisitos específicos para rastreabilidade
- Os property tests usam jqwik (`@Property`, `@ForAll`) com mínimo de 100 iterações
- Cada property test deve incluir o comentário `// Feature: supportbot-faq, Property N: <texto>`
- Entidades JPA ficam restritas à camada `infrastructure/`; entidades de domínio ficam em `domain/`
- Toda injeção de dependência deve ser por construtor, sem `@Autowired` em campos
