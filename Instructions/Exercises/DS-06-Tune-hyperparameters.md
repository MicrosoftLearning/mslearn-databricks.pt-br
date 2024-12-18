---
lab:
  title: Otimizar hiperparâmetros de aprendizado de máquina no Azure Databricks
---

# Otimizar hiperparâmetros de aprendizado de máquina no Azure Databricks

Neste exercício, você usará a biblioteca **Hyperopt** para otimizar hiperparâmetros para treinamento de modelos de machine learning no Azure Databricks.

Este exercício deve levar aproximadamente **30** minutos para ser concluído.

> **Observação**: a interface do usuário do Azure Databricks está sujeita a melhorias contínuas. A interface do usuário pode ter sido alterada desde que as instruções neste exercício foram escritas.

## Antes de começar

É necessário ter uma [assinatura do Azure](https://azure.microsoft.com/free) com acesso de nível administrativo.

## Provisionar um workspace do Azure Databricks

> **Dica**: Se você já tem um workspace do Azure Databricks, pode ignorar esse procedimento e usar o workspace existente.

Este exercício inclui um script para provisionar um novo workspace do Azure Databricks. O script tenta criar um recurso de workspace do Azure Databricks de camada *Premium* em uma região na qual sua assinatura do Azure tenha cota suficiente para os núcleos de computação necessários para este exercício; e pressupõe que sua conta de usuário tenha permissões suficientes na assinatura para criar um recurso de workspace do Azure Databricks. Se o script falhar devido a cota ou permissões insuficientes, você pode tentar [criar um workspace do Azure Databricks interativamente no portal do Azure](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace).

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
7. Aguarde a conclusão do script - isso normalmente leva cerca de 5 minutos, mas em alguns casos pode levar mais tempo. Enquanto você aguarda, revise o artigo [Ajuste de hiperparâmetros](https://learn.microsoft.com/azure/databricks/machine-learning/automl-hyperparam-tuning/) na documentação do Azure Databricks.

## Criar um cluster

O Azure Databricks é uma plataforma de processamento distribuído que usa *clusters* do Apache Spark para processar dados em paralelo em vários nós. Cada cluster consiste em um nó de driver para coordenar o trabalho e nós de trabalho para executar tarefas de processamento. Neste exercício, você criará um cluster de *nó único* para minimizar os recursos de computação usados no ambiente de laboratório (no qual os recursos podem ser restritos). Em um ambiente de produção, você normalmente criaria um cluster com vários nós de trabalho.

> **Dica**: Se você já tiver um cluster com uma versão de runtime 13.3 LTS **<u>ML</u>** ou superior em seu workspace do Azure Databricks, poderá usá-lo para concluir este exercício e ignorar este procedimento.

1. No portal do Azure, navegue até o grupo de recursos **msl-*xxxxxxx*** criado pelo script (ou o grupo de recursos que contém seu workspace do Azure Databricks existente)
1. Selecione o recurso Serviço do Azure Databricks (chamado **databricks-*xxxxxxx*** se você usou o script de instalação para criá-lo).
1. Na página **Visão geral** do seu workspace, use o botão **Iniciar workspace** para abrir seu workspace do Azure Databricks em uma nova guia do navegador, fazendo o logon se solicitado.

    > **Dica**: ao usar o portal do workspace do Databricks, várias dicas e notificações podem ser exibidas. Dispense-as e siga as instruções fornecidas para concluir as tarefas neste exercício.

1. Na barra lateral à esquerda, selecione a tarefa **(+) Novo** e, em seguida, selecione **Cluster**.
1. Na página **Novo Cluster**, crie um novo cluster com as seguintes configurações:
    - **Nome do cluster**: cluster *Nome do Usuário* (o nome do cluster padrão)
    - **Política**: Sem restrições
    - **Modo de cluster**: Nó Único
    - **Modo de acesso**: Usuário único (*com sua conta de usuário selecionada*)
    - **Versão do runtime do Databricks**: *Selecione a edição do **<u>ML</u>** da última versão não beta do runtime (**Não** uma versão de runtime Standard) que:*
        - ***Não** usa uma GPU*
        - *Inclui o Scala > **2.11***
        - *Inclui o Spark > **3.4***
    - **Usa a Aceleração do Photon**: <u>Não</u> selecionado
    - **Tipo de nó**: Standard_D4ds_v5
    - **Encerra após** *20* **minutos de inatividade**

1. Aguarde a criação do cluster. Isso pode levar alguns minutos.

> **Observação**: se o cluster não for iniciado, sua assinatura pode ter cota insuficiente na região onde seu workspace do Azure Databricks está provisionado. Consulte [Limite de núcleo da CPU impede a criação do cluster](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit) para obter detalhes. Se isso acontecer, você pode tentar excluir seu workspace e criar um novo workspace em uma região diferente. Você pode especificar uma região como um parâmetro para o script de instalação da seguinte maneira: `./mslearn-databricks/setup.ps1 eastus`

## Criar um notebook

Você executará o código que usa a biblioteca MLLib do Spark para treinar um modelo de machine learning. Portanto, a primeira etapa é criar um novo notebook em seu workspace.

1. Na barra lateral, use o link **(+) Novo** para criar um **Notebook**.
1. Altere o nome padrão do notebook (**Notebook Sem Título *[data]***) para **Ajuste de Hiperparâmetro**. Na lista suspensa **Conectar**, selecione o cluster, caso ainda não esteja selecionado. Se o cluster não executar, é porque ele pode levar cerca de um minuto para iniciar.

## Ingerir dados

O cenário deste exercício baseia-se em observações de pinguins na Antártida, com o objetivo de treinar um modelo de machine learning para prever a espécie de um pinguim observado, considerando sua localização e medidas corporais.

> **Citação**: O conjunto de dados sobre pinguins usado neste exercício é um subconjunto dos dados coletados e disponibilizados pela [Dra. Kristen Gorman](https://www.uaf.edu/cfos/people/faculty/detail/kristen-gorman.php) e pela [Estação Palmer, LTER Antártida](https://pal.lternet.edu/), membro da [Rede LTER (Rede de Pesquisa Ecológica de Longo Prazo)](https://lternet.edu/).

1. Na primeira célula do notebook, insira o código a seguir, que usa os comandos de *shell* para baixar os dados sobre pinguins do GitHub para o sistema de arquivos usado pelo cluster.

    ```bash
    %sh
    rm -r /dbfs/hyperopt_lab
    mkdir /dbfs/hyperopt_lab
    wget -O /dbfs/hyperopt_lab/penguins.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/penguins.csv
    ```

1. Use a opção de menu **&#9656; Executar Célula** à esquerda da célula para executá-la. Em seguida, aguarde o término do trabalho do Spark executado pelo código.
1. Agora, prepare os dados para o aprendizado de máquina. Na célula de código existente, use o ícone **+** para adicionar uma nova célula de código. Em seguida, insira o código a seguir na nova célula para:
    - Remover todas as linhas incompletas
    - Aplicar tipos de dados apropriados
    - Visualizar uma amostra aleatória dos dados
    - Dividir os dados em dois conjuntos: um para treinamento e outro para testes.


    ```python
   from pyspark.sql.types import *
   from pyspark.sql.functions import *
   
   data = spark.read.format("csv").option("header", "true").load("/hyperopt_lab/penguins.csv")
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

## Otimizar os valores dos hiperparâmetros para treinar um modelo

Você treina um modelo de machine learning ajustando os recursos a um algoritmo que calcula o rótulo mais provável. Os algoritmos recebem os dados de treinamento como parâmetro e tentam calcular uma relação matemática entre os recursos e os rótulos. Além dos dados, a maioria dos algoritmos usa um ou mais *hiperparâmetros* para influenciar a forma como a relação é calculada. A determinação dos valores ideais para o hiperparâmetro é uma parte importante do processo de treinamento de modelo iterativo.

Para ajudá-lo a determinar valores ideais dos hiperparâmetros, o Azure Databricks possui suporte para o **Hyperopt** — uma biblioteca experimentar vários valores de hiperparâmetro e encontrar a melhor combinação para os dados.

O primeiro passo ao usar o Hyperopt é criar uma função que:

- Treina um modelo usando um ou mais valores de hiperparâmetros que são passados para a função como parâmetros.
- Calcula uma métrica de desempenho que pode ser usada para medir a *perda* (quão distante o modelo está do desempenho de previsão perfeito)
- Retorna o valor de perda para que possa ser otimizado (minimizado) iterativamente ao tentar diferentes valores de hiperparâmetros

1. Adicione uma nova célula e use o código a seguir para criar uma função que usa os dados de pinguins para treinar um modelo de classificação capaz de prever a espécie de um pinguim com base em sua localização e medidas:

    ```python
   from hyperopt import STATUS_OK
   import mlflow
   from pyspark.ml import Pipeline
   from pyspark.ml.feature import StringIndexer, VectorAssembler, MinMaxScaler
   from pyspark.ml.classification import DecisionTreeClassifier
   from pyspark.ml.evaluation import MulticlassClassificationEvaluator
   
   def objective(params):
       # Train a model using the provided hyperparameter value
       catFeature = "Island"
       numFeatures = ["CulmenLength", "CulmenDepth", "FlipperLength", "BodyMass"]
       catIndexer = StringIndexer(inputCol=catFeature, outputCol=catFeature + "Idx")
       numVector = VectorAssembler(inputCols=numFeatures, outputCol="numericFeatures")
       numScaler = MinMaxScaler(inputCol = numVector.getOutputCol(), outputCol="normalizedFeatures")
       featureVector = VectorAssembler(inputCols=["IslandIdx", "normalizedFeatures"], outputCol="Features")
       mlAlgo = DecisionTreeClassifier(labelCol="Species",    
                                       featuresCol="Features",
                                       maxDepth=params['MaxDepth'], maxBins=params['MaxBins'])
       pipeline = Pipeline(stages=[catIndexer, numVector, numScaler, featureVector, mlAlgo])
       model = pipeline.fit(train)
       
       # Evaluate the model to get the target metric
       prediction = model.transform(test)
       eval = MulticlassClassificationEvaluator(labelCol="Species", predictionCol="prediction", metricName="accuracy")
       accuracy = eval.evaluate(prediction)
       
       # Hyperopt tries to minimize the objective function, so you must return the negative accuracy.
       return {'loss': -accuracy, 'status': STATUS_OK}
    ```

1. Adicione uma nova célula e use o código a seguir para:
    - Definir um espaço de pesquisa que especifica o intervalo de valores a serem usados para um ou mais hiperparâmetros (confira [Definir um espaço de pesquisa](http://hyperopt.github.io/hyperopt/getting-started/search_spaces/) na documentação do Hyperopt para mais detalhes).
    - Especifique o algoritmo Hyperopt que você deseja usar (confira [Algoritmos](http://hyperopt.github.io/hyperopt/#algorithms) na documentação do Hyperopt para mais detalhes).
    - Use a função **hyperopt.fmin** para chamar sua função de treinamento repetidamente e tentar minimizar a perda.

    ```python
   from hyperopt import fmin, tpe, hp
   
   # Define a search space for two hyperparameters (maxDepth and maxBins)
   search_space = {
       'MaxDepth': hp.randint('MaxDepth', 10),
       'MaxBins': hp.choice('MaxBins', [10, 20, 30])
   }
   
   # Specify an algorithm for the hyperparameter optimization process
   algo=tpe.suggest
   
   # Call the training function iteratively to find the optimal hyperparameter values
   argmin = fmin(
     fn=objective,
     space=search_space,
     algo=algo,
     max_evals=6)
   
   print("Best param values: ", argmin)
    ```

1. Observe como o código executa iterativamente a função de treinamento 6 vezes (com base na configuração de **max_evals** ). Cada execução é registrada pelo MLflow e você pode usar o botão **&#9656;** para expandir a saída da **execução do MLflow** na célula de código e selecionar o hiperlink do **experimento** para visualizá-los. Cada execução recebe um nome aleatório, e você pode visualizar cada uma delas no visualizador de execuções do MLflow para ver detalhes dos parâmetros e métricas registrados.
1. Quando todas as execuções terminarem, observe que o código exibe detalhes dos melhores valores de hiperparâmetros encontrados (a combinação que resultou na menor perda). Nesse caso, o parâmetro **MaxBins** é definido como uma opção de uma lista de três valores possíveis (10, 20 e 30). O melhor valor indica o item com base zero na lista (então 0=10, 1=20 e 2=30). O parâmetro **MaxDepth** é definido como um número inteiro aleatório entre 0 e 10, e o valor inteiro que apresentou o melhor resultado é exibido. Para obter mais informações sobre a especificação de escopos de valores de hiperparâmetros para espaços de pesquisa, confira [Expressões de Parâmetros](http://hyperopt.github.io/hyperopt/getting-started/search_spaces/#parameter-expressions) na documentação do Hyperopt.

## Use a classe Trials para registrar em log os detalhes da execução

Além de usar as execuções experimentais do MLflow para registrar os detalhes de cada iteração, você também pode usar a classe **hyperopt.Trials** para registrar e visualizar detalhes de cada execução.

1. Adicione uma nova célula e use o código a seguir para exibir detalhes de cada execução registrada pela classe **Trials**:

    ```python
   from hyperopt import Trials
   
   # Create a Trials object to track each run
   trial_runs = Trials()
   
   argmin = fmin(
     fn=objective,
     space=search_space,
     algo=algo,
     max_evals=3,
     trials=trial_runs)
   
   print("Best param values: ", argmin)
   
   # Get details from each trial run
   print ("trials:")
   for trial in trial_runs.trials:
       print ("\n", trial)
    ```

## Limpeza

No portal do Azure Databricks, na página **Computação**, selecione seu cluster e selecione **&#9632; Terminar** para encerrar o processo.

Se você terminou de explorar o Azure Databricks, exclua os recursos que criou para evitar custos desnecessários do Azure e liberar capacidade em sua assinatura.

> **Mais informações**: Para obter mais informações, confira [Ajuste de hiperparâmetros](https://learn.microsoft.com/azure/databricks/machine-learning/automl-hyperparam-tuning/) na documentação do Azure Databricks.