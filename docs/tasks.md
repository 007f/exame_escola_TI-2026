# tasks.md — Plano de Ação e Decomposição Atômica

Este documento detalha o plano de execução passo a passo para a implementação da API de estacionamento rotativo, estruturado em marcos de entrega (*Milestones*) com rastreamento cruzado direto para a `spec.md` (Casos de Uso e Regras de Negócio) e a `tests.md` (Cenários de Teste TDD)[cite: 1].

---

## 🚩 Milestone 1: Configuração do Projeto e Infraestrutura Base

- [ ] **Task 1.1 — Inicialização e Configuração do Servidor Spring Boot**
  - Criar o projeto Java Spring Boot utilizando Maven/Gradle com as dependências: `Spring Web`, `Spring Data JPA` e `H2 Database`.
  - Configurar a porta de escuta em `application.properties` para a variável `PORTA_SERVICO` com valor padrão `8001` (`server.port=8001`).
  - *[Ref: constitution.md]*

- [ ] **Task 1.2 — Padronização Global de Mapeamento JSON e Fuso Horário**
  - Criar `AppConfig.java` para registrar o `ObjectMapper` com a estratégia de nomenclatura em `snake_case`.
  - Configurar o deserializador/serializador global para datas em formato ISO-8601 respeitando o fuso horário `-03:00`.
  - *[Ref: constitution.md]*

---

## 🚩 Milestone 2: Modelo de Domínio, Exceções e Tratamento Global de Erros

- [ ] **Task 2.1 — Criação da Entidade de Domínio e Enum de Status**
  - Implementar o enum `TicketStatus` (`OPEN`, `CLOSED`, `CANCELED`).
  - Implementar a entidade `Ticket` mapeando a tabela `tickets` no H2 Database em memória (com id numérico incremental, campos de placa, datas, minutos e centavos)[cite: 1].
  - *[Ref: plan.md]*

- [ ] **Task 2.2 — Hierarquia de Exceções de Domínio**
  - Criar as exceções customizadas do sistema:
    - `InvalidPlateException`
    - `InvalidDateTimeException`
    - `TicketAlreadyOpenException`
    - `TicketNotFoundException`
    - `TicketAlreadyClosedException`
    - `TicketNotOpenException`
  - *[Ref: plan.md]*

- [ ] **Task 2.3 — Handler Global de Exceções (Contrato Estrito de Erro)**
  - Implementar `GlobalExceptionHandler` anotado com `@RestControllerAdvice`.
  - Mapear cada exceção criada na Task 2.2 para o seu respetivo código HTTP (`404`, `409`, `422`) retornando **exclusivamente** o JSON `{"erro": "codigo_do_erro"}` sem os atributos padrão do Spring Boot (`timestamp`, `path`, etc.)[cite: 1].
  - *[Ref: constitution.md, spec.md - Tabela de Erros]*

---

## 🚩 Milestone 3: Camada de Acesso a Dados e DTOs

- [ ] **Task 3.1 — Criação do Repositório JPA**
  - Criar a interface `TicketRepository` estendendo `JpaRepository<Ticket, Long>`.
  - Adicionar métodos de consulta customizados:
    - `findByPlateAndStatus(String plate, TicketStatus status)`
    - `findByStatusOrderByEntryTimeDesc(TicketStatus status)`
    - `findByPlateOrderByEntryTimeDesc(String plate)`
    - `findByStatusAndExitTimeBetween(TicketStatus status, OffsetDateTime start, OffsetDateTime end)`
  - *[Ref: plan.md]*

- [ ] **Task 3.2 — DTOs de Entrada e Saída (Contrato Publico `snake_case`)**
  - Implementar os DTOs `CreateTicketRequest`, `CloseTicketRequest`, `TicketResponse`, `DailyReportResponse` e `ErrorResponse` utilizando anotações Jackson (`@JsonProperty`)[cite: 1].
  - *[Ref: plan.md]*

---

## 🚩 Milestone 4: Implementação das Regras de Negócio e Serviços (Ciclo TDD)

- [ ] **Task 4.1 — Serviço de Abertura de Bilhetes (`UC1`, `UC8`)**
  - Implementar `TicketService.createTicket(CreateTicketRequest request)`.
  - Validar regex da placa `^[A-Z0-9]{7}$` (`RN-001`).
  - Validar/parsear campo `entrada` com fuso `-03:00` ou atribuir o horário atual (`RN-002`).
  - Garantir a unicidade de vaga ativa por placa (`RN-003`, `UC8`).
  - *[Ref: UC1, UC8, RN-001, RN-002, RN-003 | Testes: T01, T02, T03, T04, T05, T06, T07, T08, T09, T10]*

- [ ] **Task 4.2 — Serviço de Encerramento e Cálculo de Tarifas (`UC2`, `UC7`)**
  - Implementar `TicketService.closeTicket(Long id, CloseTicketRequest request)`.
  - Validar existência do bilhete e status `OPEN`.
  - Calcular permanência em minutos e validar se a saída é posterior à entrada.
  - Aplicar regra de tolerância gratuita até 15 minutos (`RN-004`, `UC7`).
  - Calcular valor por frações de 15 min arredondando para cima (`RN-005`, `RN-006`, `RN-007`).
  - Aplicar trava de teto diário de 8000 centavos (`RN-008`) e persistir valor inteiro em centavos (`RN-009`).
  - *[Ref: UC2, UC7, RN-004, RN-005, RN-006, RN-007, RN-008, RN-009 | Testes: T11, T12, T13, T14, T15, T16, T17, T18]*

- [ ] **Task 4.3 — Serviço de Listagem de Bilhetes Ativos (`UC3`)**
  - Implementar `TicketService.listActiveTickets()`.
  - Retornar bilhetes com status `OPEN` ordenados pela entrada mais recente.
  - *[Ref: UC3 | Testes: T19, T20]*

- [ ] **Task 4.4 — Serviço de Relatórios Diários e Estatísticas (`UC4`)**
  - Implementar `ReportService.getDailyReport(String date)`.
  - Validar formato da data `AAAA-MM-DD`.
  - Filtrar bilhetes encerrados no dia solicitado no fuso `-03:00` (`RN-010`).
  - Somar faturamento em centavos e calcular o tempo médio em minutos com arredondamento *Half-Up* (`RN-011`).
  - *[Ref: UC4, RN-010, RN-011 | Testes: T21, T22, T23, T24]*

- [ ] **Task 4.5 — Serviço de Cancelamento de Bilhete (`UC5`)**
  - Implementar `TicketService.cancelTicket(Long id)`.
  - Validar se o bilhete está no status `OPEN`.
  - Atualizar o status para `CANCELED` e liberar a vaga/placa para novas aberturas.
  - *[Ref: UC5, UC8 | Testes: T25, T26, T27]*

- [ ] **Task 4.6 — Serviço de Histórico por Placa (`UC6`)**
  - Implementar `TicketService.getTicketsByPlate(String plate)`.
  - Retornar histórico de todos os bilhetes registrados para a placa ordenados de forma decrescente.
  - *[Ref: UC6 | Testes: T28, T29]*

---

## 🚩 Milestone 5: Controladores REST e Testes de Integração E2E

- [ ] **Task 5.1 — Implementação do `TicketController`**
  - Mapear os endpoints HTTP para as rotas `/bilhetes`:
    - `POST /bilhetes`
    - `POST /bilhetes/{id}/encerramento`
    - `POST /bilhetes/{id}/cancelamento`
    - `GET /bilhetes/ativos`
    - `GET /bilhetes?placa={placa}`
  - *[Ref: UC1, UC2, UC3, UC5, UC6]*

- [ ] **Task 5.2 — Implementação do `ReportController`**
  - Mapear o endpoint HTTP para a rota `/relatorios/diario`:
    - `GET /relatorios/diario?data={data}`
  - *[Ref: UC4]*

- [ ] **Task 5.3 — Execução da Suíte Completa de Testes MockMvc (TDD Checkpoint)**
  - Executar a suíte completa de 29 testes de integração automatizados (`MockMvc`) definidos na `tests.md`.
  - Validar a passagem de 100% dos cenários felizes e dos casos de borda com cobertura de código de 100% nas regras de negócio.
  - *[Ref: tests.md - Testes T01 a T29]*