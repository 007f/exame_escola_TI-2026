# constitution.md — Diretrizes e Regras Imutáveis

## 1. Idioma e Convenções
* **Código-fonte:** Classes, métodos, variáveis, DTOs e entidades DEVEM ser escritos obrigatoriamente em **inglês**.
* **Contrato Público da API:** Endpoints (no plural), URLs, parâmetros de query e chaves JSON DEVEM seguir estritamente o contrato em **português** e **`snake_case`** (ex.: `/bilhetes`, `/relatorios`, `placa_veiculo`).
* **Mensagens de Erro:** É **estritamente proibido** utilizar o formato padrão de erro do Spring Boot (`/error` com `timestamp`, `path`, `status`, `message`, etc.). Todas as respostas de erro DEVEM conter exclusivamente o objeto JSON exato com a chave `"erro"` (ex.: `{"erro": "placa_invalida"}`).

## 2. Stack Tecnológica e Infraestrutura
* **Linguagem & Framework:** Java com Spring Boot.
* **Persistência de Dados:** Banco de dados relacional em memória **H2 Database**. É proibido o uso de dependências ou serviços de bancos externos instalados.
* **Porta do Servidor & Base URL:** A porta base padrão DEVE ser a `8001` (Base URL: `http://localhost:8001`), sendo configurável dinamicamente através da propriedade/variável de ambiente `PORTA_SERVICO`.

## 3. Diretrizes Financeiras e de Tipagem
* **Representação Monetária:** Valores financeiros DEVEM ser representados estritamente como números inteiros em centavos (`int` ou `long`). É **estritamente proibido** o uso de tipos de ponto flutuante (`float` ou `double`) para tratar dinheiro.
* **Regras de Arredondamento:** 
  * Cálculos de faturamento e frações financeiras devem ser arredondados para cima.
  * Média estatística em relatórios deve aplicar o arredondamento de 0,5 para cima (*Half-Up*).

## 4. Padrões de Resposta e Tratamento de Erros
* **Formatos de Data/Hora:** Todas as datas e instantes de tempo DEVEM ser formatados no padrão ISO-8601 respeitando o fuso horário `-03:00`.
* **Tratamento Global de Exceções:** Qualquer exceção de negócio ou erro de validação DEVE ser capturado globalmente e devolvido com o status HTTP adequado acompanhado do corpo estrito `{"erro": "codigo_do_erro"}`.
* **Restrição de Escopo:** O escopo limita-se EXCLUSIVAMENTE à API REST. Interfaces gráficas e módulos de back-office são proibidos.

## 5. Qualidade e Testes (TDD)
* **Casos de Borda Obrigatórios:** Toda regra de negócio (tolerância, tempo fracionado, limites financeiros, concorrência de placa) DEVE possuir obrigatoriamente pelo menos um teste automatizado cobrindo os seus cenários de borda (limites inferiores e superiores).