---
lab:
  title: Explorar dados com o Azure Databricks
---

# Explorar dados com o Azure Databricks

O Azure Databricks é uma versão baseada no Microsoft Azure da popular plataforma de código aberto Databricks. 

O Azure Databricks facilita a EDA (análise exploratória de dados), permitindo que os usuários descubram insights rapidamente e promovam a tomada de decisões. O serviço se integra a uma variedade de ferramentas e técnicas de EDA, incluindo métodos estatísticos e visualizações, para resumir as características dos dados e identificar quaisquer problemas subjacentes.

Este exercício deve levar aproximadamente **30** minutos para ser concluído.

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
7. Aguarde a conclusão do script – isso normalmente leva cerca de 5 minutos, mas em alguns casos pode levar mais tempo. Enquanto espera, revise o artigo [Análise exploratória de dados no Azure Databricks na documentação do Azure Databricks](https://learn.microsoft.com/azure/databricks/exploratory-data-analysis/).

## Criar um cluster

O Azure Databricks é uma plataforma de processamento distribuído que usa *clusters* do Apache Spark para processar dados em paralelo em vários nós. Cada cluster consiste em um nó de driver para coordenar o trabalho e nós de trabalho para executar tarefas de processamento. 

Neste exercício, você criará um cluster de *nó único* para minimizar os recursos de computação usados no ambiente de laboratório (no qual os recursos podem ser restritos). Em um ambiente de produção, você normalmente criaria um cluster com vários nós de trabalho.

> **Dica**: Se você já tiver um cluster com uma versão 13.3 LTS de runtime ou superior em seu workspace do Azure Databricks, poderá usá-lo para concluir este exercício e ignorar este procedimento.

1. No portal do Azure, navegue até o grupo de recursos **msl-*xxxxxxx*** criado pelo script (ou o grupo de recursos que contém seu workspace do Azure Databricks)
1. Selecione o recurso Serviço do Azure Databricks (chamado **databricks-*xxxxxxx*** se você usou o script de instalação para criá-lo).
1. Na página **Visão geral** do seu workspace, use o botão **Iniciar workspace** para abrir seu workspace do Azure Databricks em uma nova guia do navegador, fazendo o logon se solicitado.

    > **Dica**: ao usar o portal do workspace do Databricks, várias dicas e notificações podem ser exibidas. Dispense-as e siga as instruções fornecidas para concluir as tarefas neste exercício.

1. Na barra lateral à esquerda, selecione a tarefa **(+) Novo** e, em seguida, selecione **Cluster** (talvez você precise procurar no submenu **Mais**).
1. Na página **Novo cluster**, crie um novo cluster com as seguintes configurações:
    - **Nome do cluster**: cluster *Nome do Usuário* (o nome do cluster padrão)
    - **Política**: Sem restrições
    - **Modo de cluster**: Nó Único
    - **Modo de acesso**: Usuário único (*com sua conta de usuário selecionada*)
    - **Versão do runtime do Databricks**: 13.3 LTS (Spark 3.4.1, Scala 2.12) ou posterior
    - **Usar Aceleração do Photon**: Selecionado
    - **Tipo de nó**: Standard_D4ds_v5
    - **Encerra após** *20* **minutos de inatividade**

1. Aguarde a criação do cluster. Isso pode levar alguns minutos.

> **Observação**: se o cluster não for iniciado, sua assinatura pode ter cota insuficiente na região onde seu workspace do Azure Databricks está provisionado. Consulte [Limite de núcleo da CPU impede a criação do cluster](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit) para obter detalhes. Se isso acontecer, você pode tentar excluir seu workspace e criar um novo workspace em uma região diferente. Você pode especificar uma região como um parâmetro para o script de instalação da seguinte maneira: `./mslearn-databricks/setup.ps1 eastus`
    
## Criar um notebook

1. Na barra lateral, use o link **(+) Novo** para criar um **Notebook**.
   
1. Altere o nome padrão do notebook (**Notebook Sem Título *[data]***) para `Explore data with Spark` e, na lista suspensa **Conectar**, selecione o cluster, caso ainda não esteja selecionado. Se o cluster não executar, é porque ele pode levar cerca de um minuto para iniciar.

## Ingerir dados

1. Na primeira célula do notebook, insira o código a seguir, que usa os comandos de *shell* para baixar os arquivos de dados do GitHub para o sistema de arquivos usado pelo cluster.

    ```python
    %sh
    rm -r /dbfs/spark_lab
    mkdir /dbfs/spark_lab
    wget -O /dbfs/spark_lab/2019.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/2019.csv
    wget -O /dbfs/spark_lab/2020.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/2020.csv
    wget -O /dbfs/spark_lab/2021.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/2021.csv
    ```

1. Use a opção de menu **&#9656; Executar Célula** à esquerda da célula para executá-la. Em seguida, aguarde o término do trabalho do Spark executado pelo código.

## Consultar dados em arquivos

1. Passe o mouse sob a célula de código existente e use o ícone **+ Código** exibido para adicionar uma nova célula de código. Então, na nova célula, insira e execute o código a seguir para carregar os dados dos arquivos e exibir as primeiras 100 linhas.

    ```python
   df = spark.read.load('spark_lab/*.csv', format='csv')
   display(df.limit(100))
    ```

1. Exiba a saída e observe que os dados no arquivo estão relacionados a pedidos de vendas, mas não incluem os cabeçalhos de coluna ou informações sobre os tipos de dados. Para entender mais os dados, você pode definir um *esquema* para o dataframe.

1. Adicione uma nova célula de código e use-a para executar o seguinte código, que define um esquema para os dados:

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

1. Observe que, desta vez, o dataframe inclui os cabeçalhos de coluna. Em seguida, adicione uma nova célula de código e use-a para executar o código a seguir para exibir os detalhes do esquema do dataframe e verificar se os tipos de dados corretos foram aplicados:

    ```python
   df.printSchema()
    ```

## Consultar dados usando Spark SQL

1. Adicione uma nova célula de código e use-a para executar o código a seguir:

    ```python
   df.createOrReplaceTempView("salesorders")
   spark_df = spark.sql("SELECT * FROM salesorders")
   display(spark_df)
    ```

   Os métodos nativos do objeto dataframe usado anteriormente permitem que você consulte e analise dados com bastante eficiência. No entanto, muitos analistas de dados se sentem mais à vontade em trabalhar com a sintaxe SQL. O Spark SQL é uma API de linguagem SQL no Spark que você pode usar para executar instruções SQL ou até mesmo persistir dados em tabelas relacionais.

   O código que você acabou de executar cria uma *exibição* relacional dos dados em um dataframe e, em seguida, usa a biblioteca **spark.sql** para inserir a sintaxe do Spark SQL em seu código Python, consultando a exibição e retornando os resultados como um dataframe.

## Exibir resultados como uma visualização

1. Em uma nova célula de código, execute o código a seguir para consultar a tabela **salesorders**:

    ```sql
   %sql
    
   SELECT * FROM salesorders
    ```

1. Acima da tabela de resultados, selecione **+** e, em seguida, selecione **Visualização** para exibir o editor de visualização e aplique as seguintes opções:
    - **Tipo de visualização**: Barra
    - **Coluna X**: Item
    - **Coluna Y**: *Adicione uma nova coluna e selecione* **Quantidade**. *Aplicar a* **Agregação** de *soma*.
    
1. Salve a visualização e execute novamente a célula de código para exibir o gráfico resultante no notebook.

## Introdução à matplotlib

1. Em uma nova célula de código, execute o código a seguir para recuperar alguns dados de pedidos de venda em um dataframe:

    ```python
   sqlQuery = "SELECT CAST(YEAR(OrderDate) AS CHAR(4)) AS OrderYear, \
                   SUM((UnitPrice * Quantity) + Tax) AS GrossRevenue \
            FROM salesorders \
            GROUP BY CAST(YEAR(OrderDate) AS CHAR(4)) \
            ORDER BY OrderYear"
   df_spark = spark.sql(sqlQuery)
   df_spark.show()
    ```

1. Adicione uma nova célula de código e use-a para executar o código a seguir, que importa o **matplotlb** e o usa para criar um gráfico:

    ```python
   from matplotlib import pyplot as plt
    
   # matplotlib requires a Pandas dataframe, not a Spark one
   df_sales = df_spark.toPandas()
   # Create a bar plot of revenue by year
   plt.bar(x=df_sales['OrderYear'], height=df_sales['GrossRevenue'])
   # Display the plot
   plt.show()
    ```

1. Analise os resultados, que consistem em um gráfico de colunas com a receita bruta total de cada ano. Observe os seguintes recursos do código usado para produzir este gráfico:
    - A biblioteca **matplotlib** exige um dataframe do Pandas, ou seja, você precisa converter o dataframe do Spark retornado pela consulta Spark SQL nesse formato.
    - No núcleo da biblioteca **matplotlib** está o objeto **pyplot**. Essa é a base para a maioria das funcionalidades de plotagem.

1. As configurações padrão resultam em um gráfico utilizável, mas há muitas possibilidades de personalização. Adicione uma nova célula de código com o código a seguir e execute-a:

    ```python
   # Clear the plot area
   plt.clf()
   # Create a bar plot of revenue by year
   plt.bar(x=df_sales['OrderYear'], height=df_sales['GrossRevenue'], color='orange')
   # Customize the chart
   plt.title('Revenue by Year')
   plt.xlabel('Year')
   plt.ylabel('Revenue')
   plt.grid(color='#95a5a6', linestyle='--', linewidth=2, axis='y', alpha=0.7)
   plt.xticks(rotation=45)
   # Show the figure
   plt.show()
    ```

1. Tecnicamente, um gráfico está contido com uma **Figura**. Nos exemplos anteriores, a figura foi criada implicitamente, mas você pode criá-la de modo explícito. Tente executar o seguinte em uma nova célula:

    ```python
   # Clear the plot area
   plt.clf()
   # Create a Figure
   fig = plt.figure(figsize=(8,3))
   # Create a bar plot of revenue by year
   plt.bar(x=df_sales['OrderYear'], height=df_sales['GrossRevenue'], color='orange')
   # Customize the chart
   plt.title('Revenue by Year')
   plt.xlabel('Year')
   plt.ylabel('Revenue')
   plt.grid(color='#95a5a6', linestyle='--', linewidth=2, axis='y', alpha=0.7)
   plt.xticks(rotation=45)
   # Show the figure
   plt.show()
    ```

1. Uma figura pode conter vários subgráficos, cada um em um eixo próprio. Use este código para criar vários gráficos:

    ```python
   # Clear the plot area
   plt.clf()
   # Create a figure for 2 subplots (1 row, 2 columns)
   fig, ax = plt.subplots(1, 2, figsize = (10,4))
   # Create a bar plot of revenue by year on the first axis
   ax[0].bar(x=df_sales['OrderYear'], height=df_sales['GrossRevenue'], color='orange')
   ax[0].set_title('Revenue by Year')
   # Create a pie chart of yearly order counts on the second axis
   yearly_counts = df_sales['OrderYear'].value_counts()
   ax[1].pie(yearly_counts)
   ax[1].set_title('Orders per Year')
   ax[1].legend(yearly_counts.keys().tolist())
   # Add a title to the Figure
   fig.suptitle('Sales Data')
   # Show the figure
   plt.show()
    ```

> **Observação**: para saber mais sobre a plotagem com a matplotlib, confira a [documentação da matplotlib](https://matplotlib.org/).

## Usar a biblioteca seaborn

1. Adicione uma nova célula de código e use-a para executar o código a seguir, que usa a biblioteca **seaborn** (que é criada sobre o matplotlib e abstrai parte de sua complexidade) para criar um gráfico:

    ```python
   import seaborn as sns
   
   # Clear the plot area
   plt.clf()
   # Create a bar chart
   ax = sns.barplot(x="OrderYear", y="GrossRevenue", data=df_sales)
   plt.show()
    ```

1. A biblioteca **seaborn** torna mais simples criar gráficos complexos de dados estatísticos e permite controlar o tema visual para criar visualizações consistentes dos dados. Execute o código a seguir em uma nova célula:

    ```python
   # Clear the plot area
   plt.clf()
   
   # Set the visual theme for seaborn
   sns.set_theme(style="whitegrid")
   
   # Create a bar chart
   ax = sns.barplot(x="OrderYear", y="GrossRevenue", data=df_sales)
   plt.show()
    ```

1. Como matplotlib. O seaborn dá suporte a vários tipos de gráfico. Execute o seguinte código para criar um gráfico de linhas:

    ```python
   # Clear the plot area
   plt.clf()
   
   # Create a bar chart
   ax = sns.lineplot(x="OrderYear", y="GrossRevenue", data=df_sales)
   plt.show()
    ```

> **Observação**: para saber mais sobre como fazer uma plotagem com a seaborn, confira a [documentação da seaborn](https://seaborn.pydata.org/index.html).

## Limpeza

No portal do Azure Databricks, na página **Computação**, selecione seu cluster e selecione **&#9632; Terminar** para encerrar o processo.

Se você tiver terminado de explorar o Azure Databricks, poderá excluir os recursos que criou para evitar custos desnecessários do Azure e liberar capacidade em sua assinatura.
