# SupportBot

Assistente de suporte ao cliente para responder dúvidas frequentes sobre **prazo de entrega**, **troca/devolução** e **pagamentos**. Quando não souber responder, abre um ticket com os dados do cliente.

## Funcionalidades

- Resposta imediata 24/7 para as principais categorias de FAQ
- Abertura automática de tickets para dúvidas fora do escopo
- Redução do volume de atendimento humano para perguntas repetitivas

### Categorias de FAQ cobertas

- **Prazo de Entrega** — prazo estimado por região, rastreamento, pedidos atrasados
- **Troca e Devolução** — como solicitar, prazo, responsabilidade pelo frete
- **Pagamentos** — formas aceitas, pagamento não aprovado, reembolso

## Tech Stack

- Java 17
- Spring Boot
- H2 (banco em memória para desenvolvimento e testes)

## Arquitetura

O projeto segue a **Arquitetura Hexagonal** (Ports & Adapters):

```
src/
  domain/         # Entidades, value objects, serviços de domínio, ports (interfaces)
  application/    # Casos de uso / serviços de aplicação
  infrastructure/ # Adaptadores: HTTP, DB, controllers, etc.
```

## Como rodar

```bash
./mvnw spring-boot:run
```

O console do H2 estará disponível em `http://localhost:8080/h2-console` durante o desenvolvimento.

## Configuração

As configurações de ambiente ficam em `src/main/resources/application.properties`. Nunca hardcode valores específicos de ambiente no código.
