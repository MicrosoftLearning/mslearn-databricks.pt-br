---
lab:
  title: Treinar um modelo com AutoML
---

# Treinar um modelo com AutoML

O AutoML é um recurso do Azure Databricks que experimenta vários algoritmos e parâmetros com seus dados para treinar um modelo de machine learning ideal.

Este exercício deve levar aproximadamente **30** minutos para ser concluído.

> **Observação**: a interface do usuário do Azure Databricks está sujeita a melhorias contínuas. A interface do usuário pode ter sido alterada desde que as instruções neste exercício foram escritas.

## Antes de começar

É necessário ter uma [assinatura do Azure](https://azure.microsoft.com/free) com acesso de nível administrativo.

## Provisionar um workspace do Azure Databricks

> **Observação**: Para este exercício, você precisa de um workspace do Azure Databricks **Premium** em uma região em que a *veiculação de modelos* seja suportada. Consulte as [regiões do Azure Databricks](https://learn.microsoft.com/azure/databricks/resources/supported-regions) para obter detalhes sobre os recursos regionais do Azure Databricks. Se você já possui um workspace do Azure Databricks *Premium* ou de *Avaliação* em uma região adequada, poderá ignorar esse procedimento e usar o workspace existente.

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
7. Aguarde a conclusão do script - isso normalmente leva cerca de 5 minutos, mas em alguns casos pode levar mais tempo. Enquanto estiver esperando, leia o artigo [O que é AutoML?](https://learn.microsoft.com/azure/databricks/machine-learning/automl/) na documentação do Azure Databricks.

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

## Fazer upload de dados de treinamento para um SQL Warehouse

Para treinar um modelo de machine learning usando o AutoML, você precisa carregar os dados de treinamento. Neste exercício, você treinará um modelo para classificar um pinguim como uma das três espécies com base em observações, incluindo sua localização e medidas corporais. Você carregará dados de treinamento que incluem o rótulo de espécie em uma tabela em um data warehouse do Azure Databricks.

1. No portal do Azure Databricks para seu workspace, na barra lateral, em **SQL**, selecione **SQL Warehouses**.
1. Observe que o workspace já inclui um SQL Warehouse chamado **Warehouse Inicial**.
1. No menu **Ações** (**⁝**) do SQL Warehouse, selecione **Editar**. Em seguida, defina a propriedade **Tamanho do cluster** como **2X-Small** e salve as alterações.
1. Use o botão **Iniciar** para iniciar o SQL Warehouse (o que pode levar um ou dois minutos).

> **Observação**: se o SQL Warehouse não for iniciado, sua assinatura pode ter cota insuficiente na região em que seu workspace do Azure Databricks está provisionado. Para obter detalhes, confira [Cota de vCPU do Azure necessária](https://docs.microsoft.com/azure/databricks/sql/admin/sql-endpoints#required-azure-vcpu-quota). Se isso acontecer, você pode tentar solicitar um aumento de cota, conforme detalhado na mensagem de erro, quando o depósito falhar ao iniciar. Como alternativa, tente excluir seu workspace e criar um novo em uma região diferente. Você pode especificar uma região como um parâmetro para o script de instalação da seguinte maneira: `./mslearn-databricks/setup.ps1 eastus`

1. Faça o download do arquivo [**penguins.csv**](https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/penguins.csv) do `https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/penguins.csv` para seu computador local, salvando-o como **penguins.csv**.
1. No portal do workspace do Azure Databricks, na barra lateral, selecione **(+) Novo** e selecione **Upload de Arquivo** e carregue o arquivo **penguins.csv** que foi baixado no computador.
1. Na página **Carregar dados**, selecione o esquema **padrão** e defina o nome da tabela como **pinguins**. Em seguida, selecione **Criar tabela** no canto inferior esquerdo da página.
1. Quando a tabela tiver sido criada, revise seus detalhes.

## Criar um experimento de AutoML

Agora que você tem alguns dados, pode usá-los com o AutoML para treinar um modelo.

1. Na barra lateral à esquerda, selecione **Experimentos**.
1. Na página **Experimentos**, selecione **Criar experimento de AutoML**.
1. Configure o experimento de AutoML com as seguintes definições:
    - **Cluster**: *Selecione seu cluster*
    - **Tipo de problema de ML**: Classificação
    - **Entrada de conjunto de dados de treinamento**: *Navegue até o banco de dados de **padrão** e selecione a tabela **pinguins***
    - **Meta de previsão**: Espécie
    - **Nome do experimento**: Classificação de pinguins
    - **Configuração avançada**:
        - **Métrica de avaliação**: Precisão
        - **Estruturas de treinamento**: lightgbm, sklearn, xgboost
        - **Tempo limite**: 5
        - **Coluna de tempo para divisão de treinamento/validação/teste**: *Deixar em branco*
        - **Rótulo positivo**: *Deixar em branco*
        - **Local de armazenamento intermediário de dados**: Artefato do MLflow
1. Use o botão **Iniciar AutoML** para iniciar o experimento. Feche os diálogos de informações exibidos.
1. Aguarde a conclusão do experimento. Você pode usar o botão **Atualizar** à direita para exibir detalhes das execuções geradas.
1. Após cinco minutos, o experimento será encerrado. Atualizar as execuções mostrará a execução que resultou no modelo de melhor desempenho (com base na métrica de *precisão* selecionada) na parte superior da lista.

## Implantar o modelo de melhor desempenho.

Depois de executar um experimento de AutoML, você pode explorar o modelo de melhor desempenho que ele gerou.

1. Na página do experimento de **Classificação de pinguins**, selecione **Exibir notebook para o melhor modelo** para abrir o notebook usado para fazer chover o modelo em uma nova guia do navegador.
1. Percorra as células do notebook, anotando o código que foi usado para treinar o modelo.
1. Feche a guia do navegador que contém o notebook para retornar à página de teste de **Classificação de pinguins**.
1. Na lista de execuções, selecione o nome da primeira execução (que produziu o melhor modelo) para abri-lo.
1. Na seção **Artefatos**, observe que o modelo foi salvo como um artefato do MLflow. Em seguida, use o botão **Registrar modelo** para registrar o modelo como um novo modelo chamado **Classificador de Pinguins**.
1. Na barra lateral à esquerda, alterne para a página **Modelos**. Em seguida, selecione o modelo **Classificador de Pinguins** que você acabou de registrar.
1. Na página **Classificador de Pinguins**, use o botão **Usar modelo para inferência** para criar um novo ponto de extremidade em tempo real com as seguintes configurações:
    - **Modelo**: Classificador de Pinguins
    - **Versão do modelo**: 1
    - **Ponto de extremidade**: classificar pinguim
    - **Tamanho da computação**: Small

    O ponto de extremidade de serviço é hospedado em um novo cluster, que pode levar vários minutos para ser criado.
  
1. Quando o ponto de extremidade tiver sido criado, use o botão **Consultar ponto de extremidade** no canto superior direito para abrir uma interface da qual você pode testar o ponto de extremidade. Em seguida, na interface de teste, na guia **Navegador**, insira a seguinte solicitação JSON e use o botão **Enviar Solicitação** para chamar o ponto de extremidade e gerar uma previsão.

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

1. Faça experiências com alguns valores diferentes para os recursos de pinguim e observe os resultados retornados. Em seguida, feche a interface de teste.

## Excluir o ponto de extremidade

Quando o ponto de extremidade não for mais necessário, você deverá excluí-lo para evitar custos desnecessários.

Na página do ponto de extremidade **classificar pinguim**, no menu **&#8285;**, selecione **Excluir**.

## Limpeza

No portal do Azure Databricks, na página **Computação**, selecione seu cluster e selecione **&#9632; Terminar** para encerrar o processo.

Se você tiver terminado de explorar o Azure Databricks, poderá excluir os recursos que criou para evitar custos desnecessários do Azure e liberar capacidade em sua assinatura.

> **Mais informações**: Para obter mais informações, consulte [Como o AutoML do Databricks funciona](https://learn.microsoft.com/en-us/azure/databricks/machine-learning/automl/how-automl-works) na documentação do Azure Databricks.
