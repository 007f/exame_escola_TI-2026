# tests.md — Plano Exaustivo de Testes (TDD)

Este documento estabelece a suíte de testes automatizados (unitários e de integração) exigidos para a validação das regras de negócio, contratos de API e casos de borda do sistema de estacionamento rotativo[cite: 1].

---

## 🧪 Tabela Centralizada de Cenários de Teste

| ID | Caso de Uso / Regra | Contexto | Cenário / Entrada (Input) | Comportamento Esperado (Assert / Payload) | Tipo |
| --- | --- | --- | --- | --- | --- |
| **T01** | UC1 / RN-001 | `POST /bilhetes` | Body: `{"placa": "ABC1D23"}` | `HTTP 201 Created` — Retorna ID numérico, status `"aberto"` e data `entrada` atual no fuso `-03:00`. | Feliz |
| **T02** | UC1 / RN-002 | `POST /bilhetes` | Body: `{"placa": "ABC1D23", "entrada": "2026-10-05T10:00:00-03:00"}` | `HTTP 201 Created` — Registra exatamente o instante de entrada fornecido. | Feliz |
| **T03** | UC1 / RN-001 | `POST /bilhetes` | Body: `{"placa": "ABC1D2"}` (6 caracteres) | `HTTP 422 Unprocessable Entity` — Body: `{"erro": "placa_invalida"}`. | Borda |
| **T04** | UC1 / RN-001 | `POST /bilhetes` | Body: `{"placa": "ABC1D234"}` (8 caracteres) | `HTTP 422 Unprocessable Entity` — Body: `{"erro": "placa_invalida"}`. | Borda |
| **T05** | UC1 / RN-001 | `POST /bilhetes` | Body: `{"placa": "abc1d23"}` (letras minúsculas) | `HTTP 422 Unprocessable Entity` — Body: `{"erro": "placa_invalida"}`. | Borda |
| **T06** | UC1 / RN-001 | `POST /bilhetes` | Body: `{"placa": "ABC-1234"}` (caractere especial) | `HTTP 422 Unprocessable Entity` — Body: `{"erro": "placa_invalida"}`. | Borda |
| **T07** | UC1 / RN-001 | `POST /bilhetes` | Body: `{}` (campo `placa` ausente ou `null`) | `HTTP 422 Unprocessable Entity` — Body: `{"erro": "placa_invalida"}`. | Borda |
| **T08** | UC1 / RN-002 | `POST /bilhetes` | Body: `{"placa": "ABC1D23", "entrada": "2026-10-05T10:00:00"}` (sem fuso) | `HTTP 422 Unprocessable Entity` — Body: `{"erro": "entrada_invalida"}`. | Borda |
| **T09** | UC1 / RN-002 | `POST /bilhetes` | Body: `{"placa": "ABC1D23", "entrada": "05/10/2026 10:00"}` (formato inválido) | `HTTP 422 Unprocessable Entity` — Body: `{"erro": "entrada_invalida"}`. | Borda |
| **T10** | UC1 / RN-003, UC8 | `POST /bilhetes` | Abertura repetida para placa `"ABC1D23"` que já possui bilhete com status `"aberto"`. | `HTTP 409 Conflict` — Body: `{"erro": "bilhete_em_aberto"}`. | Borda |
| **T11** | UC2 / RN-004, UC7 | `POST /bilhetes/{id}/encerramento` | Entrada: `10:00:00`, Saída: `10:15:00` (permanência de exatamente 15 min). | `HTTP 200 OK` — `minutos: 15`, `valor_centavos: 0` (isento pela tolerância). | Borda |
| **T12** | UC2 / RN-004, UC7 | `POST /bilhetes/{id}/encerramento` | Entrada: `10:00:00`, Saída: `10:16:00` (permanência de 16 min). | `HTTP 200 OK` — `minutos: 16`, `valor_centavos: 200` (2 frações de 15 min, cobrança integral sem dedução). | Borda |
| **T13** | UC2 / RN-005, RN-006, RN-007 | `POST /bilhetes/{id}/encerramento` | Entrada: `10:00:00`, Saída: `11:35:00` (permanência de 95 min). | `HTTP 200 OK` — `minutos: 95`, `valor_centavos: 700` ($\lceil 95 / 15 \rceil = 7 \text{ frações} \times 100$). | Feliz |
| **T14** | UC2 / RN-008 | `POST /bilhetes/{id}/encerramento` | Entrada: `2026-10-05T00:00:00-03:00`, Saída: `2026-10-05T23:59:00-03:00` (permanência de 1439 min). | `HTTP 200 OK` — `minutos: 1439`, `valor_centavos: 8000` (limitado ao teto diário de 8000 centavos). | Borda |
| **T15** | UC2 | `POST /bilhetes/9999/encerramento` | ID inexistente no banco H2. | `HTTP 404 Not Found` — Body: `{"erro": "bilhete_nao_encontrado"}`. | Borda |
| **T16** | UC2 | `POST /bilhetes/{id}/encerramento` | Tentar encerrar bilhete que já possui status `"encerrado"`. | `HTTP 409 Conflict` — Body: `{"erro": "bilhete_ja_encerrado"}`. | Borda |
| **T17** | UC2 | `POST /bilhetes/{id}/encerramento` | Tentar encerrar bilhete com status `"cancelado"`. | `HTTP 409 Conflict` — Body: `{"erro": "bilhete_ja_encerrado"}`. | Borda |
| **T18** | UC2 / RN-002 | `POST /bilhetes/{id}/encerramento` | Body: `{"saida": "2026-10-05T09:00:00-03:00"}` onde a entrada foi às `10:00:00` (saída anterior à entrada). | `HTTP 422 Unprocessable Entity` — Body: `{"erro": "saida_invalida"}`. | Borda |
| **T19** | UC3 | `GET /bilhetes/ativos` | Existem 2 bilhetes abertos (`entrada` às `10:00` e `11:00`). | `HTTP 200 OK` — Array com 2 itens ordenados do mais recente para o mais antigo (ordenado por `entrada` desc). | Feliz |
| **T20** | UC3 | `GET /bilhetes/ativos` | Nenhum bilhete em aberto no banco. | `HTTP 200 OK` — Body: `[]`. | Borda |
| **T21** | UC4 / RN-010, RN-011 | `GET /relatorios/diario?data=2026-10-05` | 2 bilhetes encerrados na data com durações 90 min (600 cent) e 100 min (700 cent). Média $= 95,0$. | `HTTP 200 OK` — `{"data": "2026-10-05", "total_bilhetes": 2, "faturamento_centavos": 1300, "tempo_medio_minutos": 95}`. | Feliz |
| **T22** | UC4 / RN-011 | `GET /relatorios/diario?data=2026-10-05` | 2 bilhetes encerrados com durações 30 min e 31 min (soma = 61 min, média $= 30,5$). | `HTTP 200 OK` — `tempo_medio_minutos: 31` (arredondamento *Half-Up* do $30,5$). | Borda |
| **T23** | UC4 / RN-010, RN-011 | `GET /relatorios/diario?data=2026-10-05` | Nenhum bilhete encerrado no dia solicitado. | `HTTP 200 OK` — `{"data": "2026-10-05", "total_bilhetes": 0, "faturamento_centavos": 0, "tempo_medio_minutos": 0}`. | Borda |
| **T24** | UC4 | `GET /relatorios/diario?data=05-10-2026` | Parâmetro `data` fora do formato `AAAA-MM-DD`. | `HTTP 422 Unprocessable Entity` — Body: `{"erro": "data_invalida"}`. | Borda |
| **T25** | UC5 / RN-003, UC8 | `POST /bilhetes/{id}/cancelamento` | Bilhete com status `"aberto"`. | `HTTP 200 OK` — Status atualizado para `"cancelado"`. Placa liberada para novos bilhetes. | Feliz |
| **T26** | UC5 | `POST /bilhetes/{id}/cancelamento` | Tentar cancelar bilhete com status `"encerrado"`. | `HTTP 409 Conflict` — Body: `{"erro": "bilhete_nao_aberto"}`. | Borda |
| **T27** | UC5 | `POST /bilhetes/9999/cancelamento` | ID inexistente no banco H2. | `HTTP 404 Not Found` — Body: `{"erro": "bilhete_nao_encontrado"}`. | Borda |
| **T28** | UC6 | `GET /bilhetes?placa=ABC1D23` | Placa com 3 bilhetes em históricos variados (aberto, encerrado, cancelado). | `HTTP 200 OK` — Array contendo os 3 bilhetes ordenados da `entrada` mais recente para a mais antiga. | Feliz |
| **T29** | UC6 | `GET /bilhetes?placa=XYZ9W99` | Placa sem nenhum registro prévio no sistema. | `HTTP 200 OK` — Body: `[]`. | Borda |

---

## 🎯 Resumo da Cobertura de Testes
* **Total de Cenários Mapeados:** 29 testes automatizados.
* **Cenários do Caminho Feliz (*Happy Path*):** 7 testes (`T01`, `T02`, `T13`, `T19`, `T21`, `T25`, `T28`).
* **Cenários de Borda e Erro (*Edge Cases / Limits*):** 22 testes (garantindo cumprimento rigoroso do requisito da constituição de no mínimo 2 a 3 casos de borda por regra de negócio)[cite: 1].