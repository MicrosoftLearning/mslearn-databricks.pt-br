---
lab:
  title: Preterido – Automatizar um Notebook do Azure Databricks com o Azure Data Factory
---

# Automatizar um Notebook do Azure Databricks com o Azure Data Factory

Você pode usar notebooks no Azure Databricks para executar tarefas de engenharia de dados, como processar arquivos de dados e carregar dados em tabelas. Quando você precisar orquestrar essas tarefas como parte de um pipeline de engenharia de dados, poderá usar o Azure Data Factory.

Este exercício levará aproximadamente **40** minutos para ser concluído.

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
7. Aguarde a conclusão do script - isso normalmente leva cerca de 5 minutos, mas em alguns casos pode levar mais tempo. Enquanto espera, revise [O que é o Azure Data Factory?](https://docs.microsoft.com/azure/data-factory/introduction).

## Criar um recurso do Azure Data Factory

Além do workspace do Azure Databricks, você precisará provisionar um recurso do Azure Data Factory em sua assinatura.

1. No portal do Azure, feche o painel do Cloud Shell e navegue até o grupo de recursos ***msl-*xxxxxxx*** criado pelo script de configuração (ou o grupo de recursos que contém seu workspace do Azure Databricks).
1. Na barra de ferramentas, selecione **+ Criar** e pesquise `Data Factory`. Em seguida, crie um novo recurso do **Data Factory** com a seguinte configuração:
    - **Assinatura**: *Sua assinatura*
    - **Grupo de recursos**: msl-*xxxxxxx* (ou o grupo de recursos que contém o workspace do Azure Databricks existente)
    - **Nome**: *Um nome exclusivo, por exemplo, **adf-xxxxxxx***
    - **Região**: *A mesma região que o workspace do Azure Databricks (ou qualquer outra região disponível se não estiver listada)*
    - **Versão**: V2
1. Quando o novo recurso for criado, verifique se o grupo de recursos contém o workspace do Azure Databricks e os recursos do Azure Data Factory.

## Criar um notebook

Você pode criar notebooks em seu workspace do Azure Databricks para executar código escrito em uma variedade de linguagens de programação. Neste exercício, você criará um notebook simples que ingerirá dados de um arquivo e os salvará em uma pasta no DBFS (Sistema de Arquivos do Databricks).

1. No portal do Azure, navegue até o grupo de recursos **msl-*xxxxxxx*** criado pelo script (ou o grupo de recursos que contém seu workspace do Azure Databricks)
1. Selecione o recurso Serviço do Azure Databricks (chamado **databricks-*xxxxxxx*** se você usou o script de instalação para criá-lo).
1. Na página **Visão geral** do seu workspace, use o botão **Iniciar workspace** para abrir seu workspace do Azure Databricks em uma nova guia do navegador, fazendo o logon se solicitado.

    > **Dica**: ao usar o portal do workspace do Databricks, várias dicas e notificações podem ser exibidas. Dispense-as e siga as instruções fornecidas para concluir as tarefas neste exercício.

1. Veja o portal do workspace do Azure Databricks e observe que a barra lateral à esquerda contém ícones para as várias tarefas executáveis.
1. Na barra lateral, use o link **(+) Novo** para criar um **Notebook**.
1. Altere o nome do notebook padrão (**Untitled Notebook *[date]***) para **Processar Dados**.
1. Na primeira célula do notebook, insira (mas não execute) o código a seguir para configurar uma variável para a pasta em que esse notebook salvará os dados.

    ```python
   # Use dbutils.widget define a "folder" variable with a default value
   dbutils.widgets.text("folder", "data")
   
   # Now get the parameter value (if no value was passed, the default set above will be used)
   folder = dbutils.widgets.get("folder")
    ```

1. Na célula de código existente, use o ícone **+** para adicionar uma nova célula de código. Em seguida, na nova célula, insira (mas não execute) o seguinte código para baixar dados e salvá-los na pasta:

    ```python
   import urllib3
   
   # Download product data from GitHub
   response = urllib3.PoolManager().request('GET', 'https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/products.csv')
   data = response.data.decode("utf-8")
   
   # Save the product data to the specified folder
   path = "dbfs:/{0}/products.csv".format(folder)
   dbutils.fs.put(path, data, True)
    ```

1. Na barra lateral à esquerda, selecione **Workspace** e verifique se os notebooks **Processar Dados** estão listados. Você usará o Azure Data Factory para executar o notebook como parte de um pipeline.

    > **Observação**: O notebook pode conter praticamente qualquer lógica de processamento de dados necessária. Este exemplo simples foi projetado para mostrar os princípios essenciais.

## Ativar a integração do Azure Databricks com o Azure Data Factory

Para usar o Azure Databricks de um pipeline do Azure Data Factory, você precisa criar um serviço vinculado no Azure Data Factory que permita o acesso ao workspace do Azure Databricks.

### Gerar um token de acesso

1. No portal do Azure Databricks, na barra de menus superior direita, selecione o nome de usuário e, em seguida, selecione **Configurações do Usuário** na lista suspensa.
1. Na página **Configurações do Usuário**, selecione **Desenvolvedor**. Em seguida, ao lado de **Tokens de acesso**, selecione **Gerenciar**.
1. Selecione **Gerar novo token** e gere um novo token com o comentário *Data Factory* e um tempo de vida vazio (para que o token não expire). Tenha cuidado para **copiar o token quando ele for exibido <u>antes</u> de selecionar *Concluído***.
1. Cole o token copiado em um arquivo de texto para tê-lo à mão para mais tarde neste exercício.

### Criar um serviço vinculado no Azure Data Factory

1. Retorne ao portal do Azure e, no grupo de recursos **msl-*xxxxxxx***, selecione o recurso **adf*xxxxxxx*** do Azure Data Factory.
2. Na página **Visão geral**, selecione **Iniciar estúdio** para abrir o Azure Data Factory Studio. Entre, se for solicitado.
3. No Azure Data Factory Studio, use o ícone **>>** para expandir o painel de navegação à esquerda. Em seguida, selecione a página **Gerenciar**.
4. Na página **Gerenciar**, na guia **Serviços vinculados**, selecione **+ Novo** para adicionar um novo serviço vinculado.
5. Na janela **Novo serviço vinculado**, selecione a guia **Computação** na parte superior. Em seguida, selecione **Azure Databricks**.
6. Continue e crie o serviço vinculado com as seguintes configurações:
    - **Nome**: AzureDatabricks
    - **Descrição**: workspace do Azure Databricks
    - **Conectar por meio do tempo de execução de integração**: AutoResolveIntegrationRuntime
    - **Método de seleção de conta**: Da assinatura do Azure
    - **Assinatura do Azure**: *Selecione sua assinatura*
    - **Workspace do Databricks**: *selecione seu workspace **databricksxxxxxxx***
    - **Selecionar cluster**: Novo cluster de trabalho
    - **URL do Workspace do Databricks**: *Definir automaticamente seu URL do workspace do Databricks*
    - **Tipo de autenticação**: Token de acesso
    - **Token de acesso**: *cole seu token de acesso*
    - **Versão do cluster**: 13.3 LTS (Spark 3.4.1, Scala 2.12)
    - **Tipo de nó do cluster**: Standard_DS3_v2
    - **Versão do Python**: 3
    - **Opções de trabalho**: Fixo
    - **Trabalhos**: 1

## Usar um pipeline para executar o notebook do Azure Databricks

Agora que você criou um serviço vinculado, pode usá-lo em um pipeline para executar o notebook exibido anteriormente.

### Criar um pipeline

1. No Azure Data Factory Studio, no painel de navegação, selecione **Autor**.
2. Na página **Autor**, no painel **Recursos de Fábrica**, use o ícone **+** para adicionar um **Pipeline**.
3. No painel **Propriedades** do novo pipeline, altere seu nome para **Processar Dados com Databricks**. Em seguida, use o botão **Propriedades** (que se parece com **<sub>*</sub>**) na extremidade direita da barra de ferramentas para ocultar o painel de **Propriedades**.
4. No painel **Atividades**, expanda **Databricks** e arraste outra atividade de **Notebook** para a superfície do designer de pipeline.
5. Com a nova atividade **Notebook1** selecionada, defina as seguintes propriedades no painel inferior:
    - **Geral:**
        - **Nome**: Dados do Processo
    - **Azure Databricks**:
        - **Serviço vinculado do Databricks**: *Selecione o serviço vinculado **AzureDatabricks** que você criou anteriormente*
    - **Configurações**:
        - **Caminho do Notebook**: *Navegue até a pasta **Users/your_user_name** e selecione o notebook **Processar Dados***
        - **Parâmetros de base**: *Adicionar um novo parâmetro chamado `folder` com o valor `product_data`*
6. Use o botão **Validar** acima da superfície do designer de pipeline para validar o pipeline. Em seguida, use o botão **Publicar tudo** para publicá-lo (salvá-lo).

### Executar o pipeline

1. Acima da superfície de designer do pipeline, selecione **Adicionar gatilho** e, em seguida, selecione **Acionar agora**.
2. No painel **Execução do pipeline**, selecione **OK** para executar o pipeline.
3. No painel de navegação à esquerda, selecione **Monitorar** e observe o pipeline **Processar Dados com Databricks** na guia **Execuções de pipeline**. A execução pode demorar um pouco, pois um cluster do Spark é dinamicamente criado e um notebook é executado. Você pode usar o botão **↻ Atualizar** na página **Execuções de pipeline** para atualizar o status.

    > **Observação**: se o pipeline falhar, a assinatura pode ter cota insuficiente na região onde seu workspace do Azure Databricks é provisionado para criar um cluster de trabalho. Consulte [Limite de núcleo da CPU impede a criação do cluster](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit) para obter detalhes. Se isso acontecer, você pode tentar excluir seu workspace e criar um novo workspace em uma região diferente. Você pode especificar uma região como um parâmetro para o script de instalação da seguinte maneira: `./setup.ps1 eastus`

4. Quando a execução for bem-sucedida, selecione seu nome para exibir os detalhes da execução. Em seguida, na página **Processar Dados com Databricks**, na seção **Execuções da Atividade**, selecione a atividade **Processar Dados** e use o ícone de ***saída*** para ver a saída JSON na atividade, que será semelhante a isto:

    ```json
    {
        "runPageUrl": "https://adb-..../run/...",
        "runOutput": "dbfs:/product_data/products.csv",
        "effectiveIntegrationRuntime": "AutoResolveIntegrationRuntime (East US)",
        "executionDuration": 61,
        "durationInQueue": {
            "integrationRuntimeQueue": 0
        },
        "billingReference": {
            "activityType": "ExternalActivity",
            "billableDuration": [
                {
                    "meterType": "AzureIR",
                    "duration": 0.03333333333333333,
                    "unit": "Hours"
                }
            ]
        }
    }
    ```

5. Observe o valor **runOutput**, que é a variável de *caminho* na qual o notebook salvou os dados.

## Limpeza

Se você terminou de explorar o Azure Databricks, exclua os recursos que criou para evitar custos desnecessários do Azure e liberar capacidade em sua assinatura.
