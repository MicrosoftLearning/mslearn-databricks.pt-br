---
lab:
  title: Implantar cargas de trabalho com os Trabalhos do Lakeflow no Azure Databricks
---

# Implantar cargas de trabalho com os Trabalhos do Lakeflow no Azure Databricks

Os Trabalho do Lakeflow do Azure Databricks fornecem uma plataforma robusta para implantar cargas de trabalho com eficiência. Com recursos como Trabalhos do Azure Databricks e Delta Live Tables, os usuários podem orquestrar pipelines complexos de processamento de dados, machine learning e análise.

Este laboratório levará aproximadamente **40** minutos para ser concluído.

> **Observação**: a interface do usuário do Azure Databricks está sujeita a melhorias contínuas. A interface do usuário pode ter sido alterada desde que as instruções neste exercício foram escritas.

## Provisionar um workspace do Azure Databricks

> **Dica**: Se você já tem um workspace do Azure Databricks, pode ignorar esse procedimento e usar o workspace existente.

Este exercício inclui um script para provisionar um novo workspace do Azure Databricks. O script tenta criar um recurso de workspace do Azure Databricks de camada *Premium* em uma região na qual sua assinatura do Azure tenha cota suficiente para os núcleos de computação necessários para este exercício; e pressupõe que sua conta de usuário tenha permissões suficientes na assinatura para criar um recurso de workspace do Azure Databricks. Se o script falhar devido a cota ou permissões insuficientes, você pode tentar [criar um workspace do Azure Databricks interativamente no portal do Azure](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace).

1. Em um navegador da web, faça logon no [portal do Azure](https://portal.azure.com) em `https://portal.azure.com`.
2. Use o botão **[\>_]** à direita da barra de pesquisa na parte superior da página para criar um Cloud Shell no portal do Azure selecionando um ambiente do ***PowerShell***. O Cloud Shell fornece uma interface de linha de comando em um painel na parte inferior do portal do Azure, conforme mostrado aqui:

    ![Portal do Azure com um painel do Cloud Shell](./images/cloud-shell.png)

    > **Observação**: se você já criou um Cloud Shell que usa um ambiente *Bash*, alterne-o para o ***PowerShell***.

3. Você pode redimensionar o Cloud Shell arrastando a barra de separação na parte superior do painel ou usando os ícones **&#8212;**, **&#10530;** e **X** no canto superior direito do painel para minimizar, maximizar e fechar o painel. Para obter mais informações de como usar o Azure Cloud Shell, confira a [documentação do Azure Cloud Shell](https://docs.microsoft.com/azure/cloud-shell/overview).

4. No painel do PowerShell, insira os seguintes comandos para clonar esse repositório:

     ```powershell
    rm -r mslearn-databricks -f
    git clone https://github.com/MicrosoftLearning/mslearn-databricks
     ```

5. Depois que o repositório tiver sido clonado, insira o seguinte comando para executar **setup.ps1** do script, que provisiona um workspace do Azure Databricks em uma região disponível:

     ```powershell
    ./mslearn-databricks/setup.ps1
     ```

6. Se solicitado, escolha qual assinatura você deseja usar (isso só acontecerá se você tiver acesso a várias assinaturas do Azure).

7. Aguarde a conclusão do script – isso normalmente leva cerca de 5 minutos, mas em alguns casos pode levar mais tempo. Enquanto você aguarda, revise o artigo [Trabalhos do Lakeflow](https://learn.microsoft.com/azure/databricks/jobs/) na documentação do Azure Databricks.

## Criar um cluster

O Azure Databricks é uma plataforma de processamento distribuído que usa *clusters* do Apache Spark para processar dados em paralelo em vários nós. Cada cluster consiste em um nó de driver para coordenar o trabalho e nós de trabalho para executar tarefas de processamento. Neste exercício, você criará um cluster de *nó único* para minimizar os recursos de computação usados no ambiente de laboratório (no qual os recursos podem ser restritos). Em um ambiente de produção, você normalmente criaria um cluster com vários nós de trabalho.

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
        
## Criar um notebook e ingerir dados

1. Na barra lateral, use o link **(+) Novo** para criar um **Notebook**.

2. Na lista suspensa **Conectar**, selecione o cluster caso ainda não esteja selecionado. Se o cluster não executar, é porque ele pode levar cerca de um minuto para iniciar.

3. Na primeira célula do notebook, insira o código a seguir, que usa os comandos de *shell* para baixar os arquivos de dados do GitHub para o sistema de arquivos usado pelo cluster.

     ```python
    %sh
    rm -r /dbfs/workflow_lab
    mkdir /dbfs/workflow_lab
    wget -O /dbfs/workflow_lab/2019.csv https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/2019_edited.csv
    wget -O /dbfs/workflow_lab/2020.csv https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/2020_edited.csv
    wget -O /dbfs/workflow_lab/2021.csv https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/2021_edited.csv
     ```

4. Use a opção de menu **&#9656; Executar Célula** à esquerda da célula para executá-la. Em seguida, aguarde o término do trabalho do Spark executado pelo código.

## Criar uma tarefa de trabalho

Implemente o fluxo de trabalho de processamento e análise de dados usando tarefas. Um trabalho é composto de uma ou mais tarefas. É possível criar tarefas de trabalho usando notebooks, JARS, pipelines do Delta Live Tables, envios de Python, Scala e Spark e aplicativos Java. Neste exercício, você criará uma tarefa como um notebook que extrai, transforma e carrega dados em gráficos de visualização. 

1. Na barra lateral, use o link **(+) Novo** para criar um **Notebook**.

2. Altere o nome padrão do notebook (**Notebook Sem Título *[data]***) para `ETL task` e, na lista suspensa **Conectar**, selecione o cluster, caso ainda não esteja selecionado. Se o cluster não executar, é porque ele pode levar cerca de um minuto para iniciar.

    Confirme se a linguagem padrão do notebook está definida como **Python**.

3. Na primeira célula do notebook, insira e execute o seguinte código, que define um esquema para os dados e carrega os conjuntos de dados em um dataframe:

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
   df = spark.read.load('/workflow_lab/*.csv', format='csv', schema=orderSchema)
   display(df.limit(100))
    ```

4. Na célula de código existente, use o ícone **+ Código** para adicionar uma nova célula de código. Em seguida, na nova célula, insira e execute o seguinte código para remover linhas duplicadas e substituir as entradas `null` pelos valores corretos:

     ```python
    from pyspark.sql.functions import col
    df = df.dropDuplicates()
    df = df.withColumn('Tax', col('UnitPrice') * 0.08)
    df = df.withColumn('Tax', col('Tax').cast("float"))
     ```
    > **Observação**: depois de atualizar os valores na coluna **Imposto**, o tipo de dados da coluna é definido como `float` novamente. Isso ocorre porque seu tipo de dados muda para `double` depois que o cálculo é feito. Como `double` tem um uso de memória maior do que `float`, é melhor para o desempenho converter a coluna de volta para `float`.

5. Execute o seguinte código em uma nova célula de código para agregar e agrupar os dados do pedido:

    ```python
   yearlySales = df.select(year("OrderDate").alias("Year")).groupBy("Year").count().orderBy("Year")
   display(yearlySales)
    ```

## Criar o fluxo de trabalho

O Azure Databricks gerencia a orquestração de tarefas, o gerenciamento de clusters, o monitoramento e o relatório de erros para todos os seus trabalhos. Você pode executar seus trabalhos imediatamente, periodicamente por meio de um sistema de agendamento fácil de usar, sempre que novos arquivos chegarem em um local externo ou continuamente para garantir que uma instância do trabalho esteja sempre em execução.

1. No workspace, clique no ![ícone Fluxos de Trabalho.](./images/WorkflowsIcon.svg) **Trabalhos e Pipelines** na barra lateral.

2. No painel Trabalhos e Pipelines, selecione **Criar** e, em seguida, **Trabalho**.

3. Altere o nome do trabalho padrão (**Novo trabalho *[data]***) para `ETL job`.

4. Defina o trabalho com as seguintes configurações:
    - **Nome da tarefa:**: `Run ETL task notebook`
    - **Tipo**: Notebook
    - **Fonte**: Workspace
    - **Caminho**: *selecione o notebook* Tarefa de ETL**
    - **Cluster**: *Selecione seu cluster*

5. Selecione **Criar tarefa**.

6. Selecione **Executar agora**.

7. Depois que o trabalho começar a ser executado, você poderá monitorar a execução clicando em **Execuções de trabalho** na barra lateral esquerda.

8. Após a execução do trabalho, você poderá selecioná-lo e verificar a saída.

Além disso, você pode executar trabalhos que forem disparados, por exemplo, executando um fluxo de trabalho em uma agenda. Para agendar uma execução periódica do trabalho, você pode abrir a tarefa do trabalho e adicionar um gatilho.

## Limpar

No portal do Azure Databricks, na página **Computação**, selecione seu cluster e selecione **&#9632; Interromper** para encerrar o processo.

Se você tiver terminado de explorar o Azure Databricks, poderá excluir os recursos que criou para evitar custos desnecessários do Azure e liberar capacidade em sua assinatura.
