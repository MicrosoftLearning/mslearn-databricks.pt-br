---
lab:
  title: Transformar dados com o Apache Spark no Azure Databricks
---

# Transformar dados com o Apache Spark no Azure Databricks

O Azure Databricks é uma versão baseada no Microsoft Azure da popular plataforma de código aberto Databricks. 

O Azure Databricks foi criado no Apache Spark e oferece uma solução altamente escalonável para tarefas de engenharia e análise de dados que envolvem o trabalho com dados em arquivos.

As tarefas comuns de transformação de dados no Azure Databricks incluem limpeza de dados, execução de agregações e conversão de tipos. Essas transformações são essenciais para preparar dados para análise e fazem parte do processo maior de ETL (extrair, transformar, carregar).

Este exercício deve levar aproximadamente **30** minutos para ser concluído.

## Provisionar um workspace do Azure Databricks

> **Dica**: Se você já tem um workspace do Azure Databricks, pode ignorar esse procedimento e usar o workspace existente.

Este exercício inclui um script para provisionar um novo workspace do Azure Databricks. O script tenta criar um recurso de workspace do Azure Databricks de camada *Premium* em uma região na qual sua assinatura do Azure tenha cota suficiente para os núcleos de computação necessários para este exercício; e pressupõe que sua conta de usuário tenha permissões suficientes na assinatura para criar um recurso de workspace do Azure Databricks. Se o script falhar devido a cota ou permissões insuficientes, você pode tentar [criar um workspace do Azure Databricks interativamente no portal do Azure](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace).

1. Em um navegador da web, faça logon no [portal do Azure](https://portal.azure.com) em `https://portal.azure.com`.
2. Use o botão **[\>_]** à direita da barra de pesquisa na parte superior da página para criar um Cloud Shell no portal do Azure, selecionando um ambiente ***PowerShell*** e criando um armazenamento caso solicitado. O Cloud Shell fornece uma interface de linha de comando em um painel na parte inferior do portal do Azure, conforme mostrado aqui:

    ![Portal do Azure com um painel do Cloud Shell](./images/cloud-shell.png)

    > **Observação**: Se você tiver criado anteriormente um shell de nuvem que usa um ambiente *Bash*, use o menu suspenso no canto superior esquerdo do painel do shell de nuvem para alterá-lo para ***PowerShell***.

3. Observe que você pode redimensionar o Cloud Shell arrastando a barra do separador na parte superior do painel ou usando os ícones **&#8212;** , **&#9723;** e **X** no canto superior direito do painel para minimizar, maximizar e fechar o painel. Para obter mais informações de como usar o Azure Cloud Shell, confira a [documentação do Azure Cloud Shell](https://docs.microsoft.com/azure/cloud-shell/overview).

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
7. Aguarde a conclusão do script – isso normalmente leva cerca de 5 minutos, mas em alguns casos pode levar mais tempo. Enquanto espera, revise o artigo [Análise exploratória de dados no Azure Databricks na documentação do Azure Databricks](https://learn.microsoft.com/azure/databricks/exploratory-data-analysis/).

## Criar um cluster

O Azure Databricks é uma plataforma de processamento distribuído que usa *clusters* do Apache Spark para processar dados em paralelo em vários nós. Cada cluster consiste em um nó de driver para coordenar o trabalho e nós de trabalho para executar tarefas de processamento. Neste exercício, você criará um cluster de *nó único* para minimizar os recursos de computação usados no ambiente de laboratório (no qual os recursos podem ser restritos). Em um ambiente de produção, você normalmente criaria um cluster com vários nós de trabalho.

> **Dica**: Se você já tiver um cluster com uma versão 13.3 LTS de runtime ou superior em seu workspace do Azure Databricks, poderá usá-lo para concluir este exercício e ignorar este procedimento.

1. No portal do Azure, navegue até o grupo de recursos **msl-*xxxxxxx*** criado pelo script (ou o grupo de recursos que contém seu workspace do Azure Databricks)
2. Selecione o recurso Serviço do Azure Databricks (chamado **databricks-*xxxxxxx*** se você usou o script de instalação para criá-lo).
3. Na página **Visão geral** do seu workspace, use o botão **Iniciar workspace** para abrir seu workspace do Azure Databricks em uma nova guia do navegador, fazendo o logon se solicitado.

    > **Dica**: ao usar o portal do workspace do Databricks, várias dicas e notificações podem ser exibidas. Dispense-as e siga as instruções fornecidas para concluir as tarefas neste exercício.

4. Na barra lateral à esquerda, selecione a tarefa **(+) Novo** e, em seguida, selecione **Cluster**.
5. Na página **Novo Cluster**, crie um novo cluster com as seguintes configurações:
    - **Nome do cluster**: cluster *Nome do Usuário* (o nome do cluster padrão)
    - **Política**: Sem restrições
    - **Modo de cluster**: Nó Único
    - **Modo de acesso**: Usuário único (*com sua conta de usuário selecionada*)
    - **Versão do runtime do Databricks**: 13.3 LTS (Spark 3.4.1, Scala 2.12) ou posterior
    - **Usar Aceleração do Photon**: Selecionado
    - **Tipo de nó**: Standard_DS3_v2
    - **Encerra após** *20* **minutos de inatividade**

6. Aguarde a criação do cluster. Isso pode levar alguns minutos.

> **Observação**: se o cluster não for iniciado, sua assinatura pode ter cota insuficiente na região onde seu workspace do Azure Databricks está provisionado. Consulte [Limite de núcleo da CPU impede a criação do cluster](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit) para obter detalhes. Se isso acontecer, você pode tentar excluir seu workspace e criar um novo workspace em uma região diferente. Você pode especificar uma região como um parâmetro para o script de instalação da seguinte maneira: `./mslearn-databricks/setup.ps1 eastus`

## Criar um notebook

1. Na barra lateral, use o link **(+) Novo** para criar um **Notebook**.

2. Altere o nome padrão do notebook (**Notebook Sem Título *[data]***) para `Transform data with Spark` e, na lista suspensa **Conectar**, selecione o cluster, caso ainda não esteja selecionado. Se o cluster não executar, é porque ele pode levar cerca de um minuto para iniciar.

## Ingerir dados

1. Na primeira célula do notebook, insira o código a seguir, que usa os comandos de *shell* para baixar os arquivos de dados do GitHub para o sistema de arquivos usado pelo cluster.

     ```python
    %sh
    rm -r /dbfs/spark_lab
    mkdir /dbfs/spark_lab
    wget -O /dbfs/spark_lab/2019.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/2019_edited.csv
    wget -O /dbfs/spark_lab/2020.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/2020_edited.csv
    wget -O /dbfs/spark_lab/2021.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/2021_edited.csv
     ```

2. Use a opção de menu **&#9656; Executar Célula** à esquerda da célula para executá-la. Em seguida, aguarde o término do trabalho do Spark executado pelo código.
3. Adicione uma nova célula de código e use-a para executar o seguinte código, que define um esquema para os dados:

    ```python
   from pyspark.sql.types import *
   from pyspark.sql.functions import *
   orderSchema = StructType([
        StructField("SalesOrderNumber", StringType()),
        StructField("SalesOrderLineNumber", IntegerType()),
        StructField("OrderDate", DateType()),
        StructField("CustomerName", StringType()),
        StructField("Email", StringType()),
        StructField("Item", StringType()),
        StructField("Quantity", IntegerType()),
        StructField("UnitPrice", FloatType()),
        StructField("Tax", FloatType())
   ])
   df = spark.read.load('/spark_lab/*.csv', format='csv', schema=orderSchema)
   display(df.limit(100))
    ```

## Limpar os dados

Observe que esse conjunto de dados tem algumas linhas e valores `null` duplicados na coluna **Imposto**. Portanto, uma etapa de limpeza é necessária antes que qualquer processamento e análise adicionais sejam feitos com os dados.

![Tabela com dados a limpar.](./images/data-cleaning.png)

1. Na célula de código existente, use o ícone **+** para adicionar uma nova célula de código. Em seguida, na nova célula, insira e execute o seguinte código para remover linhas duplicadas da tabela e substituir as entradas `null` pelos valores corretos:

    ```python
    from pyspark.sql.functions import col
    df = df.dropDuplicates()
    df = df.withColumn('Tax', col('UnitPrice') * 0.08)
    df = df.withColumn('Tax', col('Tax').cast("float"))
    ```

Observe que, após atualizar os valores na coluna **Imposto**, o tipo de dados será definido como `float` novamente. Isso ocorre porque seu tipo de dados muda para `double` depois que o cálculo é feito. Como `double` tem um uso de memória maior do que `float`, é melhor para o desempenho converter a coluna de volta para `float`.

## Filtrar um dataframe

1. Adicione uma nova célula de código e use-a para executar o seguinte código, que irá:
    - Filtrar as colunas do dataframe de pedidos de vendas para incluir apenas o nome do cliente e o endereço de email.
    - Contar o número total de registros de pedidos
    - Contar o número de clientes distintos
    - Exibir os clientes distintos

    ```python
   customers = df['CustomerName', 'Email']
   print(customers.count())
   print(customers.distinct().count())
   display(customers.distinct())
    ```

    Observe os seguintes detalhes:

    - Quando você executa uma operação em um dataframe, o resultado é um novo dataframe (nesse caso, um dataframe customers é criado pela seleção de um subconjunto específico de colunas do dataframe df)
    - Os dataframes fornecem funções como count e distinct que podem ser usadas para resumir e filtrar os dados que eles contêm.
    - A sintaxe `dataframe['Field1', 'Field2', ...]` é uma forma abreviada de definir um subconjunto de colunas. Você também pode usar o método **select**, para que a primeira linha do código acima possa ser escrita como `customers = df.select("CustomerName", "Email")`

1. Agora, vamos aplicar um filtro para incluir apenas os clientes que fizeram um pedido de um produto específico executando o seguinte código em uma nova célula de código:

    ```python
   customers = df.select("CustomerName", "Email").where(df['Item']=='Road-250 Red, 52')
   print(customers.count())
   print(customers.distinct().count())
   display(customers.distinct())
    ```

    Observe que você pode "encadear" várias funções para que a saída de uma função se torne a entrada da próxima. Nesse caso, o dataframe criado pelo método “select” é o dataframe de origem do método “where” usado para aplicar os critérios de filtragem.

## Agregar e agrupar dados em um dataframe

1. Execute o seguinte código em uma nova célula de código para agregar e agrupar os dados do pedido:

    ```python
   productSales = df.select("Item", "Quantity").groupBy("Item").sum()
   display(productSales)
    ```

    Observe que os resultados mostram a soma das quantidades de pedido agrupadas por produto. O método **groupBy** agrupa as linhas por *Item*, e a função de agregação de **soma** seguinte é aplicada a todas as colunas numéricas restantes (nesse caso, *Quantidade*)

1. Em uma nova célula de código, vamos tentar outra agregação:

    ```python
   yearlySales = df.select(year("OrderDate").alias("Year")).groupBy("Year").count().orderBy("Year")
   display(yearlySales)
    ```

    Desta vez, os resultados mostram o número de pedidos de vendas por ano. Observe que o método de seleção inclui uma função SQL **year** para extrair o componente de ano do campo *OrderDate* e, em seguida, um método **alias** é usado para atribuir um nome de coluna ao valor de ano extraído. Em seguida, os dados são agrupados pela coluna *Year* derivada, e a **contagem** de linhas em cada grupo é calculada antes de finalmente o método **orderBy** ser usado para classificar o dataframe resultante.

> **Observação**: Para saber mais sobre como trabalhar com Dataframes no Azure Databricks, consulte [Introdução a DataFrames – Python](https://docs.microsoft.com/azure/databricks/spark/latest/dataframes-datasets/introduction-to-dataframes-python) na documentação do Azure Databricks.

## Executar um código SQL em uma célula

1. Embora seja útil a inserção de instruções SQL em uma célula que contém código PySpark, os analistas de dados muitas vezes preferem trabalhar diretamente com SQL. Adicione uma nova célula de código e use-a para executar o código a seguir.

    ```python
   df.createOrReplaceTempView("salesorders")
    ```

Essa linha de código criará uma exibição temporária que poderá ser usada diretamente com instruções SQL.

2. Execute o código a seguir em uma nova célula:
   
    ```python
   %sql
    
   SELECT YEAR(OrderDate) AS OrderYear,
          SUM((UnitPrice * Quantity) + Tax) AS GrossRevenue
   FROM salesorders
   GROUP BY YEAR(OrderDate)
   ORDER BY OrderYear;
    ```

    Observe que:
    
    - A linha **%sql** no início da célula (chamada de magic) indica que o runtime da linguagem Spark SQL deve ser usado para executar o código nessa célula, em vez do PySpark.
    - O código SQL referencia a exibição **salesorders** que você criou anteriormente.
    - A saída da consulta SQL é exibida automaticamente como o resultado abaixo da célula.
    
> **Observação**: para obter mais informações sobre o Spark SQL e os dataframes, confira a [documentação do Spark SQL](https://spark.apache.org/docs/2.2.0/sql-programming-guide.html).

## Limpar

No portal do Azure Databricks, na página **Computação**, selecione seu cluster e selecione **&#9632; Terminar** para encerrar o processo.

Se você tiver terminado de explorar o Azure Databricks, poderá excluir os recursos que criou para evitar custos desnecessários do Azure e liberar capacidade em sua assinatura.
