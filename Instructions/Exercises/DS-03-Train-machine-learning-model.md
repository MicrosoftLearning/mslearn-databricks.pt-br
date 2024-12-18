---
lab:
  title: Introdução ao machine learning no Azure Databricks
---

# Introdução ao machine learning no Azure Databricks

Neste exercício, você explorará as técnicas para preparar dados e treinar modelos de machine learning no Azure Databricks.

Este exercício deve levar aproximadamente **45** minutos para ser concluído.

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
7. Aguarde a conclusão do script - isso normalmente leva cerca de 5 minutos, mas em alguns casos pode levar mais tempo. Enquanto estiver esperando, leia o artigo [O que é Machine Learning do Databricks?](https://learn.microsoft.com/azure/databricks/machine-learning/) na documentação do Azure Databricks.

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
1. Altere o nome padrão do notebook (**Notebook Sem Título *[data]***) para **Machine Learning**. Na lista suspensa **Conectar**, selecione o cluster, caso ainda não esteja selecionado. Se o cluster não executar, é porque ele pode levar cerca de um minuto para iniciar.

## Ingerir dados

O cenário deste exercício baseia-se em observações de pinguins na Antártida, com o objetivo de treinar um modelo de machine learning para prever a espécie de um pinguim observado, considerando sua localização e medidas corporais.

> **Citação**: O conjunto de dados sobre pinguins usado neste exercício é um subconjunto dos dados coletados e disponibilizados pela [Dra. Kristen Gorman](https://www.uaf.edu/cfos/people/faculty/detail/kristen-gorman.php) e pela [Estação Palmer, LTER Antártida](https://pal.lternet.edu/), membro da [Rede LTER (Rede de Pesquisa Ecológica de Longo Prazo)](https://lternet.edu/).

1. Na primeira célula do notebook, insira o código a seguir, que usa os comandos de *shell* para baixar os dados sobre pinguins do GitHub para o sistema de arquivos usado pelo cluster.

    ```bash
    %sh
    rm -r /dbfs/ml_lab
    mkdir /dbfs/ml_lab
    wget -O /dbfs/ml_lab/penguins.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/penguins.csv
    ```

1. Use a opção de menu **&#9656; Executar Célula** à esquerda da célula para executá-la. Em seguida, aguarde o término do trabalho do Spark executado pelo código.

## Explorar e limpar os dados
  
Agora que você ingeriu o arquivo de dados, você pode carregá-lo em um dataframe e exibi-lo.

1. Na célula de código existente, use o ícone **+** para adicionar uma nova célula de código. Então, na nova célula, insira e execute o código a seguir para carregar os dados dos arquivos e exibi-los.

    ```python
   df = spark.read.format("csv").option("header", "true").load("/ml_lab/penguins.csv")
   display(df)
    ```

    O código inicia os *Trabalhos do Spark* necessários para carregar os dados e a saída é um objeto *pyspark.sql.dataframe.DataFrame* chamado *df*. Você verá essas informações exibidas diretamente abaixo do código e poderá usar a alternância **&#9656;** para expandir a saída **df: pyspark.sql.dataframe.DataFrame** e ver detalhes das colunas que ela contém e seus tipos de dados. Como esses dados foram carregados de um arquivo de texto e continham alguns valores em branco, o Spark atribuiu o tipo de dados **cadeia de caracteres** a todas as colunas.
    
    Os dados em si consistem em medidas das seguintes informações dos pinguins que foram observados na Antártida:
    
    - **Island**: A ilha na Antártida onde o pinguim foi observado.
    - **CulmenLength**: O comprimento em mm do cúlmen do pinguim (bico).
    - **CulmenDepth**: A profundidade em mm do cúlmen do pinguim.
    - **FlipperLength**: O comprimento em mm da nadadeira do pinguim.
    - **BodyMass**: A massa corporal do pinguim em gramas.
    - **Espécie**: Um valor inteiro que representa a espécie do pinguim:
      - **0**: *Adélia*
      - **1**: *Gentoo*
      - **2**: *Chinstrap*
      
    Nosso objetivo neste projeto é usar as características observadas de um pinguim (suas *características*) para prever sua espécie (que, na terminologia do aprendizado de máquina, chamamos de *etiqueta*).
      
    Algumas observações contêm valores de dados *nulos* ou "ausentes" para algumas características. Não é incomum que os dados brutos de origem ingeridos tenham problemas como esse, portanto, normalmente, a primeira fase em um projeto de aprendizado de máquina é explorar os dados de forma completa e limpá-los para torná-los mais adequados para treinar um modelo de machine learning.
    
1. Adicione uma célula e use-a para executar a célula a seguir para remover as linhas com dados incompletos usando o método **dropna** e aplicar tipos de dados apropriados aos dados usando o método **select** com as funções **col** e **astype**.

    ```python
   from pyspark.sql.types import *
   from pyspark.sql.functions import *
   
   data = df.dropna().select(col("Island").astype("string"),
                              col("CulmenLength").astype("float"),
                             col("CulmenDepth").astype("float"),
                             col("FlipperLength").astype("float"),
                             col("BodyMass").astype("float"),
                             col("Species").astype("int")
                             )
   display(data)
    ```
    
    Mais uma vez, você pode alternar os detalhes do dataframe retornado (desta vez denominado *dados*) para verificar se os tipos de dados foram aplicados e você pode examinar os dados para verificar se as linhas que contêm dados incompletos foram removidas.
    
    Em um projeto real, você provavelmente precisaria executar mais exploração e limpeza de dados para corrigir (ou remover) erros nos dados, identificar e remover exceções (valores pequenos ou grandes atípicos) ou para equilibrar os dados para que haja um número razoavelmente igual de linhas para cada etiqueta que você está tentando prever.

    > **Dica**: Você pode saber mais sobre métodos e funções que pode usar com dataframes na [referência do Spark SQL](https://spark.apache.org/docs/latest/sql-programming-guide.html).

## Dividir os dados

Para fins deste exercício, vamos supor que os dados agora estão limpos e prontos para usarmos para treinar um modelo de machine learning. A etiqueta que vamos tentar prever é uma categoria específica ou *classe* (a espécie de um pinguim), portanto, o tipo de modelo de machine learning que precisamos treinar é um modelo de *classificação*. A classificação (juntamente com a *regressão*, que é usada para prever um valor numérico) é um formulário ou aprendizado de máquina *supervisionado* no qual usamos dados de treinamento que incluem valores conhecidos para a etiqueta que queremos prever. O processo de treinamento de um modelo é apenas ajustar um algoritmo aos dados para calcular como os valores de características se correlacionam com o valor da etiqueta conhecida. Em seguida, podemos aplicar o modelo treinado a uma nova observação para a qual sabemos apenas os valores de característica e fazer com que ele preveja o valor da etiqueta.

Para garantir que possamos confiar em nosso modelo treinado, a abordagem típica é treinar o modelo com apenas *alguns* dos dados e reter alguns dados com valores de etiqueta conhecidos que podemos usar para testar o modelo treinado e ver o nível de precisão das suas previsões. Para atingir esse objetivo, dividiremos o conjunto de dados completo em dois subconjuntos aleatórios. Usaremos 70% dos dados para treinamento e reteremos 30% para testes.

1. Adicione e execute uma célula de código com o código a seguir para separar os dados.

    ```python
   splits = data.randomSplit([0.7, 0.3])
   train = splits[0]
   test = splits[1]
   print ("Training Rows:", train.count(), " Testing Rows:", test.count())
    ```

## Executar a engenharia de recursos

Depois de limpar os dados brutos, os cientistas de dados normalmente executam algum trabalho adicional para prepará-los para o treinamento de modelo. Esse processo é comumente conhecido como *engenharia de recursos* e envolve otimizar iterativamente os recursos no conjunto de dados de treinamento para produzir o melhor modelo possível. As modificações de recursos específicos necessárias dependem dos dados e do modelo desejado, mas há algumas tarefas comuns de engenharia de recursos com as quais você deve se familiarizar.

### Codificar recursos categóricos

Os algoritmos de aprendizado de máquina geralmente são baseados na localização de relações matemáticas entre recursos e etiquetas. Isso significa que geralmente é melhor definir os recursos em seus dados de treinamento como valores *numéricos*. Em alguns casos, você pode ter alguns recursos que são *categóricos* em vez de numéricos, e que são expressas como cadeias de caracteres como, por exemplo, o nome da ilha onde a observação do pinguim ocorreu em nosso conjunto de dados. No entanto, a maioria dos algoritmos espera recursos numéricos, portanto, esses valores categóricos baseados em cadeia de caracteres precisam ser *codificados* como números. Nesse caso, usaremos um **StringIndexer** da biblioteca **MLLib do Spark** para codificar o nome da ilha como um valor numérico atribuindo um índice inteiro exclusivo para cada nome de ilha discreto.

1. Execute o código a seguir para codificar os valores de coluna categórica **Ilha** como índices numéricos.

    ```python
   from pyspark.ml.feature import StringIndexer

   indexer = StringIndexer(inputCol="Island", outputCol="IslandIdx")
   indexedData = indexer.fit(train).transform(train).drop("Island")
   display(indexedData)
    ```

    Nos resultados, você deve ver que, em vez de um nome de ilha, cada linha agora tem uma coluna **IslandIdx** com um valor inteiro representando a ilha na qual a observação foi registrada.

### Normalizar (dimensionar) recursos numéricos

Agora, vamos voltar nossa atenção para os valores numéricos em nossos dados. Esses valores (**CulmenLength**, **CulmenDepth**, **FlipperLength** e **BodyMass**) representam medidas de um tipo ou de outro, mas estão em escalas diferentes. Ao treinar um modelo, as unidades de medida não são tão importantes quanto as diferenças relativas entre diferentes observações e os características representadas por números maiores geralmente podem dominar o algoritmo de treinamento do modelo, distorcendo a importância do recurso ao calcular uma previsão. Para atenuar isso, é comum *normalizar* valores de recursos numéricos para que estejam todos na mesma escala relativa (por exemplo, um valor decimal entre 0,0 e 1,0).

O código que usaremos para fazer isso é um pouco mais envolvido do que a codificação categórica que fizemos anteriormente. Precisamos dimensionar vários valores de coluna ao mesmo tempo, portanto, a técnica que usamos é criar uma única coluna contendo um *vetor* (essencialmente uma matriz) de todos os recursos numéricos e aplicar um dimensionador para produzir uma nova coluna de vetor com os valores normalizados equivalentes.

1. Use o código a seguir para normalizar os recursos numéricos e ver uma comparação das colunas de vetor pré-normalizadas e normalizadas.

    ```python
   from pyspark.ml.feature import VectorAssembler, MinMaxScaler

   # Create a vector column containing all numeric features
   numericFeatures = ["CulmenLength", "CulmenDepth", "FlipperLength", "BodyMass"]
   numericColVector = VectorAssembler(inputCols=numericFeatures, outputCol="numericFeatures")
   vectorizedData = numericColVector.transform(indexedData)
   
   # Use a MinMax scaler to normalize the numeric values in the vector
   minMax = MinMaxScaler(inputCol = numericColVector.getOutputCol(), outputCol="normalizedFeatures")
   scaledData = minMax.fit(vectorizedData).transform(vectorizedData)
   
   # Display the data with numeric feature vectors (before and after scaling)
   compareNumerics = scaledData.select("numericFeatures", "normalizedFeatures")
   display(compareNumerics)
    ```

    A coluna **numericFeatures** nos resultados contém um vetor para cada linha. O vetor inclui quatro valores numéricos não dimensionados (as medidas originais do pinguim). Você pode usar a alternância **&#9656;** para ver os valores discretos com mais clareza.
    
    A coluna **normalizedFeatures** também contém um vetor para cada observação de pinguim, mas desta vez os valores no vetor são normalizados em uma escala relativa com base nos valores mínimo e máximo de cada medida.

### Preparar os recursos e etiquetas para treinamento

Agora, vamos reunir tudo e criar uma única coluna com todas as características (o nome da ilha categórica codificada e as medidas normalizadas do pinguim), e outra coluna contendo a etiqueta de classe que queremos treinar um modelo para prever (a espécie de pinguim).

1. Execute o código a seguir:

    ```python
   featVect = VectorAssembler(inputCols=["IslandIdx", "normalizedFeatures"], outputCol="featuresVector")
   preppedData = featVect.transform(scaledData)[col("featuresVector").alias("features"), col("Species").alias("label")]
   display(preppedData)
    ```

    O vetor **características** contém cinco valores (a ilha codificada e o comprimento normalizado do cúlmen, a profundidade do cúlmen, o comprimento da nadadeira e a massa corporal). A etiqueta contém um código inteiro simples que indica a classe de espécies de pinguins.

## Treinar um modelo de machine learning

Agora que os dados de treinamento estão preparados, você pode usá-los para treinar um modelo. Os modelos são treinados usando um *algoritmo* que tenta estabelecer uma relação entre os recursos e etiquetas. Como, nesse caso, você deseja treinar um modelo que preveja uma categoria de *classe*, você precisa usar um algoritmo de *classificação*. Há muitos algoritmos para classificação. Vamos começar com um bem estabelecido: regressão logística, que tenta iterativamente encontrar os coeficientes ideais que podem ser aplicados aos dados de recursos em um cálculo logístico que prevê a probabilidade de cada valor de etiqueta de classe. Para treinar o modelo, você ajustará o algoritmo de regressão logística aos dados de treinamento.

1. Execute o seguinte código para treinar um modelo.

    ```python
   from pyspark.ml.classification import LogisticRegression

   lr = LogisticRegression(labelCol="label", featuresCol="features", maxIter=10, regParam=0.3)
   model = lr.fit(preppedData)
   print ("Model trained!")
    ```

    A maioria dos algoritmos dá suporte a parâmetros que fornecem algum controle sobre a maneira como o modelo é treinado. Nesse caso, o algoritmo de regressão logística exige que você identifique a coluna que contém o vetor de características e a coluna que contém a etiqueta conhecida. Ele também permite especificar o número máximo de iterações executadas para encontrar coeficientes ideais para o cálculo logístico e um parâmetro de regularização usado para impedir que o modelo faça *sobreajuste* (em outras palavras, estabelecer um cálculo logístico que funcione bem com os dados de treinamento, mas que não generalize bem quando aplicado a novos dados).

## Testar o modelo

Agora que você tem um modelo treinado, pode testá-lo com os dados retidos. Antes de fazer isso, você precisa executar as mesmas transformações de engenharia de recursos para os dados de teste que aplicou aos dados de treinamento (nesse caso, codificar o nome da ilha e normalizar as medidas). Em seguida, você pode usar o modelo para prever etiquetas de características nos dados de teste e comparar as etiquetas previstas com as etiquetas conhecidas reais.

1. Use o código a seguir para preparar os dados de teste e, em seguida, gerar previsões:

    ```python
   # Prepare the test data
   indexedTestData = indexer.fit(test).transform(test).drop("Island")
   vectorizedTestData = numericColVector.transform(indexedTestData)
   scaledTestData = minMax.fit(vectorizedTestData).transform(vectorizedTestData)
   preppedTestData = featVect.transform(scaledTestData)[col("featuresVector").alias("features"), col("Species").alias("label")]
   
   # Get predictions
   prediction = model.transform(preppedTestData)
   predicted = prediction.select("features", "probability", col("prediction").astype("Int"), col("label").alias("trueLabel"))
   display(predicted)
    ```

    Os resultados incluem as seguintes colunas:
    
    - **características**: Os dados de características preparados do conjunto de dados de teste.
    - **probabilidade**: A probabilidade calculada pelo modelo para cada classe. Isso consiste em um vetor que contém três valores de probabilidade (porque há três classes) que somam um total de 1,0 (presume-se que haja uma probabilidade de 100% de que o pinguim pertença a *uma* das três classes de espécie).
    - **previsão**: A etiqueta de classe prevista (aquela com a maior probabilidade).
    - **trueLabel**: O valor real do rótulo conhecido dos dados de teste.
    
    Para avaliar a eficácia do modelo, você pode simplesmente comparar as etiquetas previstas e as reais nesses resultados. No entanto, você pode obter métricas mais significativas usando um avaliador de modelo. Nesse caso, um avaliador de classificação multiclasse (porque há várias etiquetas de classe possíveis).

1. Use o código a seguir para obter métricas de avaliação de um modelo de classificação com base nos resultados dos dados de teste:

    ```python
   from pyspark.ml.evaluation import MulticlassClassificationEvaluator
   
   evaluator = MulticlassClassificationEvaluator(labelCol="label", predictionCol="prediction")
   
   # Simple accuracy
   accuracy = evaluator.evaluate(prediction, {evaluator.metricName:"accuracy"})
   print("Accuracy:", accuracy)
   
   # Individual class metrics
   labels = [0,1,2]
   print("\nIndividual class metrics:")
   for label in sorted(labels):
       print ("Class %s" % (label))
   
       # Precision
       precision = evaluator.evaluate(prediction, {evaluator.metricLabel:label,
                                                   evaluator.metricName:"precisionByLabel"})
       print("\tPrecision:", precision)
   
       # Recall
       recall = evaluator.evaluate(prediction, {evaluator.metricLabel:label,
                                                evaluator.metricName:"recallByLabel"})
       print("\tRecall:", recall)
   
       # F1 score
       f1 = evaluator.evaluate(prediction, {evaluator.metricLabel:label,
                                            evaluator.metricName:"fMeasureByLabel"})
       print("\tF1 Score:", f1)
   
   # Weighted (overall) metrics
   overallPrecision = evaluator.evaluate(prediction, {evaluator.metricName:"weightedPrecision"})
   print("Overall Precision:", overallPrecision)
   overallRecall = evaluator.evaluate(prediction, {evaluator.metricName:"weightedRecall"})
   print("Overall Recall:", overallRecall)
   overallF1 = evaluator.evaluate(prediction, {evaluator.metricName:"weightedFMeasure"})
   print("Overall F1 Score:", overallF1)
    ```

    As métricas de avaliação calculadas para classificação multiclasse incluem:
    
    - **Precisão**: A proporção de previsões gerais que estavam corretas.
    - Métricas por classe:
      - **Precisão**: A proporção de previsões dessa classe que estavam corretas.
      - **Recall**: A proporção de instâncias reais dessa classe que foram previstas corretamente.
      - **Medida F**: Uma métrica combinada para precisão e recall
    - Precisão combinada (ponderada), recall e métricas F1 para todas as classes.
    
    > **Observação**: Inicialmente, pode parecer que a métrica de precisão geral fornece a melhor maneira de avaliar o desempenho preditivo de um modelo. No entanto, considere o seguinte. Suponha que pinguins gentoo compensem 95% da população de pinguins no local do seu estudo. Um modelo que sempre prevê a etiqueta **1** (a classe de Gentoo) terá uma precisão de 0,95. Isso não significa que seja um bom modelo para prever uma espécie de pinguim com base nas características! É por isso que os cientistas de dados tendem a explorar métricas adicionais para entender melhor o quão bem um modelo de classificação prevê para cada etiqueta de classe possível.

## Use um pipeline

Você treinou seu modelo executando as etapas necessárias de engenharia de recursos e, em seguida, ajustando um algoritmo aos dados. Para usar o modelo com alguns dados de teste para gerar previsões (conhecidas como *inferência*), você precisou aplicar as mesmas etapas de engenharia de recursos aos dados de teste. Uma maneira mais eficiente de criar e usar modelos é encapsular os transformadores usados para preparar os dados e o modelo usado para treiná-los em um *pipeline*.

1. Use o código a seguir para criar um pipeline que encapsula as etapas de preparação de dados e treinamento de modelo:

    ```python
   from pyspark.ml import Pipeline
   from pyspark.ml.feature import StringIndexer, VectorAssembler, MinMaxScaler
   from pyspark.ml.classification import LogisticRegression
   
   catFeature = "Island"
   numFeatures = ["CulmenLength", "CulmenDepth", "FlipperLength", "BodyMass"]
   
   # Define the feature engineering and model training algorithm steps
   catIndexer = StringIndexer(inputCol=catFeature, outputCol=catFeature + "Idx")
   numVector = VectorAssembler(inputCols=numFeatures, outputCol="numericFeatures")
   numScaler = MinMaxScaler(inputCol = numVector.getOutputCol(), outputCol="normalizedFeatures")
   featureVector = VectorAssembler(inputCols=["IslandIdx", "normalizedFeatures"], outputCol="Features")
   algo = LogisticRegression(labelCol="Species", featuresCol="Features", maxIter=10, regParam=0.3)
   
   # Chain the steps as stages in a pipeline
   pipeline = Pipeline(stages=[catIndexer, numVector, numScaler, featureVector, algo])
   
   # Use the pipeline to prepare data and fit the model algorithm
   model = pipeline.fit(train)
   print ("Model trained!")
    ```

    Como as etapas de engenharia de recursos agora estão encapsuladas no modelo treinado pelo pipeline, você pode usar o modelo com os dados de teste sem precisar aplicar cada transformação (elas serão aplicadas automaticamente pelo modelo).

1. Use o seguinte código para aplicar o pipeline aos dados de teste:

    ```python
   prediction = model.transform(test)
   predicted = prediction.select("Features", "probability", col("prediction").astype("Int"), col("Species").alias("trueLabel"))
   display(predicted)
    ```

## Experimente outro algoritmo

Até agora, você treinou um modelo de classificação usando o algoritmo de regressão logística. Vamos alterar esse estágio no pipeline para experimentar um algoritmo diferente.

1. Execute o seguinte código para criar um pipeline que usa um algoritmo de árvore de decisão:

    ```python
   from pyspark.ml import Pipeline
   from pyspark.ml.feature import StringIndexer, VectorAssembler, MinMaxScaler
   from pyspark.ml.classification import DecisionTreeClassifier
   
   catFeature = "Island"
   numFeatures = ["CulmenLength", "CulmenDepth", "FlipperLength", "BodyMass"]
   
   # Define the feature engineering and model steps
   catIndexer = StringIndexer(inputCol=catFeature, outputCol=catFeature + "Idx")
   numVector = VectorAssembler(inputCols=numFeatures, outputCol="numericFeatures")
   numScaler = MinMaxScaler(inputCol = numVector.getOutputCol(), outputCol="normalizedFeatures")
   featureVector = VectorAssembler(inputCols=["IslandIdx", "normalizedFeatures"], outputCol="Features")
   algo = DecisionTreeClassifier(labelCol="Species", featuresCol="Features", maxDepth=10)
   
   # Chain the steps as stages in a pipeline
   pipeline = Pipeline(stages=[catIndexer, numVector, numScaler, featureVector, algo])
   
   # Use the pipeline to prepare data and fit the model algorithm
   model = pipeline.fit(train)
   print ("Model trained!")
    ```

    Desta vez, o pipeline inclui os mesmos estágios de preparação de recursos de antes, mas usa um algoritmo de *árvore de decisão* para treinar o modelo.
    
   1. Execute o seguinte código para usar o novo pipeline com os dados de teste:

    ```python
   # Get predictions
   prediction = model.transform(test)
   predicted = prediction.select("Features", "probability", col("prediction").astype("Int"), col("Species").alias("trueLabel"))
   
   # Generate evaluation metrics
   from pyspark.ml.evaluation import MulticlassClassificationEvaluator
   
   evaluator = MulticlassClassificationEvaluator(labelCol="Species", predictionCol="prediction")
   
   # Simple accuracy
   accuracy = evaluator.evaluate(prediction, {evaluator.metricName:"accuracy"})
   print("Accuracy:", accuracy)
   
   # Class metrics
   labels = [0,1,2]
   print("\nIndividual class metrics:")
   for label in sorted(labels):
       print ("Class %s" % (label))
   
       # Precision
       precision = evaluator.evaluate(prediction, {evaluator.metricLabel:label,
                                                       evaluator.metricName:"precisionByLabel"})
       print("\tPrecision:", precision)
   
       # Recall
       recall = evaluator.evaluate(prediction, {evaluator.metricLabel:label,
                                                evaluator.metricName:"recallByLabel"})
       print("\tRecall:", recall)
   
       # F1 score
       f1 = evaluator.evaluate(prediction, {evaluator.metricLabel:label,
                                            evaluator.metricName:"fMeasureByLabel"})
       print("\tF1 Score:", f1)
   
   # Weighed (overall) metrics
   overallPrecision = evaluator.evaluate(prediction, {evaluator.metricName:"weightedPrecision"})
   print("Overall Precision:", overallPrecision)
   overallRecall = evaluator.evaluate(prediction, {evaluator.metricName:"weightedRecall"})
   print("Overall Recall:", overallRecall)
   overallF1 = evaluator.evaluate(prediction, {evaluator.metricName:"weightedFMeasure"})
   print("Overall F1 Score:", overallF1)
    ```

## Salvar o modelo

Na realidade, você tentaria iterativamente treinar o modelo com algoritmos (e parâmetros) diferentes para encontrar o melhor modelo para seus dados. Por enquanto, manteremos o modelo de árvores de decisão que treinamos. Vamos salvá-lo para que possamos usá-lo mais tarde com novas observações de pinguins.

1. Use o seguinte código para salvar o modelo:

    ```python
   model.save("/models/penguin.model")
    ```

    Agora, quando você viu um novo pinguim, você pode carregar o modelo salvo e usá-lo para prever a espécie do pinguim com base nas medidas de suas características. Usar um modelo para gerar previsões de novos dados é chamado *inferência*.

1. Execute o código a seguir para carregar o modelo usá-lo para fazer previsões da espécie de uma nova observação de pinguim:

    ```python
   from pyspark.ml.pipeline import PipelineModel

   persistedModel = PipelineModel.load("/models/penguin.model")
   
   newData = spark.createDataFrame ([{"Island": "Biscoe",
                                     "CulmenLength": 47.6,
                                     "CulmenDepth": 14.5,
                                     "FlipperLength": 215,
                                     "BodyMass": 5400}])
   
   
   predictions = persistedModel.transform(newData)
   display(predictions.select("Island", "CulmenDepth", "CulmenLength", "FlipperLength", "BodyMass", col("prediction").alias("PredictedSpecies")))
    ```

## Limpeza

No portal do Azure Databricks, na página **Computação**, selecione seu cluster e selecione **&#9632; Terminar** para encerrar o processo.

Se você tiver terminado de explorar o Azure Databricks, poderá excluir os recursos que criou para evitar custos desnecessários do Azure e liberar capacidade em sua assinatura.

> **Mais informações**: Para obter mais informações, confira a [documentação da MLLib do Spark](https://spark.apache.org/docs/latest/ml-guide.html).
