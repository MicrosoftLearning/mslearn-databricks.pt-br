---
lab:
  title: Proteger dados no catálogo do Unity
---

# Proteger dados no catálogo do Unity

A segurança de dados é uma preocupação crítica para organizações que gerenciam informações confidenciais em seu data lakehouse. À medida que as equipes de dados colaboram entre diferentes funções e departamentos, garantir que as pessoas certas tenham acesso aos dados certos, ao mesmo tempo em que se protege informações confidenciais contra acessos não autorizados, torna-se cada vez mais complexo.

Este laboratório prático demonstrará dois poderosos recursos de segurança do Catálogo do Unity que vão além do controle de acesso básico:

1. **Filtragem de linhas e mascaramento de colunas**: Saiba como proteger dados confidenciais no nível da tabela ocultando linhas específicas ou mascarando valores de coluna com base nas permissões do usuário
2. **Exibições dinâmicas**: Criar exibições inteligentes que ajustem automaticamente quais dados os usuários podem ver com base em suas associações a grupos e níveis de acesso

Este laboratório levará aproximadamente **45** minutos para ser concluído.

> **Observação**: a interface do usuário do Azure Databricks está sujeita a melhorias contínuas. A interface do usuário pode ter sido alterada desde que as instruções neste exercício foram escritas.

## Provisionar um workspace do Azure Databricks

> **Dica**: Se você já tem um workspace do Azure Databricks, pode ignorar esse procedimento e usar o workspace existente.

Este exercício inclui um script para provisionar um novo workspace do Azure Databricks. O script tenta criar um recurso de workspace do Azure Databricks de camada *Premium* em uma região na qual sua assinatura do Azure tenha cota suficiente para os núcleos de computação necessários para este exercício; e pressupõe que sua conta de usuário tenha permissões suficientes na assinatura para criar um recurso de workspace do Azure Databricks. 

Se o script falhar devido a cota ou permissões insuficientes, você pode tentar [criar um workspace do Azure Databricks interativamente no portal do Azure](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace).

1. Em um navegador da web, faça logon no [portal do Azure](https://portal.azure.com) em `https://portal.azure.com`.
2. Use o botão **[\>_]** à direita da barra de pesquisa na parte superior da página para criar um Cloud Shell no portal do Azure selecionando um ambiente do ***PowerShell***. O Cloud Shell fornece uma interface de linha de comando em um painel na parte inferior do portal do Azure, conforme mostrado aqui:

    ![Portal do Azure com um painel do Cloud Shell](./images/cloud-shell.png)

    > **Observação**: se você já criou um Cloud Shell que usa um ambiente *Bash*, alterne-o para o ***PowerShell***.

3. Você pode redimensionar o Cloud Shell arrastando a barra de separação na parte superior do painel ou usando os ícones **&#8212;**, **&#10530;** e **X** no canto superior direito do painel para minimizar, maximizar e fechar o painel. Para obter mais informações de como usar o Azure Cloud Shell, confira a [documentação do Azure Cloud Shell](https://docs.microsoft.com/azure/cloud-shell/overview).

4. No painel do PowerShell, insira os seguintes comandos para clonar esse repositório:

    ```
    rm -r mslearn-databricks -f
    git clone https://github.com/MicrosoftLearning/mslearn-databricks
    ```

5. Depois que o repositório tiver sido clonado, insira o seguinte comando para executar **setup.ps1** do script, que provisiona um workspace do Azure Databricks em uma região disponível:

    ```
    ./mslearn-databricks/setup.ps1
    ```

6. Se solicitado, escolha qual assinatura você deseja usar (isso só acontecerá se você tiver acesso a várias assinaturas do Azure).
7. Aguarde a conclusão do script – isso normalmente leva cerca de 5 minutos, mas em alguns casos pode levar mais tempo. Enquanto aguarda, revise o módulo de aprendizado [Implementar segurança e controle de acesso no Catálogo do Unity](https://learn.microsoft.com/training/modules/implement-security-unity-catalog) no Microsoft Learn.

## Criar um catálogo

1. Faça logon em um workspace vinculado ao metastore.
2. Selecione **Catálogo** no menu à esquerda.
3. Selecione **Catálogos** abaixo de **Acesso rápido**.
4. Selecione **Criar catálogo**.
5. Na caixa de diálogo **Criar um novo catálogo**, insira um **Nome do catálogo** e selecione o **Tipo** de catálogo que deseja criar: Catálogo **padrão**.
6. Especifique um **Local de armazenamento** gerenciado.

## Criar um notebook

1. Na barra lateral, use o link **(+) Novo** para criar um **Notebook**.
   
1. Altere o nome padrão do notebook (**Untitled Notebook *[date]***) para `Secure data in Unity Catalog` e, na lista suspensa **Conectar**, selecione o **Cluster sem servidor**, caso ainda não esteja selecionado.  Lembre-se de que **Sem servidor** está habilitado por padrão.

1. Copie e execute o código a seguir em uma nova célula em seu notebook para configurar seu ambiente de trabalho para este curso. Ele também definirá seu catálogo padrão para o seu catálogo específico e o esquema para o nome de esquema mostrado abaixo, usando as instruções `USE`.

```SQL
USE CATALOG `<your catalog>`;
USE SCHEMA `default`;
```

1. Execute o código abaixo e confirme que o catálogo atual está definido com o nome do seu catálogo exclusivo e que o esquema atual é **padrão**.

```sql
SELECT current_catalog(), current_schema()
```

## Proteger colunas e linhas com mascaramento de coluna e filtragem de linhas

### Criar a Tabela Customers

1. Execute o código abaixo para criar a tabela **customers** em seu esquema **padrão**.

```sql
CREATE OR REPLACE TABLE customers AS
SELECT *
FROM samples.tpch.customer;
```

2. Execute uma consulta para exibir *10* linhas da tabela **customers** em seu esquema **padrão**. Observe que a tabela contém informações como **c_name**, **c_phone** e **c_mktsegment**.
   
```sql
SELECT *
FROM customers  
LIMIT 10;
```

### Criar uma função para executar o mascaramento de colunas

Consulte a documentação [Filtrar dados sensíveis de tabela usando filtros de linha e máscaras de coluna](https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/filters-and-masks/) para obter ajuda adicional.

1. Crie uma função chamada **phone_mask** que rasure a coluna **c_phone** na tabela **customers** se o usuário não for membro do grupo ('admin'), usando a função `is_account_group_member`. A função **phone_mask** deverá retornar a cadeia de caracteres *REDACTED PHONE NUMBER* se o usuário não for um membro.

```sql
CREATE OR REPLACE FUNCTION phone_mask(c_phone STRING)
  RETURN CASE WHEN is_account_group_member('metastore_admins') 
    THEN c_phone 
    ELSE 'REDACTED PHONE NUMBER' 
  END;
```

2. Aplique a função de mascaramento de coluna **phone_mask** à tabela **customers** usando a instrução `ALTER TABLE`.

```sql
ALTER TABLE customers 
  ALTER COLUMN c_phone 
  SET MASK phone_mask;
```

3. Execute a consulta abaixo para exibir a tabela **customers** com a máscara de coluna aplicada. Confirme se a coluna **c_phone** exibe o valor *REDACTED PHONE NUMBER*.

```sql
SELECT *
FROM customers
LIMIT 10;
```

### Criar uma função para executar a filtragem de linhas

Consulte a documentação [Filtrar dados sensíveis de tabela usando filtros de linha e máscaras de coluna](https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/filters-and-masks/) para obter ajuda adicional.

1. Execute o código abaixo para contar o número total de linhas na tabela **customers**. Confirme se a tabela contém 750.000 linhas de dados.

```sql
SELECT count(*) AS TotalRows
FROM customers;
```

2. Crie uma função chamada **nation_filter** que filtre com base em **c_nationkey** na tabela **customers** se o usuário não for membro do grupo ('admin'), usando a função `is_account_group_member`. A função só deve retornar linhas em que **c_nationkey** seja igual a *21*.

    Consulte a documentação da [função if](https://learn.microsoft.com/en-us/azure/databricks/sql/language-manual/functions/if) para obter ajuda adicional.

```sql
CREATE OR REPLACE FUNCTION nation_filter(c_nationkey INT)
  RETURN IF(is_account_group_member('admin'), true, c_nationkey = 21);
```

3. Aplique a função `nation_filter` de filtragem de linhas à tabela **customers** usando a instrução `ALTER TABLE`.

```sql
ALTER TABLE customers 
SET ROW FILTER nation_filter ON (c_nationkey);
```

4. Execute a consulta abaixo para contar o número de linhas na tabela **customers** para você, já que as linhas foram filtradas para usuários que não são administradores. Confirme que você só consegue visualizar *29.859* linhas (*onde c_nationkey = 21*). 

```sql
SELECT count(*) AS TotalRows
FROM customers;
```

5. Execute a consulta abaixo para exibir a tabela de **customers**. 

    Confirme a tabela final:
    - oculta a coluna **c_phone** e
    - filtra as linhas com base na coluna **c_nationkey** para usuários que não são *administradores*.

```sql
SELECT *
FROM customers;
```

## Proteger colunas e linhas com exibições dinâmicas

### Criar a tabela Customers_new

1. Execute o código abaixo para criar a tabela **customers_new** em seu esquema **padrão**.

```sql
CREATE OR REPLACE TABLE customers_new AS
SELECT *
FROM samples.tpch.customer;
```

2. Execute uma consulta para exibir *10* linhas da tabela **customers_new** no esquema **padrão**. Observe que a tabela contém informações como **c_name**, **c_phone** e **c_mktsegment**.

```sql
SELECT *
FROM customers_new
LIMIT 10;
```

### Criar a Exibição dinâmica

Vamos criar uma exibição chamada **vw_customers** que apresenta uma visão processada dos dados da tabela **customers_new** com as seguintes transformações:

- Seleciona todas as colunas da tabela **customers_new**.

- Rasure todos os valores da coluna **c_phone** como *REDACTED PHONE NUMBER*, a menos que você esteja no grupo `is_account_group_member('admins')`
    - DICA: Use uma instrução `CASE WHEN` na cláusula `SELECT`.

- Restrinja as linhas em que **c_nationkey** seja igual a *21*, a menos que você esteja em `is_account_group_member('admins')`.
    - DICA: Use uma instrução `CASE WHEN` na cláusula `WHERE`.

```sql
-- Create a movies_gold view by redacting the "votes" column and restricting the movies with a rating below 6.0

CREATE OR REPLACE VIEW vw_customers AS
SELECT 
  c_custkey, 
  c_name, 
  c_address, 
  c_nationkey,
  CASE 
    WHEN is_account_group_member('admins') THEN c_phone
    ELSE 'REDACTED PHONE NUMBER'
  END as c_phone,
  c_acctbal, 
  c_mktsegment, 
  c_comment
FROM customers_new
WHERE
  CASE WHEN
    is_account_group_member('admins') THEN TRUE
    ELSE c_nationkey = 21
  END;
```

3. Exiba os dados na exibição **vw_customers**. Confirme se a coluna **c_phone** foi rasurada. Confirme se **c_nationkey** é igual a *21*, a menos que você seja o administrador.

```sql
SELECT * 
FROM vw_customers;
```

6. Conte o número de linhas na exibição **vw_customers**. Confirme se a exibição contém *29.859* linhas.

```sql
SELECT count(*)
FROM vw_customers;
```

### Realizar a concessão de acesso à exibição

1. Vamos conceder permissão para que os "usuários da conta" visualizem a exibição **vw_customers**.

**Observação:** Você também precisará fornecer aos usuários acesso ao catálogo e ao esquema. Nesse ambiente de treinamento compartilhado, não é possível conceder acesso ao catálogo a outros usuários.

```sql
GRANT SELECT ON VIEW vw_customers TO `account users`
```

2. Use a instrução `SHOW` para exibir todos os privilégios (herdados, negados e concedidos) que afetam a exibição **vw_customers**. Confirme se a coluna **Principal** contém os *usuários da conta*.

    Consulte a documentação de [SHOW GRANTS](https://learn.microsoft.com/en-us/azure/databricks/sql/language-manual/security-show-grant) para obter ajuda.

```sql
SHOW GRANTS ON vw_customers
```