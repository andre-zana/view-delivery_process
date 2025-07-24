# view-delivery_process
  A 'delivery_process' é uma view MySQL que consolida e formata dados de pedidos da tabela 'orders', focando no processo de entrega. Ela transforma códigos internos em descrições legíveis e calcula métricas importantes para acompanhamento logístico.


# Análise da Query SQL

## 1. Comando Principal: `ALTER VIEW delivery_process as`

Este comando é utilizado para modificar uma view existente no banco de dados. Uma `VIEW` é uma tabela virtual baseada no resultado de uma instrução `SELECT`. Ela não armazena dados fisicamente, mas atua como uma janela para os dados de uma ou mais tabelas subjacentes. 

Ao usar `ALTER VIEW`, estamos redefinindo a estrutura ou a lógica da view `delivery_process`. Se a view não existisse, um erro ocorreria na maioria dos SGBDs, a menos que a sintaxe específica do SGBD permita a criação implícita.

## 2. Cláusula `SELECT`

A cláusula `SELECT` é o coração da view, definindo quais dados serão recuperados e como serão apresentados. Cada linha na `SELECT` define uma coluna na view `delivery_process`.

### Colunas e Transformações:

- `number AS Pedido`:
  - **O que faz:** Seleciona a coluna `number` da tabela `orders` e a renomeia para `Pedido` na view. Provavelmente, `number` é o identificador único de um pedido.
  - **Tipo de dado esperado:** Numérico ou string, dependendo do formato do número do pedido.

- `invoice_number AS Nota_Fiscal`:
  - **O que faz:** Seleciona a coluna `invoice_number` e a renomeia para `Nota_Fiscal`.
  - **Tipo de dado esperado:** Numérico ou string, representando o número da nota fiscal.

- **Tradução de Códigos com `CASE` (Canal de Venda, Status e CD):**
  - `CASE channel_name ... END AS Canal_Venda`
  - `CASE status ... END AS status`
  - `CASE status WHEN 'extrema' THEN 'Extrema' WHEN 'sbc' THEN 'SBC' ELSE location_id END AS CD`
  - **O que fazem:** Estas expressões `CASE` são utilizadas para traduzir códigos internos (como nomes de canais de venda, status de pedidos e IDs de localização/CD) para descrições mais amigáveis e legíveis em português. Se o valor original não corresponder a nenhuma das opções listadas, ele será mantido.
  - **Tipo de dado esperado:** String.
  - **Exemplos de tradução:**
    - `channel_name`: 'nuvemshop' vira 'Ecommerce', 'MERCADO_LIVRE' vira 'Mercado Livre'.
    - `status`: 'pending' vira 'Pagamento Pendente', 'approved' vira 'Pagamento Aprovado'.
    - `location_id`: 'extrema' vira 'Extrema', 'sbc' vira 'SBC'.

- **Formatação de Datas com `DATE_FORMAT` (Múltiplas Colunas):**
  - `DATE_FORMAT(purchased_at, '%d/%m/%Y %H:%i') AS Data_Compra`
  - `DATE_FORMAT(approved_at, '%d/%m/%Y %H:%i') AS Data_Pagamento`
  - `DATE_FORMAT(shipping_limit_date, '%d/%m/%Y %H:%i') AS Data_Limite_Envio`
  - `DATE_FORMAT(delivery_limit_date, '%d/%m/%Y %H:%i') AS Data_Limite_Entrega`
  - `DATE_FORMAT(delivered_at, '%d/%m/%Y %H:%i') AS Data_Entrega`
  - `DATE_FORMAT(canceled_at, '%d/%m/%Y %H:%i') AS Data_Cancelado`
  - **O que fazem:** Todas estas linhas utilizam a função `DATE_FORMAT` para converter colunas de data e hora (como `purchased_at`, `approved_at`, etc.) para um formato de string padronizado e legível: 'dia/mês/ano hora:minuto'.
  - **Tipo de dado esperado:** Data/Hora na origem, String formatada na view.
  - **Observação:** A função `DATE_FORMAT` é específica do MySQL. Para outros SGBDs, seriam necessárias funções equivalentes (e.g., `TO_CHAR` no PostgreSQL, `FORMAT` ou `CONVERT` no SQL Server).

- `CASE WHEN delivered_at IS NOT NULL THEN DATEDIFF(delivered_at, delivery_limit_date) ELSE DATEDIFF(CURDATE(), delivery_limit_date) END AS Dias_Atraso_Entrega`:
  - **O que faz:** Calcula o número de dias de atraso na entrega.
    - Se o pedido já foi entregue (`delivered_at IS NOT NULL`), calcula a diferença em dias entre a data de entrega (`delivered_at`) e a data limite de entrega (`delivery_limit_date`).
    - Se o pedido ainda não foi entregue, calcula a diferença em dias entre a data atual (`CURDATE()`) e a data limite de entrega.
  - **Tipo de dado esperado:** Inteiro.
  - **Observação:** `DATEDIFF` e `CURDATE()` são funções específicas do MySQL. Em PostgreSQL, `(delivered_at::date - delivery_limit_date::date)` ou `(CURRENT_DATE - delivery_limit_date::date)` seriam usados. Em SQL Server, `DATEDIFF(day, delivery_limit_date, delivered_at)` ou `DATEDIFF(day, delivery_limit_date, GETDATE())`.

- `CASE WHEN delivered_at IS NOT NULL THEN CASE WHEN date(delivered_at) <= date(delivery_limit_date) THEN 'Dentro do Prazo' ELSE 'Fora do Prazo' END ELSE CASE WHEN CURDATE() <= date(delivery_limit_date) THEN 'Dentro do Prazo' ELSE 'Fora do Prazo' END END AS Situacao`:
  - **O que faz:** Determina a situação da entrega (Dentro do Prazo ou Fora do Prazo).
    - Se o pedido foi entregue: Compara a data de entrega (apenas a data, sem a hora, usando `date()`) com a data limite de entrega. Se a data de entrega for menor ou igual à data limite, é 'Dentro do Prazo', caso contrário, 'Fora do Prazo'.
    - Se o pedido ainda não foi entregue: Compara a data atual (`CURDATE()`) com a data limite de entrega. Se a data atual for menor ou igual à data limite, é 'Dentro do Prazo', caso contrário, 'Fora do Prazo'.
  - **Tipo de dado esperado:** String.
  - **Observação:** `date()` e `CURDATE()` são funções MySQL. Equivalentes em PostgreSQL seriam `delivered_at::date` e `CURRENT_DATE`. Em SQL Server, `CAST(delivered_at AS DATE)` e `GETDATE()`.

- `carrier_name AS Transportador`:
  - **O que faz:** Seleciona a coluna `carrier_name` e a renomeia para `Transportador`.
  - **Tipo de dado esperado:** String.

- `0 OcorrenciaDevolucao`:
  - **O que faz:** Cria uma coluna chamada `OcorrenciaDevolucao` com o valor fixo `0`. Isso pode ser um placeholder para futuras funcionalidades ou para garantir que a view tenha uma estrutura consistente com outras views/tabelas.
  - **Tipo de dado esperado:** Numérico (inteiro).

## 3. Cláusula `FROM`

- `FROM orders`:
  - **O que faz:** Indica que todos os dados selecionados vêm da tabela `orders`.

## 4. Cláusula `WHERE`

- `WHERE YEAR(purchased_at) = '2025'`:
  - **O que faz:** Filtra os resultados para incluir apenas os pedidos cuja data de compra (`purchased_at`) seja do ano de 2025. A função `YEAR()` extrai o ano de uma data.
  - **Observação:** `YEAR()` é uma função MySQL. Em PostgreSQL, seria `EXTRACT(YEAR FROM purchased_at)`. Em SQL Server, `YEAR(purchased_at)` também funciona.

## 5. O que a Query Faz no Geral?

Esta query cria (ou recria) uma view chamada `delivery_process` que consolida e formata informações sobre pedidos, focando no processo de entrega. Ela traduz códigos internos para descrições amigáveis, formata datas para melhor legibilidade, calcula dias de atraso na entrega e classifica a situação da entrega (dentro ou fora do prazo). O resultado é uma visão clara e processada dos dados de pedidos, útil para relatórios e análises de logística. O filtro `WHERE YEAR(purchased_at) = '2025'` indica que esta view é específica para pedidos feitos no ano de 2025.

## 6. É SQL? MySQL? PostgreSQL e etc?

Sim, é SQL. Mais especificamente, esta query utiliza sintaxe e funções que são **altamente compatíveis com MySQL**.

### Compatibilidade:

- **MySQL:** Totalmente compatível. Todas as funções (`DATE_FORMAT`, `DATEDIFF`, `CURDATE()`, `YEAR()`) e a sintaxe `ALTER VIEW` são padrão no MySQL.

- **PostgreSQL:** Não é diretamente compatível sem modificações. As funções de data e hora (`DATE_FORMAT`, `DATEDIFF`, `CURDATE()`, `YEAR()`) precisariam ser substituídas por suas equivalentes no PostgreSQL (`TO_CHAR`, operadores de subtração de data, `CURRENT_DATE`, `EXTRACT(YEAR FROM ...)`).

- **SQL Server:** Não é diretamente compatível sem modificações. As funções de data e hora (`DATE_FORMAT`, `DATEDIFF`, `CURDATE()`, `YEAR()`) precisariam ser substituídas por suas equivalentes no SQL Server (`FORMAT` ou `CONVERT`, `DATEDIFF`, `GETDATE()`, `YEAR()`). A sintaxe `ALTER VIEW` é similar, mas as funções internas diferem.

- **Oracle:** Não é diretamente compatível. As funções de data e hora seriam diferentes (e.g., `TO_CHAR`, operadores de subtração de data, `SYSDATE`, `TO_CHAR(..., 'YYYY')`).

**Conclusão:** A query é escrita em SQL, mas com um dialeto **específico do MySQL** devido ao uso de funções como `DATE_FORMAT`, `DATEDIFF`, `CURDATE()` e `YEAR()`. Para ser executada em outros SGBDs (PostgreSQL, SQL Server, Oracle, etc.), as funções de data e hora precisariam ser adaptadas para a sintaxe e funções nativas de cada banco de dados. A estrutura básica `ALTER VIEW ... SELECT ... FROM ... WHERE` é padrão SQL e amplamente suportada, mas as funções internas a tornam dependente do MySQL.
