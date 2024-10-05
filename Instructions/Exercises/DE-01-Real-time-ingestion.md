---
lab:
  title: Ingestão e processamento em tempo real com o Spark Structured Streaming e o Delta Lake com o Azure Databricks
---

# Ingestão e processamento em tempo real com o Spark Structured Streaming e o Delta Lake com o Azure Databricks

Com o Spark Structured Streaming, você pode processar dados em tempo real com tolerância a falhas de ponta a ponta. O Delta Lake aprimora isso fornecendo uma camada de armazenamento com transações ACID, garantindo a integridade e a consistência dos dados. Você pode assimilar dados do armazenamento em nuvem no Delta Lake e usar o Delta Live Tables para gerenciar e otimizar os pipelines de dados de streaming.

Este laboratório levará aproximadamente **30** minutos para ser concluído.

## Provisionar um workspace do Azure Databricks

> **Dica**: Se você já tem um workspace do Azure Databricks, pode ignorar esse procedimento e usar o workspace existente.

Este exercício inclui um script para provisionar um novo workspace do Azure Databricks. O script tenta criar um recurso de workspace do Azure Databricks de camada *Premium* em uma região na qual sua assinatura do Azure tenha cota suficiente para os núcleos de computação necessários para este exercício; e pressupõe que sua conta de usuário tenha permissões suficientes na assinatura para criar um recurso de workspace do Azure Databricks. Se o script falhar devido a cota ou permissões insuficientes, você pode tentar [criar um workspace do Azure Databricks interativamente no portal do Azure](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace).

1. Em um navegador da web, faça logon no [portal do Azure](https://portal.azure.com) em `https://portal.azure.com`.

2. Use o botão **[\>_]** à direita da barra de pesquisa na parte superior da página para criar um Cloud Shell no portal do Azure, selecionando um ambiente ***PowerShell*** e criando um armazenamento caso solicitado. O Cloud Shell fornece uma interface de linha de comando em um painel na parte inferior do portal do Azure, conforme mostrado aqui:

    ![Portal do Azure com um painel do Cloud Shell](./images/cloud-shell.png)

    > **Observação**: se você tiver criado anteriormente um cloud shell que usa um ambiente *Bash*, use o menu suspenso no canto superior esquerdo do painel do cloud shell para alterá-lo para ***PowerShell***.

3. Observe que você pode redimensionar o Cloud Shell arrastando a barra do separador na parte superior do painel ou usando os ícones **&#8212;** , **&#9723;** e **X** no canto superior direito do painel para minimizar, maximizar e fechar o painel. Para obter mais informações de como usar o Azure Cloud Shell, confira a [documentação do Azure Cloud Shell](https://docs.microsoft.com/azure/cloud-shell/overview).

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

1. Na barra lateral à esquerda, selecione a tarefa **(+) Novo** e, em seguida, selecione **Cluster**.

1. Na página **Novo Cluster**, crie um novo cluster com as seguintes configurações:
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

Você pode criar notebooks em seu workspace do Azure Databricks para executar código escrito em uma variedade de linguagens de programação. Neste exercício, você criará um notebook simples que ingerirá dados de um arquivo e os salvará em uma pasta no DBFS (Sistema de Arquivos do Databricks).

1. Veja o portal do workspace do Azure Databricks e observe que a barra lateral à esquerda contém ícones para as várias tarefas executáveis.

1. Na barra lateral, use o link **(+) Novo** para criar um **Notebook**.
   
1. Altere o nome do notebook padrão (**Untitled Notebook *[date]***) para **RealTimeIngestion**.

1. Na primeira célula do notebook, insira o código a seguir, que usa os comandos de *shell* para baixar os arquivos de dados do GitHub para o sistema de arquivos usado pelo cluster.

     ```python
    %sh
    rm -r /dbfs/device_stream
    mkdir /dbfs/device_stream
    wget -O /dbfs/device_stream/devices1.json https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/devices1.json
     ```

1. Use a opção de menu **&#9656; Executar Célula** à esquerda da célula para executá-la. Em seguida, aguarde o término do trabalho do Spark executado pelo código.

## Usar tabelas delta para transmitir dados

O Delta Lake dá suporte a dados de *streaming*. As tabelas delta podem ser um *coletor* ou uma *fonte* para fluxos de dados criados por meio da API de Streaming Estruturado do Spark. Neste exemplo, você usará uma tabela delta como um coletor para alguns dados de streaming em um cenário simulado de IoT (Internet das Coisas). Os dados simulados do dispositivo estão no formato JSON, da seguinte maneira:

```json
{"device":"Dev1","status":"ok"}
{"device":"Dev1","status":"ok"}
{"device":"Dev1","status":"ok"}
{"device":"Dev2","status":"error"}
{"device":"Dev1","status":"ok"}
{"device":"Dev1","status":"error"}
{"device":"Dev2","status":"ok"}
{"device":"Dev2","status":"error"}
{"device":"Dev1","status":"ok"}
```

1. Em uma nova célula, execute o seguinte código para criar um fluxo com base na pasta que contém os dados do dispositivo JSON:

    ```python
   from pyspark.sql.types import *
   from pyspark.sql.functions import *
   
   # Create a stream that reads data from the folder, using a JSON schema
   inputPath = '/device_stream/'
   jsonSchema = StructType([
   StructField("device", StringType(), False),
   StructField("status", StringType(), False)
   ])
   iotstream = spark.readStream.schema(jsonSchema).option("maxFilesPerTrigger", 1).json(inputPath)
   print("Source stream created...")
    ```

1. Adicione uma nova célula de código e use-a para gravar perpetuamente o fluxo de dados em uma pasta delta:

    ```python
   # Write the stream to a delta table
   delta_stream_table_path = '/delta/iotdevicedata'
   checkpointpath = '/delta/checkpoint'
   deltastream = iotstream.writeStream.format("delta").option("checkpointLocation", checkpointpath).start(delta_stream_table_path)
   print("Streaming to delta sink...")
    ```

1. Adicione código para ler os dados, assim como qualquer outra pasta delta:

    ```python
   # Read the data in delta format into a dataframe
   df = spark.read.format("delta").load(delta_stream_table_path)
   display(df)
    ```

1. Adicione o seguinte código para criar uma tabela com base na pasta delta à qual os dados de streaming estão sendo gravados:

    ```python
   # create a catalog table based on the streaming sink
   spark.sql("CREATE TABLE IotDeviceData USING DELTA LOCATION '{0}'".format(delta_stream_table_path))
    ```

1. Use o seguinte código para consultar a tabela:

    ```sql
   %sql
   SELECT * FROM IotDeviceData;
    ```

1. Execute o seguinte código para adicionar alguns novos dados de dispositivo ao fluxo:

    ```Bash
    %sh
    wget -O /dbfs/device_stream/devices2.json https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/devices2.json
    ```

1. Execute novamente o seguinte código de consulta SQL para verificar se os novos dados foram adicionados ao fluxo e gravados na pasta delta:

    ```sql
   %sql
   SELECT * FROM IotDeviceData;
    ```

1. Execute o código a seguir para interromper o fluxo:

    ```python
   deltastream.stop()
    ```

## Limpeza

No portal do Azure Databricks, na página **Computação**, selecione seu cluster e selecione **&#9632; Terminar** para encerrar o processo.

Se você tiver terminado de explorar o Azure Databricks, poderá excluir os recursos que criou para evitar custos desnecessários do Azure e liberar capacidade em sua assinatura.
