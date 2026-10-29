# Quiz Master - Arquitetura N2

Este repositório documenta uma proposta de evolução arquitetural do Quiz Master para DDD/Arquitetura Hexagonal e uma camada de mensageria. Os diagramas representam uma **arquitetura-alvo**; Outbox, RabbitMQ, publisher, worker, portas e adaptadores são propostas e não devem ser confundidos com funcionalidades já implementadas.

Projeto de implementação: <https://github.com/GustavodePaulaGorges/DesWeb2026-2>

## Estrutura deste repositório

```text
quiz-master-arquitetura-n2/
├── README.md
├── arquitetura/
│   ├── src/
│   │   ├── bootstrap/
│   │   ├── config/
│   │   ├── infrastructure/
│   │   │   ├── database/
│   │   │   │   └── migrations/
│   │   │   ├── filesystem/
│   │   │   ├── logging/
│   │   │   ├── messaging/
│   │   │   │   └── rabbitmq/
│   │   │   └── security/
│   │   ├── modules/
│   │   │   ├── discovery/
│   │   │   ├── identity-profile/
│   │   │   ├── notifications/
│   │   │   ├── outbox/
│   │   │   ├── quiz-management/
│   │   │   └── social-interaction/
│   │   └── shared/
│   │       ├── application/ports/
│   │       └── domain/
│   └── tests/
│       ├── e2e/
│       ├── integration/
│       └── unit/
└── c4/
    ├── c1-contexto.puml
    ├── c2-containers-mensageria.puml
    └── c3-componentes-api-ddd.puml
```

## Diagramas C4

- `c4/c1-contexto.puml`: usuário e sistema Quiz Master.
- `c4/c2-containers-mensageria.puml`: SPA, API, MySQL e o fluxo de mensageria proposto.
- `c4/c3-componentes-api-ddd.puml`: componentes internos da API, contratos e adaptadores de infraestrutura.

Os arquivos usam C4-PlantUML por meio de `!include <C4/C4_...>`. Abra-os em uma extensão PlantUML ou em um renderizador compatível com a biblioteca C4-PlantUML.

## Estrutura de pastas em `arquitetura/`

A estrutura abaixo foi criada como esqueleto para a proposta de organização futura do backend. Ela mantém um monólito modular e separa domínio, casos de uso, contratos e adaptadores. As pastas estão vazias; os nomes de arquivos apresentados anteriormente eram exemplos da proposta e ainda não foram criados.

```text
arquitetura/
├── src/
│   ├── bootstrap/
│   ├── config/
│   ├── infrastructure/
│   │   ├── database/
│   │   │   └── migrations/
│   │   ├── filesystem/
│   │   ├── logging/
│   │   ├── messaging/
│   │   │   └── rabbitmq/
│   │   └── security/
│   ├── modules/
│   │   ├── discovery/
│   │   │   ├── adapters/
│   │   │   │   ├── inbound/http/
│   │   │   │   └── outbound/persistence/
│   │   │   └── application/
│   │   │       ├── ports/
│   │   │       └── use-cases/
│   │   ├── identity-profile/
│   │   │   ├── adapters/
│   │   │   │   ├── inbound/http/
│   │   │   │   └── outbound/
│   │   │   │       ├── persistence/
│   │   │   │       └── security/
│   │   │   ├── application/
│   │   │   │   ├── ports/
│   │   │   │   └── use-cases/
│   │   │   └── domain/
│   │   │       ├── entities/
│   │   │       └── value-objects/
│   │   ├── notifications/
│   │   │   ├── adapters/
│   │   │   │   ├── inbound/messaging/
│   │   │   │   └── outbound/persistence/
│   │   │   ├── application/
│   │   │   │   ├── handlers/
│   │   │   │   └── ports/
│   │   │   └── domain/entities/
│   │   ├── outbox/
│   │   │   ├── adapters/outbound/
│   │   │   │   ├── messaging/
│   │   │   │   └── persistence/
│   │   │   └── application/
│   │   │       ├── ports/
│   │   │       └── use-cases/
│   │   ├── quiz-management/
│   │   │   ├── adapters/
│   │   │   │   ├── inbound/http/
│   │   │   │   └── outbound/persistence/
│   │   │   ├── application/
│   │   │   │   ├── ports/
│   │   │   │   └── use-cases/
│   │   │   └── domain/
│   │   │       ├── entities/
│   │   │       └── events/
│   │   └── social-interaction/
│   │       ├── adapters/
│   │       │   ├── inbound/http/
│   │       │   └── outbound/persistence/
│   │       ├── application/
│   │       │   ├── ports/
│   │       │   └── use-cases/
│   │       └── domain/entities/
│   ├── shared/
│   │   ├── domain/
│   │   └── application/
│   │       └── ports/
└── tests/
    ├── e2e/
    ├── integration/
    └── unit/
```

### Responsabilidade dos módulos

- **Identidade e perfil:** cadastro, autenticação, dados do usuário e autoria.
- **Gestão de quizzes:** criação, edição, consulta e exclusão de quizzes e alternativas; `Quiz` é a raiz sugerida do agregado.
- **Interação social:** curtidas e comentários.
- **Descoberta:** feed e pesquisa; separado do módulo de usuário.
- **Outbox:** leitura dos eventos confirmados no MySQL e publicação no broker.
- **Notificações:** consumo dos eventos e gravação das notificações de forma idempotente.

O domínio não deve importar Express, Sequelize, filesystem ou RabbitMQ. Os casos de uso coordenam o fluxo e dependem de contratos; os adaptadores implementam esses contratos. A migração pode ocorrer gradualmente, sem reescrever todo o backend de uma vez.

## Mensageria proposta

O fluxo previsto é: (1) a API persiste a ação de negócio e o evento na tabela `outbox_events` na mesma transação MySQL; (2) o Outbox Publisher lê eventos pendentes e os publica no RabbitMQ; (3) o worker consome a fila e grava notificações no MySQL. O worker deve confirmar a mensagem depois de persistir a notificação, e usar `eventId` ou uma chave única para evitar duplicação em caso de reentrega. Também se recomenda retry e dead-letter queue (DLQ).

Eventos iniciais sugeridos: `quiz.liked`, `comment.created`, `quiz.published` e `quiz.updated`. O payload deve carregar identificadores e versão de schema, evitando senhas, tokens e outros dados privados desnecessários.
