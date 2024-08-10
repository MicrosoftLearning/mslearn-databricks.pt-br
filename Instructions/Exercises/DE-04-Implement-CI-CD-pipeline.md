---
lab:
  title: Implementar fluxos de trabalho de CI/CD com o Azure Databricks
---

# Implementar fluxos de trabalho de CI/CD com o Azure Databricks

A implementação de pipelines de CI (Integração Contínua) e CD (Implantação Contínua) com o Azure Databricks e o Azure DevOps ou o Azure Databricks e o GitHub envolve a configuração de uma série de etapas automatizadas para garantir que as alterações de código sejam integradas, testadas e implantadas de modo eficiente. O processo normalmente inclui a conexão com um repositório Git, a execução de trabalhos usando o Azure Pipelines para compilar e testar o código, e implantar os artefatos de build para uso em notebooks do Databricks. Esse fluxo de trabalho permite um ciclo de desenvolvimento robusto, possibilitando a integração e a entrega contínuas alinhadas com as práticas modernas de DevOps.

Este laboratório levará aproximadamente **40** minutos para ser concluído.

>**Observação:** você precisa de uma conta do Github e acesso ao Azure DevOps para concluir este exercício.

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
    - **Tipo de nó**: Standard_DS3_v2
    - **Encerra após** *20* **minutos de inatividade**

1. Aguarde a criação do cluster. Isso pode levar alguns minutos.

    > **Observação**: se o cluster não for iniciado, sua assinatura pode ter cota insuficiente na região onde seu workspace do Azure Databricks está provisionado. Consulte [Limite de núcleo da CPU impede a criação do cluster](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit) para obter detalhes. Se isso acontecer, você pode tentar excluir seu workspace e criar um novo workspace em uma região diferente. Você pode especificar uma região como um parâmetro para o script de instalação da seguinte maneira: `./mslearn-databricks/setup.ps1 eastus`

## Criar um notebook e ingerir dados

1. Na barra lateral, use o link **(+) Novo** para criar um **Notebook**. Na lista suspensa **Conectar**, selecione o cluster caso ainda não esteja selecionado. Se o cluster não executar, é porque ele pode levar cerca de um minuto para iniciar.

2. Na primeira célula do notebook, insira o código a seguir, que usa os comandos de *shell* para baixar os arquivos de dados do GitHub para o sistema de arquivos usado pelo cluster.

     ```python
    %sh
    rm -r /dbfs/FileStore
    mkdir /dbfs/FileStore
    wget -O /dbfs/FileStore/sample_sales.csv https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/sample_sales.csv
     ```

3. Use a opção de menu **&#9656; Executar Célula** à esquerda da célula para executá-la. Em seguida, aguarde o término do trabalho do Spark executado pelo código.
   
## Configurar um repositório GitHub e um projeto do Azure DevOps

Depois de conectar um repositório GitHub a um projeto do Azure DevOps, você pode configurar pipelines de CI que são disparados com todas as alterações feitas no repositório.

1. Acesse sua [conta do GitHub](https://github.com/) e crie um novo repositório para seu projeto.

2. Clone o repositório em seu computador local usando `git clone`.

3. Baixe o [arquivo CSV](https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/sample_sales.csv) no repositório local e confirme as alterações.

4. Baixe o [notebook do Databricks](https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/sample_sales_notebook.dbc) que será usado para ler o arquivo CSV e executar a transformação de dados. Confirme as alterações.

5. Acesse o [portal do Azure DevOps](https://azure.microsoft.com/en-us/products/devops/) e crie um novo projeto.

6. No projeto do Azure DevOps, acesse a seção **Repos** e selecione **Importar** para conectá-lo ao repositório GitHub.

7. Na barra do lado esquerdo, vá até **Configurações do projeto > Conexões de serviço**.

8. Selecione **Criar conexão de serviço** e **Azure Resource Manager**.

9. Em **Método de autenticação**, selecione **Federação de Identidade de carga de trabalho (automática)**. Selecione **Avançar**.

10. Em **Nível do escopo**, selecione **Assinatura**. Selecione a assinatura e o grupo de recursos em que você criou o workspace do Databricks.

11. Insira um nome para sua conexão de serviço e marque a opção **Conceder permissão de acesso a todos os pipelines**. Selecione **Salvar**.

Agora seu projeto do [Azure] DevOps tem acesso ao workspace do Databricks e você pode conectá-lo aos pipelines.

## Configurar um pipeline de CI

1. Na barra do lado esquerdo, vá até **Pipelines** e selecione **Criar pipeline**.

2. Selecione o **GitHub** como fonte e selecione seu repositório.

3. No painel **Configurar seu pipeline**, selecione **Pipeline inicial** e use a seguinte configuração YAML para o pipeline de CI:

```yaml
trigger:
- main

pool:
  vmImage: 'ubuntu-latest'

steps:
- task: UsePythonVersion@0
  inputs:
    versionSpec: '3.x'
    addToPath: true

- script: |
    pip install databricks-cli
  displayName: 'Install Databricks CLI'

- script: |
    databricks configure --token <<EOF
    <your-databricks-host>
    <your-databricks-token>
    EOF
  displayName: 'Configure Databricks CLI'

- script: |
    databricks fs cp dbfs:/FileStore/sample_sales.csv . --overwrite
  displayName: 'Download Sample Data from DBFS'
```

4. Substitua `<your-databricks-host>` e `<your-databricks-token>` pelo URL e pelo token do host do Databricks. Isso configurará a CLI do Databricks antes de tentar usá-la.

5. Selecione **Salvar e executar**.

Esse arquivo YAML configurará um pipeline de CI que é acionado por alterações na ramificação `main` do repositório. O pipeline configura um ambiente do Python, instala a CLI do Databricks e baixa os dados de exemplo do seu workspace do Databricks. Essa é uma configuração comum para fluxos de trabalho de CI.

## Configurar um pipeline de CD

1. Na barra do lado esquerdo, vá até **Pipelines > Lançamentos** e selecione **Criar lançamento**.

2. Selecione o pipeline de build como a origem do artefato.

3. Adicione um estágio e configure as tarefas que serão implantadas no Azure Databricks:

```yaml
stages:
- stage: Deploy
  jobs:
  - job: DeployToDatabricks
    pool:
      vmImage: 'ubuntu-latest'
    steps:
    - task: UsePythonVersion@0
      inputs:
        versionSpec: '3.x'
        addToPath: true

    - script: |
        pip install databricks-cli
      displayName: 'Install Databricks CLI'

    - script: |
        databricks configure --token <<EOF
        <your-databricks-host>
        <your-databricks-token>
        EOF
      displayName: 'Configure Databricks CLI'

    - script: |
        databricks workspace import_dir /path/to/notebooks /Workspace/Notebooks
      displayName: 'Deploy Notebooks to Databricks'
```

Antes de executar esse pipeline, substitua `/path/to/notebooks` pelo caminho para o diretório do notebook em seu repositório e `/Workspace/Notebooks` pelo caminho do arquivo em que você deseja que o notebook seja salvo no workspace do Databricks.

4. Selecione **Salvar e executar**.

## Executar os pipelines

1. No repositório local, adicione a seguinte linha ao final do arquivo `sample_sales.csv`:

     ```sql
    2024-01-01,ProductG,1,500
     ```

2. Faça commit das alterações e efetue push delas para o repositório GitHub.

3. As alterações no repositório acionarão o pipeline de CI. Verifique se a execução do pipeline de CI foi concluída com êxito.

4. Crie uma nova versão no pipeline de lançamento e implante os notebooks no Databricks. Verifique se os notebooks foram implantados e executados com êxito no workspace do Databricks.

## Limpar

No portal do Azure Databricks, na página **Computação**, selecione seu cluster e selecione **&#9632; Terminar** para encerrar o processo.

Se você tiver terminado de explorar o Azure Databricks, poderá excluir os recursos que criou para evitar custos desnecessários do Azure e liberar capacidade em sua assinatura.







