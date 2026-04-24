
## Tech Stack

- **Linguagem**: Java 17
- **Framework**: Spring Boot
- **Banco de Dados**: H2 (em memória, para desenvolvimento e testes)

## Convenções Principais

- Use a auto-configuração do Spring Boot; evite configuração manual de beans onde as convenções já atendem.
- Defina interfaces de repositório usando Spring Data JPA; mantenha anotações JPA/ORM restritas às entidades da camada de infraestrutura.
- Use `application.properties` ou `application.yml` para configuração; nunca hardcode valores específicos de ambiente.
- O console do H2 pode ser habilitado em desenvolvimento local (`spring.h2.console.enabled=true`), mas deve ser desabilitado em perfis de produção.
- Use injeção por construtor (não injeção por campo) em todos os beans gerenciados pelo Spring.
- DTOs e objetos de domínio são separados — não exponha entidades JPA diretamente nas respostas HTTP.
