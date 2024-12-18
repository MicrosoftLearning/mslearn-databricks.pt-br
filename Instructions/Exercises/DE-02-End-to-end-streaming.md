---
lab:
  title: Pipeline de streaming de ponta a ponta com Delta Live Tables no Azure Databricks
---

# Pipeline de streaming de ponta a ponta com Delta Live Tables no Azure Databricks

A criação de um pipeline de streaming de ponta a ponta com o Delta Live Tables no Azure Databricks envolve a definição de transformações nos dados, que o Delta Live Tables gerencia por meio da orquestração de tarefas, gerenciamento de cluster e monitoramento. Essa estrutura dá suporte a tabelas de streaming para lidar com dados atualizados continuamente, exibições materializadas para transformações complexas, exibições para transformações intermediárias e verificações de qualidade de dados.

Este laboratório levará aproximadamente **30** minutos para ser concluído.

> **Observação**: a interface do usuário do Azure Databricks está sujeita a melhorias contínuas. A interface do usuário pode ter sido alterada desde que as instruções neste exercício foram escritas.

## Provisionar um workspace do Azure Databricks

> **Dica**: Se você já tem um workspace do Azure Databricks, pode ignorar esse procedimento e usar o workspace existente.

Este exercício inclui um script para provisionar um novo workspace do Azure Databricks. O script tenta criar um recurso de workspace do Azure Databricks de camada *Premium* em uma região na qual sua assinatura do Azure tenha cota suficiente para os núcleos de computação necessários para este exercício; e pressupõe que sua conta de usuário tenha permissões suficientes na assinatura para criar um recurso de workspace do Azure Databricks. Se o script falhar devido a cota ou permissões insuficientes, você pode tentar [criar um workspace do Azure Databricks interativamente no portal do Azure](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace).

1. Em um navegador da web, faça logon no [portal do Azure](https://portal.azure.com) em `https://portal.azure.com`.
2. Use o botão **[\>_]** à direita da barra de pesquisa na parte superior da página para criar um Cloud Shell no portal do Azure selecionando um ambiente do ***PowerShell***. O Cloud Shell fornece uma interface de linha de comando em um painel na parte inferior do portal do Azure, conforme mostrado aqui:

    ![Portal do Azure com um painel do Cloud Shell](./images/cloud-shell.png)

    > **Observação**: se você já criou um Cloud Shell que usa um ambiente *Bash* , alterne-o para o ***PowerShell***.

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

7. Aguarde a conclusão do script - isso normalmente leva cerca de 5 minutos, mas em alguns casos pode levar mais tempo. Enquanto você aguarda, revise o artigo [Introdução ao Delta Lake](https://docs.microsoft.com/azure/databricks/delta/delta-intro) na documentação do Azure Databricks.

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

1. Na barra lateral, use o link **(+) Novo** para criar um **Notebook**. Na lista suspensa **Conectar**, selecione o cluster caso ainda não esteja selecionado. Se o cluster não executar, é porque ele pode levar cerca de um minuto para iniciar.
2. Altere o nome do notebook padrão (**Notebook sem título *[data]***) para **Ingestão do Delta Live Tables**.

3. Na primeira célula do notebook, insira o código a seguir, que usa os comandos de *shell* para baixar os arquivos de dados do GitHub para o sistema de arquivos usado pelo cluster.

     ```python
    %sh
    rm -r /dbfs/device_stream
    mkdir /dbfs/device_stream
    wget -O /dbfs/device_stream/device_data.csv https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/device_data.csv
     ```

4. Use a opção de menu **&#9656; Executar Célula** à esquerda da célula para executá-la. Em seguida, aguarde o término do trabalho do Spark executado pelo código.

## Usar tabelas delta para transmitir dados

O Delta Lake dá suporte a dados de *streaming*. As tabelas delta podem ser um *coletor* ou uma *fonte* para fluxos de dados criados por meio da API de Streaming Estruturado do Spark. Neste exemplo, você usará uma tabela delta como um coletor para alguns dados de streaming em um cenário simulado de IoT (Internet das Coisas). Na próxima tarefa, essa tabela delta funcionará como uma fonte para transformação de dados em tempo real.

1. Em uma nova célula, execute o seguinte código para criar um fluxo com base na pasta que contém os dados do dispositivo CSV:

     ```python
    from pyspark.sql.functions import *
    from pyspark.sql.types import *

    # Define the schema for the incoming data
    schema = StructType([
        StructField("device_id", StringType(), True),
        StructField("timestamp", TimestampType(), True),
        StructField("temperature", DoubleType(), True),
        StructField("humidity", DoubleType(), True)
    ])

    # Read streaming data from folder
    inputPath = '/device_stream/'
    iotstream = spark.readStream.schema(schema).option("header", "true").csv(inputPath)
    print("Source stream created...")

    # Write the data to a Delta table
    query = (iotstream
             .writeStream
             .format("delta")
             .option("checkpointLocation", "/tmp/checkpoints/iot_data")
             .start("/tmp/delta/iot_data"))
     ```

2. Use a opção de menu **&#9656; Executar Célula** à esquerda da célula para executá-la.

Essa tabela delta agora será a fonte para transformação de dados em tempo real.

   > Observação: a célula de código acima cria o fluxo de origem. Portanto, a execução do trabalho nunca será alterada para um status concluído. Para interromper manualmente o streaming, você pode executar `query.stop()` em uma nova célula.
   
## Criar um pipeline do Delta Live Tables

Um pipeline é a unidade principal usada para configurar e executar fluxos de trabalho de processamento de dados com o Delta Live Tables. Ele vincula fontes de dados a conjuntos de dados de destino por meio de um DAG (Directed Acyclic Graph) declarado em Python ou SQL.

1. Selecione **Delta Live Tables** na barra lateral esquerda e, em seguida, selecione **Criar pipeline**.

2. Na página **Criar pipeline**, crie um novo pipeline com as seguintes configurações:
    - **Nome do pipeline**: `Ingestion Pipeline`
    - **Edição do produto**: avançado
    - **Modo do pipeline**: acionado
    - **Código-fonte**: *deixe em branco*
    - **Opções de armazenamento**: metastore do Hive
    - **Local de armazenamento**: `dbfs:/pipelines/device_stream`
    - **Esquema de destino**: `default`

3. Selecione **Criar** para criar o pipeline (que também criará um notebook em branco para o código do pipeline).

4. Depois que o pipeline for criado, abra o link para o notebook em branco em **Código-fonte** no painel à direita. O notebook abrirá em uma nova guia do navegador:

    ![delta-live-table-pipeline](./images/delta-live-table-pipeline.png)

5. Na primeira célula do notebook em branco, insira (mas não execute) o seguinte código para criar Delta Live Tables e transformar os dados:

     ```python
    import dlt
    from pyspark.sql.functions import col, current_timestamp
     
    @dlt.table(
        name="raw_iot_data",
        comment="Raw IoT device data"
    )
    def raw_iot_data():
        return spark.readStream.format("delta").load("/tmp/delta/iot_data")

    @dlt.table(
        name="transformed_iot_data",
        comment="Transformed IoT device data with derived metrics"
    )
    def transformed_iot_data():
        return (
            dlt.read("raw_iot_data")
            .withColumn("temperature_fahrenheit", col("temperature") * 9/5 + 32)
            .withColumn("humidity_percentage", col("humidity") * 100)
            .withColumn("event_time", current_timestamp())
        )
     ```

6. Feche a guia do navegador que contém o notebook (o conteúdo é salvo automaticamente) e retorne ao pipeline. Em seguida, selecione ** Iniciar **.

7. Depois que o pipeline for concluído, volte para a **Ingestão de Delta Live Tables** recente que você criou primeiro e verifique se as novas tabelas foram criadas no local de armazenamento especificado executando o seguinte código em uma nova célula:

     ```sql
    %sql
    SHOW TABLES
     ```

## Exibir resultados como uma visualização

Depois de criar as tabelas, é possível carregá-las em dataframes e visualizar os dados.

1. No primeiro notebook, adicione uma nova célula de código e execute o seguinte código para carregar o `transformed_iot_data` em um dataframe:

    ```python
    %sql
    SELECT * FROM transformed_iot_data
    ```

1. Acima da tabela de resultados, selecione **+** e, em seguida, selecione **Visualização** para exibir o editor de visualização e aplique as seguintes opções:
    - **Tipo de visualização**: linha
    - **Coluna X**: carimbo de data/hora
    - **Coluna Y**: *adicione uma nova coluna e selecione***temperature_fahrenheit**. *Aplicar a* **Agregação** de *soma*.

1. Salve a visualização para exibir o gráfico resultante no notebook.
1. Adicione uma nova célula de código e insira o seguinte código para interromper a consulta de streaming:

    ```python
    query.stop()
    ```
    

## Limpar

No portal do Azure Databricks, na página **Computação**, selecione seu cluster e selecione **&#9632; Terminar** para encerrar o processo.

Se você tiver terminado de explorar o Azure Databricks, poderá excluir os recursos que criou para evitar custos desnecessários do Azure e liberar capacidade em sua assinatura.
