---
lab:
  title: Treinar um modelo de aprendizado profundo
---

# Treinar um modelo de aprendizado profundo

Neste exercício, você usará a biblioteca **PyTorch** para treinar um modelo de aprendizado profundo no Azure Databricks. Em seguida, você usará a biblioteca **Horovod** para distribuir o treinamento de aprendizado profundo entre vários nós de trabalho em um cluster.

Este exercício deve levar aproximadamente **45** minutos para ser concluído.

## Antes de começar

É necessário ter uma [assinatura do Azure](https://azure.microsoft.com/free) com acesso de nível administrativo.

## Provisionar um workspace do Azure Databricks

> **Dica**: Se você já tem um workspace do Azure Databricks, pode ignorar esse procedimento e usar o workspace existente.

Este exercício inclui um script para provisionar um novo workspace do Azure Databricks. O script tenta criar um recurso de workspace do Azure Databricks de camada *Premium* em uma região na qual sua assinatura do Azure tenha cota suficiente para os núcleos de computação necessários para este exercício; e pressupõe que sua conta de usuário tenha permissões suficientes na assinatura para criar um recurso de workspace do Azure Databricks. Se o script falhar devido a cota ou permissões insuficientes, você pode tentar [criar um workspace do Azure Databricks interativamente no portal do Azure](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace).

1. Em um navegador da web, faça logon no [portal do Azure](https://portal.azure.com) em `https://portal.azure.com`.
2. Use o botão **[\>_]** à direita da barra de pesquisa na parte superior da página para criar um Cloud Shell no portal do Azure, selecionando um ambiente ***PowerShell*** e criando um armazenamento caso solicitado. O Cloud Shell fornece uma interface de linha de comando em um painel na parte inferior do portal do Azure, conforme mostrado aqui:

    ![Portal do Azure com um painel do Cloud Shell](./images/cloud-shell.png)

    > **Observação**: se você tiver criado anteriormente um cloud shell que usa um ambiente *Bash*, use o menu suspenso no canto superior esquerdo do painel do cloud shell para alterá-lo para ***PowerShell***.

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
7. Aguarde a conclusão do script - isso normalmente leva cerca de 5 minutos, mas em alguns casos pode levar mais tempo. Enquanto você aguarda, revise o artigo [Treinamento distribuído](https://learn.microsoft.com/azure/databricks/machine-learning/train-model/distributed-training/) na documentação do Azure Databricks.

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
    - **Tipo de nó**: Standard_DS3_v2
    - **Encerra após** *20* **minutos de inatividade**

1. Aguarde a criação do cluster. Isso pode levar alguns minutos.

> **Observação**: se o cluster não for iniciado, sua assinatura pode ter cota insuficiente na região onde seu workspace do Azure Databricks está provisionado. Consulte [Limite de núcleo da CPU impede a criação do cluster](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit) para obter detalhes. Se isso acontecer, você pode tentar excluir seu workspace e criar um novo workspace em uma região diferente. Você pode especificar uma região como um parâmetro para o script de instalação da seguinte maneira: `./mslearn-databricks/setup.ps1 eastus`

## Criar um notebook

Você executará o código que usa a biblioteca MLLib do Spark para treinar um modelo de machine learning. Portanto, a primeira etapa é criar um novo notebook em seu workspace.

1. Na barra lateral, use o link **(+) Novo** para criar um **Notebook**.
1. Altere o nome padrão do notebook (**Notebook Sem Título *[data]***) para **Aprendizado Profundo**. Na lista suspensa **Conectar**, selecione o cluster, caso ainda não esteja selecionado. Se o cluster não executar, é porque ele pode levar cerca de um minuto para iniciar.

## Ingerir e preparar dados

O cenário deste exercício baseia-se em observações de pinguins na Antártida, com o objetivo de treinar um modelo de machine learning para prever a espécie de um pinguim observado, considerando sua localização e medidas corporais.

> **Citação**: O conjunto de dados sobre pinguins usado neste exercício é um subconjunto dos dados coletados e disponibilizados pela [Dra. Kristen Gorman](https://www.uaf.edu/cfos/people/faculty/detail/kristen-gorman.php) e pela [Estação Palmer, LTER Antártida](https://pal.lternet.edu/), membro da [Rede LTER (Rede de Pesquisa Ecológica de Longo Prazo)](https://lternet.edu/).

1. Na primeira célula do notebook, insira o código a seguir, que usa os comandos de *shell* para baixar os dados sobre pinguins do GitHub para o sistema de arquivos usado pelo cluster.

    ```bash
    %sh
    rm -r /dbfs/deepml_lab
    mkdir /dbfs/deepml_lab
    wget -O /dbfs/deepml_lab/penguins.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/penguins.csv
    ```

1. Use a opção de menu **&#9656; Executar Célula** à esquerda da célula para executá-la. Em seguida, aguarde o término do trabalho do Spark executado pelo código.
1. Agora, prepare os dados para o aprendizado de máquina. Na célula de código existente, use o ícone **+** para adicionar uma nova célula de código. Em seguida, insira o código a seguir na nova célula para:
    - Remover todas as linhas incompletas
    - Codificar o nome da ilha (cadeia de caracteres) como um inteiro
    - Aplicar tipos de dados apropriados
    - Normalizar os dados numéricos para uma escala semelhante
    - Divida os dados em dois conjuntos: um para treinamento e outro para testes.

    ```python
   from pyspark.sql.types import *
   from pyspark.sql.functions import *
   from sklearn.model_selection import train_test_split
   
   # Load the data, removing any incomplete rows
   df = spark.read.format("csv").option("header", "true").load("/deepml_lab/penguins.csv").dropna()
   
   # Encode the Island with a simple integer index
   # Scale FlipperLength and BodyMass so they're on a similar scale to the bill measurements
   islands = df.select(collect_set("Island").alias('Islands')).first()['Islands']
   island_indexes = [(islands[i], i) for i in range(0, len(islands))]
   df_indexes = spark.createDataFrame(island_indexes).toDF('Island', 'IslandIdx')
   data = df.join(df_indexes, ['Island'], 'left').select(col("IslandIdx"),
                      col("CulmenLength").astype("float"),
                      col("CulmenDepth").astype("float"),
                      (col("FlipperLength").astype("float")/10).alias("FlipperScaled"),
                       (col("BodyMass").astype("float")/100).alias("MassScaled"),
                      col("Species").astype("int")
                       )
   
   # Oversample the dataframe to triple its size
   # (Deep learning techniques like LOTS of data)
   for i in range(1,3):
       data = data.union(data)
   
   # Split the data into training and testing datasets   
   features = ['IslandIdx','CulmenLength','CulmenDepth','FlipperScaled','MassScaled']
   label = 'Species'
      
   # Split data 70%-30% into training set and test set
   x_train, x_test, y_train, y_test = train_test_split(data.toPandas()[features].values,
                                                       data.toPandas()[label].values,
                                                       test_size=0.30,
                                                       random_state=0)
   
   print ('Training Set: %d rows, Test Set: %d rows \n' % (len(x_train), len(x_test)))
    ```

## Instalar e importar as bibliotecas do PyTorch

O PyTorch é uma estrutura para criar modelos de machine learning, incluindo DNNs (redes neurais profundas). Como planejamos usar o PyTorch para criar nosso classificador de pinguins, precisaremos importar as bibliotecas do PyTorch que pretendemos usar. O PyTorch já está instalado em clusters do Azure Databricks com um runtime do ML Databricks (a instalação específica do PyTorch depende se o cluster tem GPUs (unidades de processamento gráfico) que podem ser usadas para processamento de alto desempenho via *cuda*).

1. Adicione uma nova célula de código e execute o seguinte código para se preparar para o uso do PyTorch:

    ```python
   import torch
   import torch.nn as nn
   import torch.utils.data as td
   import torch.nn.functional as F
   
   # Set random seed for reproducability
   torch.manual_seed(0)
   
   print("Libraries imported - ready to use PyTorch", torch.__version__)
    ```

## Criar carregadores de dados

O PyTorch usa *carregadores de dados* para carregar os dados de treinamento e validação em lotes. Já carregamos os dados em matrizes numpy, mas precisamos encapsulá-los em conjuntos de dados do PyTorch (nos quais os dados são convertidos em objetos do PyTorch do tipo *tensor*) e criar carregadores para ler lotes desses conjuntos de dados.

1. Adicione uma célula e execute o seguinte código para preparar os carregadores de dados:

    ```python
   # Create a dataset and loader for the training data and labels
   train_x = torch.Tensor(x_train).float()
   train_y = torch.Tensor(y_train).long()
   train_ds = td.TensorDataset(train_x,train_y)
   train_loader = td.DataLoader(train_ds, batch_size=20,
       shuffle=False, num_workers=1)

   # Create a dataset and loader for the test data and labels
   test_x = torch.Tensor(x_test).float()
   test_y = torch.Tensor(y_test).long()
   test_ds = td.TensorDataset(test_x,test_y)
   test_loader = td.DataLoader(test_ds, batch_size=20,
                                shuffle=False, num_workers=1)
   print('Ready to load data')
    ```

## Definir uma rede neural

Agora estamos prontos para definir nossa rede neural. Nesse caso, criaremos uma rede que consiste em três camadas totalmente conectadas:

- Uma camada de entrada que recebe um valor de entrada para cada característica (nesse caso, o índice de ilha e quatro medidas de pinguim) e gera 10 saídas.
- Uma camada oculta que recebe dez entradas da camada de entrada e envia dez saídas para a próxima camada.
- Uma camada de saída que gera um vetor de probabilidades para cada uma das três espécies possíveis de pinguins.

Conforme treinamos a rede passando dados por ela, a função de **forward** aplicará funções de ativação *RELU* às duas primeiras camadas (para limitar os resultados a números positivos) e retornará uma camada de saída final que usa uma função *log_softmax* para retornar um valor que representa uma pontuação de probabilidade para cada uma das três classes possíveis.

1. Execute o seguinte código para definir a rede neural:

    ```python
   # Number of hidden layer nodes
   hl = 10
   
   # Define the neural network
   class PenguinNet(nn.Module):
       def __init__(self):
           super(PenguinNet, self).__init__()
           self.fc1 = nn.Linear(len(features), hl)
           self.fc2 = nn.Linear(hl, hl)
           self.fc3 = nn.Linear(hl, 3)
   
       def forward(self, x):
           fc1_output = torch.relu(self.fc1(x))
           fc2_output = torch.relu(self.fc2(fc1_output))
           y = F.log_softmax(self.fc3(fc2_output).float(), dim=1)
           return y
   
   # Create a model instance from the network
   model = PenguinNet()
   print(model)
    ```

## Criar funções para treinar e testar um modelo de rede neural

Para treinar o modelo, precisamos encaminhar repetidamente os valores de treinamento pela rede, usar uma função de perda para calcular a perda, usar um otimizador para retropropagar os ajustes nos valores de pesos e viés e validar o modelo usando os dados de teste que retivemos.

1. Para fazer isso, use o código a seguir para criar uma função que treina e otimiza o modelo e uma função que testa o modelo.

    ```python
   def train(model, data_loader, optimizer):
       device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
       model.to(device)
       # Set the model to training mode
       model.train()
       train_loss = 0
       
       for batch, tensor in enumerate(data_loader):
           data, target = tensor
           #feedforward
           optimizer.zero_grad()
           out = model(data)
           loss = loss_criteria(out, target)
           train_loss += loss.item()
   
           # backpropagate adjustments to the weights
           loss.backward()
           optimizer.step()
   
       #Return average loss
       avg_loss = train_loss / (batch+1)
       print('Training set: Average loss: {:.6f}'.format(avg_loss))
       return avg_loss
              
               
   def test(model, data_loader):
       device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
       model.to(device)
       # Switch the model to evaluation mode (so we don't backpropagate)
       model.eval()
       test_loss = 0
       correct = 0
   
       with torch.no_grad():
           batch_count = 0
           for batch, tensor in enumerate(data_loader):
               batch_count += 1
               data, target = tensor
               # Get the predictions
               out = model(data)
   
               # calculate the loss
               test_loss += loss_criteria(out, target).item()
   
               # Calculate the accuracy
               _, predicted = torch.max(out.data, 1)
               correct += torch.sum(target==predicted).item()
               
       # Calculate the average loss and total accuracy for this epoch
       avg_loss = test_loss/batch_count
       print('Validation set: Average loss: {:.6f}, Accuracy: {}/{} ({:.0f}%)\n'.format(
           avg_loss, correct, len(data_loader.dataset),
           100. * correct / len(data_loader.dataset)))
       
       # return average loss for the epoch
       return avg_loss
    ```

## Treinar um modelo

Agora você pode usar as funções **train** e **test** para treinar um modelo de rede neural. Você treina redes neurais de forma interativa ao longo de várias *épocas*, registrando as estatísticas de perda e precisão para cada época.

1. Use o seguinte código para treinar o modelo:

    ```python
   # Specify the loss criteria (we'll use CrossEntropyLoss for multi-class classification)
   loss_criteria = nn.CrossEntropyLoss()
   
   # Use an optimizer to adjust weights and reduce loss
   learning_rate = 0.001
   optimizer = torch.optim.Adam(model.parameters(), lr=learning_rate)
   optimizer.zero_grad()
   
   # We'll track metrics for each epoch in these arrays
   epoch_nums = []
   training_loss = []
   validation_loss = []
   
   # Train over 100 epochs
   epochs = 100
   for epoch in range(1, epochs + 1):
   
       # print the epoch number
       print('Epoch: {}'.format(epoch))
       
       # Feed training data into the model
       train_loss = train(model, train_loader, optimizer)
       
       # Feed the test data into the model to check its performance
       test_loss = test(model, test_loader)
       
       # Log the metrics for this epoch
       epoch_nums.append(epoch)
       training_loss.append(train_loss)
       validation_loss.append(test_loss)
    ```

    Enquanto o processo de treinamento está em execução, vamos tentar entender o que está acontecendo:

    - Em cada *época*, o conjunto completo de dados de treinamento é encaminhado pela rede. Há cinco características para cada observação e cinco nós correspondentes na camada de entrada. Portanto, as características de cada observação são passadas como um vetor de cinco valores para essa camada. No entanto, por questões de eficiência, os vetores de características são agrupados em lotes, então, na verdade, uma matriz de vários vetores de características é alimentada de cada vez.
    - A matriz de valores das características é processada por uma função que executa uma soma ponderada usando valores de pesos e viés inicializados. O resultado dessa função é processado pela função de ativação da camada de entrada para limitar os valores passados para os nós na próxima camada.
    - A soma ponderada e as funções de ativação são repetidas em cada camada. Observe que as funções operam em vetores e matrizes em vez de valores escalares individuais. Em outras palavras, a passagem direta é essencialmente uma série de funções de álgebra linear aninhadas. Esse é o motivo pelo qual os cientistas de dados preferem usar computadores com GPUs (unidades de processamento gráfico), pois elas são otimizadas para cálculos de matriz e vetor.
    - Na camada final da rede, os vetores de saída contêm um valor calculado para cada classe possível (nesse caso, as classes 0, 1 e 2). Esse vetor é processado por uma *função de perda* que determina o quão distante eles estão dos valores esperados com base nas classes reais. Então, por exemplo, suponha que a saída para uma observação de Pinguim-gentoo (classe 1) seja \[0,3; 0,4; 0,3\]. A previsão correta seria \[0,0; 1,0; 0,0\]. Portanto, a variância entre os valores previstos e reais (o quão distante cada valor previsto está do que deveria ser) é \[0,3; 0,6; 0,3\]. Essa variância é agregada para cada lote e mantida como um agregado contínuo para calcular o nível geral de erro (*perda*) incorrido pelos dados de treinamento durante a época.
    - No final de cada época, os dados de validação são encaminhados pela rede, e sua perda e precisão (proporção de previsões corretas com base no valor de probabilidade mais alto no vetor de saída) também são calculadas. É útil fazer isso porque nos permite comparar o desempenho do modelo após cada época usando dados nos quais ele não foi treinado, ajudando-nos a determinar se ele generalizará bem para novos dados ou se ele está *sobreajustado* aos dados de treinamento.
    - Depois que todos os dados forem encaminhados pela rede, a saída da função de perda para os dados de *treinamento* (mas <u>não</u> os dados de *validação*) será passada para o otimizador. Os detalhes precisos de como o otimizador processa a perda variam dependendo do algoritmo de otimização específico que está sendo usado. Mas, fundamentalmente, você pode pensar em toda a rede, desde a camada de entrada até a função de perda, como sendo uma grande função aninhada (*composta*). O otimizador aplica cálculo diferencial para calcular as *derivadas parciais* da função em relação a cada valor de peso e viés que foi usado na rede. É possível fazer isso com eficiência para uma função aninhada devido a algo chamado de *regra da cadeia*, que permite determinar a derivada de uma função composta das derivadas de suas funções internas e externas. Você realmente não precisa se preocupar com os detalhes da matemática aqui (o otimizador faz isso para você), mas o resultado final é que as derivadas parciais nos informam sobre a inclinação (ou *gradiente*) da função de perda em relação a cada valor de peso e viés. Em outras palavras, podemos determinar se devemos aumentar ou diminuir os valores de peso e viés para minimizar a perda.
    - Tendo determinado em qual direção ajustar os pesos e vieses, o otimizador usa a *taxa de aprendizado* para determinar o quanto ajustá-los, e depois trabalha retroativamente por meio da rede em um processo chamado *retropropagação* para atribuir novos valores aos pesos e vieses em cada camada.
    - A próxima época repete todo o processo de treinamento, validação e retropropagação, começando com os pesos e vieses revisados da época anterior, o que, idealmente, resultará em um nível de perda mais baixo.
    - O processo continua assim por 100 épocas.

## Revisar a perda no treinamento e na validação

Após a conclusão do treinamento, podemos examinar as métricas de perda que registramos durante o treinamento e a validação do modelo. Estamos procurando duas coisas:

- A perda deve diminuir a cada época, mostrando que o modelo está aprendendo os pesos e vieses corretos para prever os rótulos corretos.
- A perda no treinamento e a perda na validação devem seguir uma tendência semelhante, mostrando que o modelo não está sofrendo de sobreajuste aos dados de treinamento.

1. Use o seguinte código para criar um gráfico da perda:

    ```python
   %matplotlib inline
   from matplotlib import pyplot as plt
   
   plt.plot(epoch_nums, training_loss)
   plt.plot(epoch_nums, validation_loss)
   plt.xlabel('epoch')
   plt.ylabel('loss')
   plt.legend(['training', 'validation'], loc='upper right')
   plt.show()
    ```

## Exibir os pesos e vieses aprendidos

O modelo treinado consiste nos pesos e vieses finais que foram determinados pelo otimizador durante o treinamento. Com base em nosso modelo de rede, devemos esperar os seguintes valores para cada camada:

- Camada 1 (*fc1*): Há cinco valores de entrada indo para dez nós de saída, portanto, deve haver 10 x 5 pesos e 10 valores de viés.
- Camada 2 (*fc2*): Há dez valores de entrada indo para dez nós de saída, portanto, deve haver 10 x 10 pesos e 10 valores de viés.
- Camada 3 (*fc3*): Há dez valores de entrada indo para três nós de saída, portanto, deve haver 3 x 10 pesos e três valores de viés.

1. Use o seguinte código para exibir as camadas em seu modelo treinado:

    ```python
   for param_tensor in model.state_dict():
       print(param_tensor, "\n", model.state_dict()[param_tensor].numpy())
    ```

## Salvar e usar o modelo treinado

Agora que temos um modelo treinado, podemos salvar seus pesos treinados para uso posterior.

1. Use o seguinte código para salvar o modelo:

    ```python
   # Save the model weights
   model_file = '/dbfs/penguin_classifier.pt'
   torch.save(model.state_dict(), model_file)
   del model
   print('model saved as', model_file)
    ```

1. Use o código a seguir para carregar os pesos do modelo e fazer previsões da espécie para uma nova observação de pinguim:

    ```python
   # New penguin features
   x_new = [[1, 50.4,15.3,20,50]]
   print ('New sample: {}'.format(x_new))
   
   # Create a new model class and load weights
   model = PenguinNet()
   model.load_state_dict(torch.load(model_file))
   
   # Set model to evaluation mode
   model.eval()
   
   # Get a prediction for the new data sample
   x = torch.Tensor(x_new).float()
   _, predicted = torch.max(model(x).data, 1)
   
   print('Prediction:',predicted.item())
    ```

## Limpar

No portal do Azure Databricks, na página **Computação**, selecione seu cluster e selecione **&#9632; Terminar** para encerrar o processo.

Se você tiver terminado de explorar o Azure Databricks, poderá excluir os recursos que criou para evitar custos desnecessários do Azure e liberar capacidade em sua assinatura.
