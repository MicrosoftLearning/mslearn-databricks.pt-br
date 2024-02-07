---
lab:
  title: Usar o MLflow no Azure Databricks
---

# Usar o MLflow no Azure Databricks

Neste exercício, você explorará o uso do MLflow para treinar e fornecer modelos de machine learning no Azure Databricks.

Este exercício deve levar aproximadamente **45** minutos para ser concluído.

## Antes de começar

É necessário ter uma [assinatura do Azure](https://azure.microsoft.com/free) com acesso de nível administrativo.

## Provisionar um workspace do Azure Databricks

> **Observação**: Para este exercício, você precisa de um workspace do Azure Databricks **Premium** em uma região com suporte para *serviço de modelos*. Consulte as [regiões do Azure Databricks](https://learn.microsoft.com/azure/databricks/resources/supported-regions) para obter detalhes sobre os recursos regionais do Azure Databricks. Se você já possui um workspace do Azure Databricks *Premium* ou *Avaliação* em uma região adequada, pode ignorar esse procedimento e usar o workspace existente.

Este exercício inclui um script para provisionar um novo workspace do Azure Databricks. O script tenta criar um recurso de workspace do Azure Databricks de camada *Premium* em uma região na qual sua assinatura do Azure tenha cota suficiente para os núcleos de computação necessários para este exercício; e pressupõe que sua conta de usuário tenha permissões suficientes na assinatura para criar um recurso de workspace do Azure Databricks. Se o script falhar devido a cotas ou permissões insuficientes, tente criar um workspace do Azure Databricks interativamente no portal do Azure.

1. Em um navegador da web, entre no [portal da Azure](https://portal.azure.com) em `https://portal.azure.com`.
2. Use o botão **[\>_]** à direita da barra de pesquisa na parte superior da página para criar um Cloud Shell no portal do Azure, selecionando um ambiente ***PowerShell*** e criando um armazenamento caso solicitado. O Cloud Shell fornece uma interface de linha de comando em um painel na parte inferior do portal do Azure, conforme mostrado aqui:

    ![Portal do Azure com um painel do Cloud Shell](./images/cloud-shell.png)

    > **Observação**: se você tiver criado anteriormente um cloud shell que usa um ambiente *Bash*, use o menu suspenso no canto superior esquerdo do painel do cloud shell para alterá-lo para ***PowerShell***.

3. Observe que você pode redimensionar o Cloud Shell arrastando a barra do separador na parte superior do painel ou usando os ícones **&#8212;** , **&#9723;** e **X** no canto superior direito do painel para minimizar, maximizar e fechar o painel. Para obter mais informações de como usar o Azure Cloud Shell, confira a [documentação do Azure Cloud Shell](https://docs.microsoft.com/azure/cloud-shell/overview).

4. No painel do PowerShell, insira os seguintes comandos para clonar esse repositório:

    ```
    rm -r mslearn-databricks -f
    git clone https://github.com/MicrosoftLearning/mslearn-databricks
    ```

5. Após clonar o repositório, insira o seguinte comando para executar o script **setup.ps1** para provisionar um workspace do Azure Databricks em uma região disponível:

    ```
    ./mslearn-databricks/setup.ps1
    ```

6. Se solicitado, escolha qual assinatura você deseja usar (isso só acontecerá se você tiver acesso a várias assinaturas do Azure).
7. Aguarde a conclusão do script - isso normalmente leva cerca de 5 minutos, mas em alguns casos pode levar mais tempo. Enquanto você aguarda, revise o artigo [guia do MLflow](https://learn.microsoft.com/azure/databricks/mlflow/) na documentação do Azure Databricks.

## Criar um cluster

O Azure Databricks é uma plataforma de processamento distribuído que usa *clusters* do Apache Spark para processar dados em paralelo em vários nós. Cada cluster consiste em um nó de driver para coordenar o trabalho e nós de trabalho para executar tarefas de processamento. Neste exercício, você criará um cluster de *nó único* para minimizar os recursos de computação usados no ambiente de laboratório (no qual os recursos podem ser restritos). Em um ambiente de produção, você normalmente criaria um cluster com vários nós de trabalho.

> **Dica**: Se você já tiver um cluster com uma versão de runtime 13.3 LTS **<u>ML</u>** ou superior em seu workspace do Azure Databricks, poderá usá-lo para concluir este exercício e pular esta etapa.

1. No portal do Azure, navegue até o grupo de recursos **msl-*xxxxxxx*** criado pelo script (ou o grupo de recursos que contém seu workspace do Azure Databricks)
1. Selecione o recurso Serviço do Azure Databricks (chamado **databricks-*xxxxxxx*** se você usou o script de instalação para criá-lo).
1. Na página **Visão geral** do seu workspace, use o botão **Iniciar workspace** para abrir seu workspace do Azure Databricks em uma nova guia do navegador, fazendo o logon se solicitado.

    > **Dica**: ao usar o portal do workspace do Databricks, várias dicas e notificações podem ser exibidas. Dispense-as e siga as instruções fornecidas para concluir as tarefas neste exercício.

1. Na barra lateral à esquerda, selecione **(+) Nova** tarefa e, em seguida, selecione **Cluster**.
1. Na página **Novo Cluster**, crie um novo cluster com as seguintes configurações:
    - **Nome do cluster**: cluster *Nome do Usuário* (o nome do cluster padrão)
    - **Política**: Sem restrições
    - **Modo de cluster**: Nó Único
    - **Modo de acesso**: Usuário único (*com sua conta de usuário selecionada*)
    - **Versão de runtime do Databricks**: *Selecione a edição**<u> ML</u>** da versão não beta mais recente do runtime (**Não** uma versão de runtime Standard) que:*
        - ***Não** use uma GPU*
        - *Inclui o Scala > **2.11***
        - *Inclui o Spark > **3.4***
    - **Use o Photon Acceleration**: <u>Não</u> selecionado
    - **Tipo de nó**: Standard_DS3_v2
    - **Encerrar após** *20* **minutos de inatividade**

1. Aguarde a criação do cluster. Isso pode levar alguns minutos.

> **Observação**: se o cluster não for iniciado, sua assinatura pode ter cota insuficiente na região onde seu workspace do Azure Databricks está provisionado. Consulte [Limite de núcleo da CPU impede a criação do cluster](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit) para obter detalhes. Se isso acontecer, você pode tentar excluir seu workspace e criar um novo workspace em uma região diferente. Você pode especificar uma região como um parâmetro para o script de instalação da seguinte maneira: `./mslearn-databricks/setup.ps1 eastus`

## Criar um notebook

Você executará o código que usa a biblioteca Spark MLLib para treinar um modelo de machine learning. Portanto, a primeira etapa é criar um novo notebook em seu workspace.

1. Na barra lateral, use o link **(+) Novo** para criar um **Notebook**.
1. Altere o nome padrão do notebook (**Notebook Sem Título *[data]***) para **MLflow**. Na lista suspensa **Conectar**, selecione o cluster, caso ainda não esteja selecionado. Se o cluster não executar, é porque ele pode levar cerca de um minuto para iniciar.

## Ingerir e preparar dados

O cenário deste exercício baseia-se em observações de pinguins na Antártida, com o objetivo de treinar um modelo de machine learning para prever a espécie de um pinguim observado, considerando sua localização e medidas corporais.

> **Citação**: O conjunto de dados sobre pinguins usado neste exercício é um subconjunto dos dados coletados e disponibilizados pela [Dra. Kristen Gorman](https://www.uaf.edu/cfos/people/faculty/detail/kristen-gorman.php) e pela [Estação Palmer, LTER Antártida](https://pal.lternet.edu/), membro da [Rede LTER (Rede de Pesquisa Ecológica de Longo Prazo)](https://lternet.edu/).

1. Na primeira célula do notebook, insira o código a seguir, que usa comandos *shell* para baixar os dados do GitHub no sistema de arquivos do Databricks (DBFS) usado pelo cluster.

    ```bash
    %sh
    rm -r /dbfs/mlflow_lab
    mkdir /dbfs/mlflow_lab
    wget -O /dbfs/mlflow_lab/penguins.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/penguins.csv
    ```

1. Use a opção de menu **&#9656; Executar Célula** no canto superior direito da célula seguinte para executá-la. Em seguida, aguarde o término do trabalho do Spark executado pelo código.

1. Agora, prepare os dados para o aprendizado de máquina. Na célula de código existente, use o ícone **+** para adicionar uma nova célula de código. Em seguida, insira o código a seguir na nova célula para:
    - Remover todas as linhas incompletas
    - Aplicar tipos de dados apropriados
    - Visualizar uma amostra aleatória dos dados
    - Dividir os dados em dois conjuntos: um para treinamento e outro para testes.


    ```python
   from pyspark.sql.types import *
   from pyspark.sql.functions import *
   
   data = spark.read.format("csv").option("header", "true").load("/mlflow_lab/penguins.csv")
   data = data.dropna().select(col("Island").astype("string"),
                               col("CulmenLength").astype("float"),
                               col("CulmenDepth").astype("float"),
                               col("FlipperLength").astype("float"),
                               col("BodyMass").astype("float"),
                               col("Species").astype("int")
                             )
   display(data.sample(0.2))
   
   splits = data.randomSplit([0.7, 0.3])
   train = splits[0]
   test = splits[1]
   print ("Training Rows:", train.count(), " Testing Rows:", test.count())
    ```

## Executar um experimento do MLflow

O MLflow permite executar experimentos que acompanham o processo de treinamento do modelo e as métricas de avaliação de log. Essa capacidade de registrar detalhes de execuções de treinamento de modelo pode ser extremamente útil no processo iterativo de criação de um modelo de machine learning eficaz.

Você pode usar as mesmas bibliotecas e técnicas que normalmente usa para treinar e avaliar um modelo (nesse caso, usaremos a biblioteca Spark MLLib), mas faça isso no contexto de um experimento do MLflow que inclui comandos adicionais para registrar informações e métricas importantes durante o processo.

1. Adicione uma nova célula e insira o código a seguir nela:

    ```python
   import mlflow
   import mlflow.spark
   from pyspark.ml import Pipeline
   from pyspark.ml.feature import StringIndexer, VectorAssembler, MinMaxScaler
   from pyspark.ml.classification import LogisticRegression
   from pyspark.ml.evaluation import MulticlassClassificationEvaluator
   import time
   
   # Start an MLflow run
   with mlflow.start_run():
       catFeature = "Island"
       numFeatures = ["CulmenLength", "CulmenDepth", "FlipperLength", "BodyMass"]
     
       # parameters
       maxIterations = 5
       regularization = 0.5
   
       # Define the feature engineering and model steps
       catIndexer = StringIndexer(inputCol=catFeature, outputCol=catFeature + "Idx")
       numVector = VectorAssembler(inputCols=numFeatures, outputCol="numericFeatures")
       numScaler = MinMaxScaler(inputCol = numVector.getOutputCol(), outputCol="normalizedFeatures")
       featureVector = VectorAssembler(inputCols=["IslandIdx", "normalizedFeatures"], outputCol="Features")
       algo = LogisticRegression(labelCol="Species", featuresCol="Features", maxIter=maxIterations, regParam=regularization)
   
       # Chain the steps as stages in a pipeline
       pipeline = Pipeline(stages=[catIndexer, numVector, numScaler, featureVector, algo])
   
       # Log training parameter values
       print ("Training Logistic Regression model...")
       mlflow.log_param('maxIter', algo.getMaxIter())
       mlflow.log_param('regParam', algo.getRegParam())
       model = pipeline.fit(train)
      
       # Evaluate the model and log metrics
       prediction = model.transform(test)
       metrics = ["accuracy", "weightedRecall", "weightedPrecision"]
       for metric in metrics:
           evaluator = MulticlassClassificationEvaluator(labelCol="Species", predictionCol="prediction", metricName=metric)
           metricValue = evaluator.evaluate(prediction)
           print("%s: %s" % (metric, metricValue))
           mlflow.log_metric(metric, metricValue)
   
           
       # Log the model itself
       unique_model_name = "classifier-" + str(time.time())
       mlflow.spark.log_model(model, unique_model_name, mlflow.spark.get_default_conda_env())
       modelpath = "/model/%s" % (unique_model_name)
       mlflow.spark.save_model(model, modelpath)
       
       print("Experiment run complete.")
    ```

1. Quando a execução do experimento for concluída, na célula de código, se necessário, use a alternância **&#9656;** para expandir os detalhes da **execução do MLflow**. Use o hiperlink do **experimento** que é exibido lá para abrir a página do MLflow que lista as execuções do experimento. Cada execução recebe um nome exclusivo.
1. Selecione a execução mais recente e exiba seus detalhes. Observe que você pode expandir seções para ver os **Parâmetros** e as **Métricas** que foram registradas e você pode ver detalhes do modelo que foi treinado e salvo.

    > **Dica**: Você também pode usar o ícone de **experimentos do MLflow** no menu da barra lateral à direita deste notebook para exibir detalhes das execuções do experimento.

## Criar uma função

Em projetos de machine learning, os cientistas de dados geralmente tentam modelos de treinamento com parâmetros diferentes, sempre registrando os resultados. Para fazer isso, é comum criar uma função que encapsula o processo de treinamento e chamá-la com os parâmetros que você deseja experimentar.

1. Em uma nova célula, execute o seguinte código para criar uma função com base no código de treinamento usado anteriormente:

    ```python
   def train_penguin_model(training_data, test_data, maxIterations, regularization):
       import mlflow
       import mlflow.spark
       from pyspark.ml import Pipeline
       from pyspark.ml.feature import StringIndexer, VectorAssembler, MinMaxScaler
       from pyspark.ml.classification import LogisticRegression
       from pyspark.ml.evaluation import MulticlassClassificationEvaluator
       import time
   
       # Start an MLflow run
       with mlflow.start_run():
   
           catFeature = "Island"
           numFeatures = ["CulmenLength", "CulmenDepth", "FlipperLength", "BodyMass"]
   
           # Define the feature engineering and model steps
           catIndexer = StringIndexer(inputCol=catFeature, outputCol=catFeature + "Idx")
           numVector = VectorAssembler(inputCols=numFeatures, outputCol="numericFeatures")
           numScaler = MinMaxScaler(inputCol = numVector.getOutputCol(), outputCol="normalizedFeatures")
           featureVector = VectorAssembler(inputCols=["IslandIdx", "normalizedFeatures"], outputCol="Features")
           algo = LogisticRegression(labelCol="Species", featuresCol="Features", maxIter=maxIterations, regParam=regularization)
   
           # Chain the steps as stages in a pipeline
           pipeline = Pipeline(stages=[catIndexer, numVector, numScaler, featureVector, algo])
   
           # Log training parameter values
           print ("Training Logistic Regression model...")
           mlflow.log_param('maxIter', algo.getMaxIter())
           mlflow.log_param('regParam', algo.getRegParam())
           model = pipeline.fit(training_data)
   
           # Evaluate the model and log metrics
           prediction = model.transform(test_data)
           metrics = ["accuracy", "weightedRecall", "weightedPrecision"]
           for metric in metrics:
               evaluator = MulticlassClassificationEvaluator(labelCol="Species", predictionCol="prediction", metricName=metric)
               metricValue = evaluator.evaluate(prediction)
               print("%s: %s" % (metric, metricValue))
               mlflow.log_metric(metric, metricValue)
   
   
           # Log the model itself
           unique_model_name = "classifier-" + str(time.time())
           mlflow.spark.log_model(model, unique_model_name, mlflow.spark.get_default_conda_env())
           modelpath = "/model/%s" % (unique_model_name)
           mlflow.spark.save_model(model, modelpath)
   
           print("Experiment run complete.")
    ```

1. Em uma nova célula, use o seguinte código para chamar sua função:

    ```python
   train_penguin_model(train, test, 10, 0.2)
    ```

1. Exiba os detalhes do experimento do MLflow para a segunda execução.

## Registrar e implantar um modelo com o MLflow

Além de acompanhar os detalhes das execuções de teste de treinamento, você pode usar o MLflow para gerenciar os modelos de machine learning treinados. Você já registrou o modelo treinado por cada execução de experimento. Você também pode *registrar* modelos e implantá-los para que eles possam ser atendidos em aplicativos cliente.

> **Observação**: O serviço de modelo só tem suporte nos workspaces do Azure Databricks *Premium* e é restrito a [algumas regiões](https://learn.microsoft.com/azure/databricks/resources/supported-regions).

1. Exiba a página de detalhes da execução mais recente do experimento.
1. Use o botão **Registrar Modelo** para registrar o modelo que foi registrado nesse experimento e, quando solicitado, crie um novo modelo chamado **Penguin Predictor**.
1. Quando o modelo tiver sido registrado, exiba a página **Modelos** (na barra de navegação à esquerda) e selecione o modelo **Penguin Predictor**.
1. Na página do modelo **Penguin Predictor**, use o botão **Usar modelo para inferência** para criar um novo ponto de extremidade em tempo real com as seguintes configurações:
    - **Modelo**: Penguin Predictor
    - **Versão do modelo**: 1
    - **Ponto de extremidade**: predict-penguin
    - **Tamanho da computação**: Small

    O ponto de extremidade de serviço é hospedado em um novo cluster, o que pode levar vários minutos para ser criado.
  
1. Quando o ponto de extremidade tiver sido criado, use o botão **Ponto de extremidade de consulta** no canto superior direito para abrir uma interface da qual você pode testar o ponto de extremidade. Em seguida, na interface de teste, na guia **Navegador**, insira a seguinte solicitação JSON e use o botão **Enviar Solicitação** para chamar o ponto de extremidade e gerar uma previsão.

    ```json
    {
      "dataframe_records": [
      {
         "Island": "Biscoe",
         "CulmenLength": 48.7,
         "CulmenDepth": 14.1,
         "FlipperLength": 210,
         "BodyMass": 4450
      }
      ]
    }
    ```

1. Experimente alguns valores diferentes para as características do Penguin e observe os resultados retornados. Em seguida, feche a interface de teste.

## Excluir o ponto de extremidade

Quando o ponto de extremidade não for mais necessário, você deverá excluí-lo para evitar custos desnecessários.

Na página **predict-penguin** do ponto de extremidade, no menu **&#8285;**, selecione **Excluir**.

## Limpeza

No portal do Azure Databricks, na página **Computação**, selecione seu cluster e escolha a opção **&#9632; Encerrar** para desligá-lo.

Se você terminou de explorar o Azure Databricks, exclua os recursos que criou para evitar custos desnecessários do Azure e liberar capacidade em sua assinatura.

> **Mais informações**: Para obter mais informações, confira a [documentação do Spark MLLib](https://spark.apache.org/docs/latest/ml-guide.html).
