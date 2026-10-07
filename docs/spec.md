# `spec.md` — Especificação de Requisitos e Casos de Uso

## 1. Parâmetros da Variante

| Parâmetro | Valor | Descrição |
| --- | --- | --- |
| `TARIFA_HORA_CENTAVOS` | `400` | Valor da hora cheia (R$ 4,00)|
| `FRACAO_MINUTOS` | `15` | Granularidade mínima de cobrança em minutos|
| `VALOR_FRACAO_CENTAVOS` | `100` | $400 \div (60 \div 15) = 100$ centavos por fração (R$ 1,00)|
| `TETO_DIARIO_CENTAVOS` | `8000` | Valor máximo cobrado por bilhete (R$ 80,00)|
| `TOLERANCIA_MINUTOS` | `15` | Minutos iniciais isentos de cobrança|
| `PORTA_SERVICO` | `8001` | Porta de escuta da aplicação HTTP|


---

## 2. Regras de Negócio (RN)

* **RN-001 — Validação de Placa**: A placa deve conter exatamente 7 caracteres alfanuméricos maiúsculos (`^[A-Z0-9]{7}$`). Ausente ou inválida gera `HTTP 422` com `{"erro": "placa_invalida"}`.


* **RN-002 — Entrada Customizada**: O campo `entrada` é opcional na abertura. Se enviado, deve ser uma string ISO-8601 válida com fuso `-03:00`. Inválido ou sem fuso gera `HTTP 422` com `{"erro": "entrada_invalida"}`. Se ausente, assume o instante atual do servidor (`now`).


* **RN-003 — Unicidade de Vaga Ativa (UC8)**: Uma placa só pode possuir um bilhete com `status: "aberto"` por vez. Tentativa de abrir bilhete para placa ocupada gera `HTTP 409` com `{"erro": "bilhete_em_aberto"}`.


* **RN-004 — Tolerância Gratuita (UC7)**:
* Duração em minutos $\le 15$: `valor_centavos = 0`.


* Duração em minutos $> 15$: Cobra-se integralmente desde o $1º$ minuto (a tolerância **não** é deduzida).




* **RN-005 — Arredondamento de Fração**: O tempo cobrado em minutos é dividido por 15 e arredondado sempre **para cima** ($\lceil \text{minutos} / 15 \rceil$).


* **RN-006 — Valor por Fração**: Cada fração de 15 minutos custa $400 \div (60 \div 15) = 100$ centavos.


* **RN-007 — Cálculo do Valor Bruto**: $\text{valor\_bruto} = \lceil \text{minutos} / 15 \rceil \times 100$.
* **RN-008 — Teto Diário**: O valor cobrado é o menor entre o valor bruto e o teto: $\text{valor\_centavos} = \min(\text{valor\_bruto}, 8000)$.


* **RN-009 — Formato Monetário Inteiro**: Todos os valores financeiros são obrigatoriamente inteiros em centavos. Ponto flutuante é proibido.


* **RN-010 — Filtro Temporal para Relatório**: O relatório diário considera estritamente bilhetes encerrados cuja data de `saida` pertenca ao dia solicitado no fuso `-03:00`.


* **RN-011 — Arredondamento do Tempo Médio**: Média aritmética dos minutos de duração dos bilhetes encerrados no dia com arredondamento **0,5 para cima (*Half-Up*)**. Sem bilhetes encerrados no dia, retorna `0`.



---

## 3. Casos de Uso (UC) e Critérios de Aceite Mensuráveis

### UC1 — Abrir Bilhete

* **Rota**: `POST /bilhetes`

* **Request Body**: `{"placa": "ABC1D23", "entrada": "2026-10-05T10:00:00-03:00"}` (`entrada` opcional)


* **Resposta Esperada**: `201 Created`


```json
{
  "id": 1,
  "placa": "ABC1D23",
  "entrada": "2026-10-05T10:00:00-03:00",
  "status": "aberto"
}

```

* **Critérios de Aceite**:
1. Cria bilhete com status `aberto` e atribui ID numérico incremental.


2. Valida regex da placa (`RN-001`).


3. Impede abertura se a placa já tiver bilhete aberto (`RN-003`).





---

### UC2 — Encerrar Bilhete

* **Rota**: `POST /bilhetes/{id}/encerramento`

* **Resposta Esperada**: `200 OK`


```json
{
  "id": 1,
  "placa": "ABC1D23",
  "entrada": "2026-10-05T10:00:00-03:00",
  "saida": "2026-10-05T11:35:00-03:00",
  "minutos": 95,
  "valor_centavos": 700
}

```

* **Critérios de Aceite**:
1. Registra o instante de `saida` e calcula a duração total em minutos.


2. Atualiza o status do bilhete para `encerrado`.


3. Se minutos $\le 15$, `valor_centavos` = 0.


4. Se minutos = 95, calcula $\lceil 95 / 15 \rceil = 7$ frações $\rightarrow 7 \times 100 = 700$ centavos.


5. Se valor calculado $> 8000$, limita o retorno em `8000`.


6. Tentar encerrar bilhete inexistente retorna `404` (`bilhete_nao_encontrado`).


7. Tentar encerrar bilhete já encerrado/cancelado retorna `409` (`bilhete_ja_encerrado`).





---

### UC3 — Listar Bilhetes Ativos

* **Rota**: `GET /bilhetes/ativos`

* **Resposta Esperada**: `200 OK` com array dos bilhetes com `status: "aberto"`, ordenados por `entrada` decrescente (mais recentes primeiro).



---

### UC4 — Relatório Diário

* **Rota**: `GET /relatorios/diario?data=AAAA-MM-DD`

* **Resposta Esperada**: `200 OK`


```json
{
  "data": "2026-10-05",
  "total_bilhetes": 2,
  "faturamento_centavos": 1400,
  "tempo_medio_minutos": 95
}

```

* **Critérios de Aceite**:
1. Filtra bilhetes com status `encerrado` onde a data da `saida` no fuso `-03:00` seja igual ao parâmetro `data`.


2. Retorna a soma dos valores cobrados e a média de minutos com arredondamento *Half-Up*.


3. Formato de `data` inválido retorna `422` (`data_invalida`).





---

### UC5 — Cancelar Bilhete

* **Rota**: `POST /bilhetes/{id}/cancelamento`

* **Resposta Esperada**: `200 OK`


```json
{
  "id": 1,
  "placa": "ABC1D23",
  "entrada": "2026-10-05T10:00:00-03:00",
  "status": "cancelado"
}

```

* **Critérios de Aceite**:
1. Somente bilhetes em aberto podem ser cancelados.


2. Não gera cobrança e não preenche campos `saida`, `minutos` ou `valor_centavos`.


3. Liberar a placa para novas aberturas.


4. Tentar cancelar bilhete não aberto (encerrado ou já cancelado) retorna `409` (`bilhete_nao_aberto`).





---

### UC6 — Histórico por Placa

* **Rota**: `GET /bilhetes?placa=ABC1D23`

* **Resposta Esperada**: `200 OK` com array de todos os bilhetes registrados para a placa (todos os status), do mais recente para o mais antigo. Placa sem registros retorna array vazio `[]`.


---

### UC7 — Tolerância Gratuita

* **Regra Vinculada**: `RN-004`

* **Comportamento e Validação**:
1. Os primeiros `TOLERANCIA_MINUTOS` (15 minutos) do bilhete são isentos de cobrança.


2. Permanência $\le 15$ minutos resulta em `valor_centavos: 0` no encerramento (`UC2`).


3. Permanência $> 15$ minutos (ex.: 16 minutos) é cobrada **integralmente desde o 1º minuto** ($2 \text{ frações} \times 100 = 200 \text{ centavos}$), sem qualquer desconto dos 15 minutos iniciais.



---

### UC8 — Uma Vaga por Placa

* **Regra Vinculada**: `RN-003`

* **Comportamento e Validação**:
1. Uma placa só pode possuir **um único bilhete aberto** (`status: "aberto"`) por vez.


2. Tentativa de chamar `POST /bilhetes` (`UC1`) para uma placa com bilhete ativo retorna `HTTP 409 Conflict` com `{"erro": "bilhete_em_aberto"}`.


3. A placa volta a ficar elegível para novos bilhetes imediatamente após o encerramento (`UC2`) ou cancelamento (`UC5`) do bilhete ativo.



---
## 4. Tabela de Tratamento de Erros

| Erro / Condição | Status HTTP | Payload de Resposta |
| --- | --- | --- |
| Placa inválida ou ausente | `422` | `{"erro": "placa_invalida"}`<br> |
| Campo `entrada` fora do ISO-8601 / sem fuso | `422` | `{"erro": "entrada_invalida"}`<br> |
| Parâmetro `data` fora do formato `AAAA-MM-DD` | `422` | `{"erro": "data_invalida"}`<br> |
| ID de bilhete não encontrado no banco | `404` | `{"erro": "bilhete_nao_encontrado"}`<br> |
| Tentar encerrar bilhete que não está aberto | `409` | `{"erro": "bilhete_ja_encerrado"}`<br> |
| Tentar cancelar bilhete que não está aberto | `409` | `{"erro": "bilhete_nao_aberto"}`<br> |
| Tentar abrir bilhete para placa com bilhete ativo | `409` | `{"erro": "bilhete_em_aberto"}`<br> |