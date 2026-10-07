# `plan.md` — Arquitetura e Design de Engenharia

## 1. Visão Geral da Arquitetura

O sistema adota o padrão **MVC (Model-View-Controller)** estendido com uma camada intermediária de **Service**, isolando completamente as responsabilidades da entrada de requisições, regras de negócio e acesso ao banco de dados em memória.

```
[ HTTP Client ]
      │
      ▼
[ Controller Layer ]  ───> Recebe requisições HTTP, valida DTOs e mapeia exceções
      │
      ▼
[ Service Layer ]     ───> Executa regras de negócio (RN-001 a RN-011) e cálculos
      │
      ▼
[ Repository / Model ] ───> Entidades JPA e persistência no H2 Database em memória

```

---

## 2. Árvore Física de Arquivos e Pastas

Todos os arquivos de código, pacotes, interfaces e classes devem ser escritos estritamente em **inglês**.

```
src/main/java/com/parking/api/
├── ParkingApiApplication.java
├── config/
│   └── AppConfig.java
├── controller/
│   ├── TicketController.java
│   └── ReportController.java
├── service/
│   ├── TicketService.java
│   └── ReportService.java
├── repository/
│   └── TicketRepository.java
├── model/
│   ├── Ticket.java
│   └── TicketStatus.java
├── dto/
│   ├── request/
│   │   ├── CreateTicketRequest.java
│   │   └── CloseTicketRequest.java
│   └── response/
│       ├── TicketResponse.java
│       ├── DailyReportResponse.java
│       └── ErrorResponse.java
└── exception/
    ├── GlobalExceptionHandler.java
    ├── InvalidPlateException.java
    ├── InvalidDateTimeException.java
    ├── TicketAlreadyOpenException.java
    ├── TicketNotFoundException.java
    ├── TicketAlreadyClosedException.java
    └── TicketNotOpenException.java

```

### Responsabilidades dos Módulos:

* **`controller`**: Expõe os endpoints REST e realiza a mediação de entrada/saída transformando payloads JSON/DTOs.
* **`service`**: Contém a inteligência da aplicação, validações de domínio, cálculos de permanência/tarifas e gerenciamento de transações.
* **`repository`**: Interface Spring Data JPA para interação com a tabela `tickets` no H2 Database.
* **`model`**: Entidades ORM e Enums representando o domínio persistido.
* **`dto`**: Objetos de transferência de dados anotados com Jackson para garantir o contrato em `snake_case`.
* **`exception`**: Captura global de exceções anotada com `@RestControllerAdvice` para garantir respostas padronizadas no formato `{"erro": "codigo"}`.

---

## 3. Modelagem de Dados & Schemas Conceituais

### 3.1. Enum: `TicketStatus`

* `OPEN`: Bilhete ativo em parqueamento.
* `CLOSED`: Bilhete encerrado com pagamento calculado.
* `CANCELED`: Bilhete cancelado sem cobrança.

### 3.2. Entidade: `Ticket` (Tabela `tickets` no H2)

```java
@Entity
@Table(name = "tickets")
public class Ticket {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 7)
    private String plate;

    @Column(nullable = false)
    private OffsetDateTime entryTime;

    @Column
    private OffsetDateTime exitTime;

    @Column
    private Integer durationMinutes;

    @Column
    private Long amountCents;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private TicketStatus status;
}

```

---

## 4. DTOs e Mapeamento do Contrato JSON (`snake_case`)

Todos os DTOs utilizam anotações do Jackson (`@JsonProperty`) para garantir aderência ao contrato público da API.

### 4.1. `CreateTicketRequest`

```java
public record CreateTicketRequest(
    @JsonProperty("placa") String plate,
    @JsonProperty("entrada") String entry
) {}

```

### 4.2. `CloseTicketRequest`

```java
public record CloseTicketRequest(
    @JsonProperty("saida") String exit
) {}

```

### 4.3. `TicketResponse`

```java
@JsonInclude(JsonInclude.Include.NON_NULL)
public record TicketResponse(
    @JsonProperty("id") Long id,
    @JsonProperty("placa") String plate,
    @JsonProperty("entrada") String entry,
    @JsonProperty("saida") String exit,
    @JsonProperty("minutos") Integer durationMinutes,
    @JsonProperty("valor_centavos") Long amountCents,
    @JsonProperty("status") String status
) {}

```

### 4.4. `DailyReportResponse`

```java
public record DailyReportResponse(
    @JsonProperty("data") String date,
    @JsonProperty("total_bilhetes") Long totalTickets,
    @JsonProperty("faturamento_centavos") Long totalRevenueCents,
    @JsonProperty("tempo_medio_minutos") Long averageTimeMinutes
) {}

```

### 4.5. `ErrorResponse`

```java
public record ErrorResponse(
    @JsonProperty("erro") String error
) {}

```

---

## 5. Algoritmos Críticos e Fórmulas de Cálculo

### 5.1. Validação de Placa (`RN-001`)

* Expressão Regular: `^[A-Z0-9]{7}$`
* Caso falhe ou seja `null`: Lançar `InvalidPlateException` $\rightarrow$ `HTTP 422 {"erro": "placa_invalida"}`.

### 5.2. Cálculo de Duração e Tarifa (`RN-004` a `RN-008`)

Dada a entrada $T_{\text{entry}}$ e a saída $T_{\text{exit}}$ em `OffsetDateTime` (fuso `-03:00`):

1. **Duração Total em Minutos:**

$$\text{minutos} = \text{ChronoUnit.MINUTES.between}(T_{\text{entry}}, T_{\text{exit}})$$


* Se $T_{\text{exit}} < T_{\text{entry}}$: Lançar `InvalidDateTimeException` $\rightarrow$ `HTTP 422 {"erro": "saida_invalida"}`.


2. **Verificação de Tolerância (`RN-004`):**

$$\text{valor\_centavos} = \begin{cases} 0, & \text{se } \text{minutos} \le 15 \\ \text{Cálculo Integral}, & \text{se } \text{minutos} > 15 \end{cases}$$


3. **Cálculo de Frações e Valor Bruto (`RN-005`, `RN-006`, `RN-007`):**

$$\text{fracoes} = \left\lceil \frac{\text{minutos}}{15} \right\rceil$$


$$\text{valor\_bruto} = \text{fracoes} \times 100$$


4. **Aplicação do Teto Diário (`RN-008`):**

$$\text{valor\_centavos} = \min(\text{valor\_bruto}, 8000)$$



### 5.3. Processamento do Relatório Diário (`RN-010`, `RN-011`)

Dado a data informada no parâmetro `data` (`YYYY-MM-DD`):

1. Filtrar bilhetes no banco onde:
* `status == TicketStatus.CLOSED`
* $\text{LocalDate}(T_{\text{exit}} \text{ no fuso } -03:00) == \text{data}$


2. Se a lista de bilhetes filtrados for vazia:
* `total_bilhetes` = 0
* `faturamento_centavos` = 0
* `tempo_medio_minutos` = 0


3. Caso existam bilhetes encerrados no dia:

$$\text{faturamento\_centavos} = \sum \text{amountCents}$$


$$\text{tempo\_medio\_minutos} = \text{Math.round}\left( \frac{\sum \text{durationMinutes}}{N} \right) \quad (\text{Arredondamento Half-Up})$$



---

## 6. Tratamento Global de Exceções (`GlobalExceptionHandler`)

A classe `@RestControllerAdvice` intercepta exceções do sistema e mapeia para os status HTTP e payloads JSON conforme especificado:

| Exceção Lançada | Status HTTP | Payload JSON Retornado |
| --- | --- | --- |
| `InvalidPlateException` | `422 Unprocessable Entity` | `{"erro": "placa_invalida"}` |
| `InvalidDateTimeException` | `422 Unprocessable Entity` | `{"erro": "entrada_invalida"}` ou `{"erro": "saida_invalida"}` ou `{"erro": "data_invalida"}` |
| `TicketAlreadyOpenException` | `409 Conflict` | `{"erro": "bilhete_em_aberto"}` |
| `TicketNotFoundException` | `404 Not Found` | `{"erro": "bilhete_nao_encontrado"}` |
| `TicketAlreadyClosedException` | `409 Conflict` | `{"erro": "bilhete_ja_encerrado"}` |
| `TicketNotOpenException` | `409 Conflict` | `{"erro": "bilhete_nao_aberto"}` |

---