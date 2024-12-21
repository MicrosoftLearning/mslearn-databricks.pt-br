---
lab:
  title: Otimizar pipelines de dados para melhorar o desempenho no Azure Databricks
---

# Otimizar pipelines de dados para melhorar o desempenho no Azure Databricks

A otimização de pipelines de dados no Azure Databricks pode melhorar significativamente o desempenho e a eficiência. A utilização do Carregador Automático para ingestão incremental de dados, juntamente com a camada de armazenamento do Delta Lake, garante a confiabilidade e as transações ACID. A implementação de salting pode evitar a distorção de dados, enquanto o clustering de ordem Z otimiza as leituras de arquivos colocando informações relacionadas. Os recursos de ajuste automático do Azure Databricks e o otimizador com base em custo podem melhorar ainda mais o desempenho ajustando as configurações usando como base os requisitos de carga de trabalho.

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

1. Na barra lateral, use o link **(+) Novo** para criar um **Notebook** e altere o nome padrão do notebook (**Notebook sem título *[data]***) para **Otimizar a ingestão de dados**. Em seguida, na lista suspensa **Conectar**, selecione o cluster caso ainda não esteja selecionado. Se o cluster não executar, é porque ele pode levar cerca de um minuto para iniciar.

2. Na primeira célula do notebook, insira o código a seguir, que usa os comandos de *shell* para baixar os arquivos de dados do GitHub para o sistema de arquivos usado pelo cluster.

     ```python
    %sh
    rm -r /dbfs/nyc_taxi_trips
    mkdir /dbfs/nyc_taxi_trips
    wget -O /dbfs/nyc_taxi_trips/yellow_tripdata_2021-01.parquet https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/yellow_tripdata_2021-01.parquet
     ```

3. Na saída da primeira célula, use o ícone **+ Código** para adicionar uma nova célula e execute o seguinte código nela para carregar o conjunto de dados em um dataframe:
   
     ```python
    # Load the dataset into a DataFrame
    df = spark.read.parquet("/nyc_taxi_trips/yellow_tripdata_2021-01.parquet")
    display(df)
     ```

4. Use a opção de menu **&#9656; Executar Célula** à esquerda da célula para executá-la. Em seguida, aguarde o término do trabalho do Spark executado pelo código.

## Otimize a ingestão de dados com o carregador automático:

A otimização da ingestão de dados é importante para lidar com grandes conjuntos de dados de modo eficiente. O Carregador Automático foi projetado para processar novos arquivos de dados à medida que chegam ao armazenamento em nuvem, com suporte para vários formatos de arquivo e serviços de armazenamento em nuvem. 

O Carregador automático fornece uma fonte de Fluxo estruturado chamada `cloudFiles`. Dado um caminho de diretório de entrada no armazenamento de arquivos em nuvem, a origem `cloudFiles` processa automaticamente novos arquivos conforme chegam, com a opção de também processar arquivos existentes nesse diretório. 

1. Em uma nova célula, execute o seguinte código para criar um fluxo com base na pasta que contém os dados de exemplo:

     ```python
     df = (spark.readStream
             .format("cloudFiles")
             .option("cloudFiles.format", "parquet")
             .option("cloudFiles.schemaLocation", "/stream_data/nyc_taxi_trips/schema")
             .load("/nyc_taxi_trips/"))
     df.writeStream.format("delta") \
         .option("checkpointLocation", "/stream_data/nyc_taxi_trips/checkpoints") \
         .option("mergeSchema", "true") \
         .start("/delta/nyc_taxi_trips")
     display(df)
     ```

2. Em uma nova célula, execute o seguinte código para adicionar um novo arquivo Parquet ao fluxo:

     ```python
    %sh
    rm -r /dbfs/nyc_taxi_trips
    mkdir /dbfs/nyc_taxi_trips
    wget -O /dbfs/nyc_taxi_trips/yellow_tripdata_2021-02_edited.parquet https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/yellow_tripdata_2021-02_edited.parquet
     ```
   
    O novo arquivo tem uma nova coluna, portanto, o fluxo é interrompido com um erro `UnknownFieldException`. Antes de o fluxo gerar esse erro, o Carregador Automático executa a inferência de esquema no microlote de dados mais recente e atualiza o local do esquema com o esquema mais recente, mesclando as novas colunas ao final do esquema. Os tipos de dados das colunas existentes permanecem inalterados.

3. Execute a célula de código de streaming novamente e verifique se duas novas colunas (**new_column** e *_rescued_data**) foram adicionadas à tabela. A coluna **_rescued_data** contém todos os dados que não são analisados devido à incompatibilidade de tipo, incompatibilidade de maiúsculas e minúsculas ou coluna ausente do esquema.

4. Selecione **Interromper** para interromper o streaming de dados.
   
    Os dados de streaming são gravados em tabelas Delta. O Delta Lake fornece um conjunto de aprimoramentos em relação aos arquivos Parquet tradicionais, incluindo transações ACID, a evolução de esquema, a viagem no tempo. Além disso, unifica o streaming e o processamento de dados em lote, tornando-o uma solução avançada para gerenciar cargas de trabalho de big data.

## Otimizar a transformação de dados

A distorção de dados é um desafio significativo na computação distribuída, principalmente no processamento de big data com estruturas como o Apache Spark. Salting é uma técnica eficaz para otimizar a distorção de dados adicionando um componente aleatório, ou "salt", às chaves antes do particionamento. Esse processo ajuda a distribuir os dados de modo mais uniforme entre as partições, resultando em uma carga de trabalho mais equilibrada e melhor desempenho.

1. Em uma nova célula, execute o seguinte código para dividir uma partição distorcida grande em partições menores, acrescentando uma coluna *salt* com inteiros aleatórios:

     ```python
    from pyspark.sql.functions import lit, rand

    # Convert streaming DataFrame back to batch DataFrame
    df = spark.read.parquet("/nyc_taxi_trips/*.parquet")
     
    # Add a salt column
    df_salted = df.withColumn("salt", (rand() * 100).cast("int"))

    # Repartition based on the salted column
    df_salted.repartition("salt").write.format("delta").mode("overwrite").save("/delta/nyc_taxi_trips_salted")

    display(df_salted)
     ```   

## Otimizar o armazenamento

O Delta Lake oferece um conjunto de comandos de otimização que podem melhorar significativamente o desempenho e o gerenciamento do armazenamento de dados. O comando `optimize` foi projetado para melhorar a velocidade de consulta, organizando os dados com mais eficiência por meio de técnicas como compactação e ordens Z.

A compactação consolida arquivos menores em arquivos maiores, o que pode ser benéfico para consultas de leitura. As ordens Z envolvem a organização de pontos de dados para que as informações relacionadas sejam armazenadas juntas, reduzindo o tempo necessário para acessar esses dados durante as consultas.

1. Em uma nova célula, execute o seguinte código para realizar a compactação na tabela Delta:

     ```python
    from delta.tables import DeltaTable

    delta_table = DeltaTable.forPath(spark, "/delta/nyc_taxi_trips")
    delta_table.optimize().executeCompaction()
     ```

2. Em uma nova célula, execute o seguinte código para realizar o clustering de ordem Z:

     ```python
    delta_table.optimize().executeZOrderBy("tpep_pickup_datetime")
     ```

Essa técnica colocalizará informações relacionadas no mesmo conjunto de arquivos, melhorando o desempenho da consulta.

## Limpar

No portal do Azure Databricks, na página **Computação**, selecione seu cluster e selecione **&#9632; Terminar** para encerrar o processo.

Se você tiver terminado de explorar o Azure Databricks, poderá excluir os recursos que criou para evitar custos desnecessários do Azure e liberar capacidade em sua assinatura.
