
## Estrutura do Projeto & Convenções de Código

### Arquitetura
Este projeto segue a **Arquitetura Hexagonal** (Ports & Adapters) com três camadas:

- **Domínio** — Lógica de negócio central e entidades. Sem dependências de frameworks externos ou infraestrutura.
- **Aplicação** — Casos de uso e orquestração. Depende apenas de interfaces do domínio (ports).
- **Infraestrutura** — Adaptadores para sistemas externos (bancos de dados, APIs, mensageria, controllers HTTP). Implementa os ports definidos no domínio.

A estrutura de pastas deve refletir essa separação:
```
src/
  domain/        # Entidades, value objects, serviços de domínio, ports (interfaces)
  application/   # Casos de uso / serviços de aplicação
  infrastructure/ # Adaptadores: HTTP, DB, APIs externas, etc.
```

### Estilo de Código
- **Clean Code**: Funções e variáveis devem ter nomes claros e que revelem a intenção. Mantenha funções pequenas e focadas em uma única responsabilidade.
- **Return Early Pattern**: Valide pré-condições e retorne (ou lance exceção) cedo para evitar condicionais profundamente aninhadas.
- **camelCase**: Use camelCase para variáveis, funções e métodos. Use PascalCase para classes e interfaces.

### Regras Gerais
- Dependa de abstrações (interfaces/ports), não de implementações concretas.
- Mantenha a camada de domínio livre de código específico de frameworks.
- Adaptadores de infraestrutura devem implementar os ports definidos no domínio.
- Evite vazar preocupações de infraestrutura (ex: modelos ORM, objetos de requisição HTTP) para as camadas de domínio ou aplicação.
