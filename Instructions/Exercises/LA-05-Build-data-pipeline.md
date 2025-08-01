---
lab:
  title: Criar um pipeline de dados com tabelas do Delta Live
---

# Criar um pipeline de dados com tabelas do Delta Live

As Tabelas Dinâmicas Delta são uma estrutura declarativa para a criação de pipelines de processamento de dados confiáveis, testáveis e de fácil manutenção. Um pipeline é a principal unidade usada para configurar e executar fluxos de trabalho de processamento de dados com o Delta Live Tables. Ele vincula fontes de dados a conjuntos de dados de destino por meio de um DAG (grafo direcionado acíclico) declarado em Python ou SQL.

Este laboratório levará aproximadamente **40** minutos para ser concluído.

> **Observação**: a interface do usuário do Azure Databricks está sujeita a melhorias contínuas. A interface do usuário pode ter sido alterada desde que as instruções neste exercício foram escritas.

## Provisionar um workspace do Azure Databricks

> **Dica**: Se você já tem um workspace do Azure Databricks, pode ignorar esse procedimento e usar o workspace existente.

Este exercício inclui um script para provisionar um novo workspace do Azure Databricks. O script tenta criar um recurso de workspace do Azure Databricks de camada *Premium* em uma região na qual sua assinatura do Azure tenha cota suficiente para os núcleos de computação necessários para este exercício; e pressupõe que sua conta de usuário tenha permissões suficientes na assinatura para criar um recurso de workspace do Azure Databricks. Se o script falhar devido a cota ou permissões insuficientes, você pode tentar [criar um workspace do Azure Databricks interativamente no portal do Azure](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace).

1. Em um navegador da web, faça logon no [portal do Azure](https://portal.azure.com) em `https://portal.azure.com`.
2. Use o botão **[\>_]** à direita da barra de pesquisa na parte superior da página para criar um Cloud Shell no portal do Azure selecionando um ambiente do ***PowerShell***. O Cloud Shell fornece uma interface de linha de comando em um painel na parte inferior do portal do Azure, conforme mostrado aqui:

    ![Portal do Azure com um painel do Cloud Shell](./images/cloud-shell.png)

    > **Observação**: se você já criou um Cloud Shell que usa um ambiente *Bash*, alterne-o para o ***PowerShell***.

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

7. Aguarde a conclusão do script – isso normalmente leva cerca de 5 minutos, mas em alguns casos pode levar mais tempo. Enquanto espera, revise o artigo [O que é o Delta Live Tables?](https://learn.microsoft.com/azure/databricks/delta-live-tables/) na documentação do Azure Databricks.

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

1. Na barra lateral, use o link **(+) Novo** para criar um **Notebook**.

2. Altere o nome padrão do notebook (**Notebook Sem Título *[data]***) para `Create a pipeline with Delta Live tables` e, na lista suspensa **Conectar**, selecione o cluster, caso ainda não esteja selecionado. Se o cluster não executar, é porque ele pode levar cerca de um minuto para iniciar.

3. Na primeira célula do notebook, insira o código a seguir, que usa os comandos de *shell* para baixar os arquivos de dados do GitHub para o sistema de arquivos usado pelo cluster.

     ```python
    %sh
    rm -r /dbfs/delta_lab
    mkdir /dbfs/delta_lab
    wget -O /dbfs/delta_lab/covid_data.csv https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/covid_data.csv
     ```

4. Use a opção de menu **&#9656; Executar Célula** à esquerda da célula para executá-la. Em seguida, aguarde o término do trabalho do Spark executado pelo código.

## Criar pipeline do Delta Live Tables usando SQL

1. Crie um novo notebook e renomeie-o para `Pipeline Notebook`.

1. Ao lado do nome do notebook, selecione **Python** e altere o idioma padrão para **SQL**.

1. Insira o código a seguir na primeira célula, mas não execute-o. Todas as células serão executadas após a criação do pipeline. Esse código define uma tabela do Delta Live que será preenchida pelos dados brutos baixados anteriormente:

     ```sql
    CREATE OR REFRESH LIVE TABLE raw_covid_data
    COMMENT "COVID sample dataset. This data was ingested from the COVID-19 Data Repository by the Center for Systems Science and Engineering (CSSE) at Johns Hopkins University."
    AS
    SELECT
      Last_Update,
      Country_Region,
      Confirmed,
      Deaths,
      Recovered
    FROM read_files('dbfs:/delta_lab/covid_data.csv', format => 'csv', header => true)
     ```

1. Na primeira célula, use o ícone **+ Código** para adicionar uma nova célula e insira o código a seguir para consultar, filtrar e formatar os dados na tabela anterior antes da análise.

     ```sql
    CREATE OR REFRESH LIVE TABLE processed_covid_data(
      CONSTRAINT valid_country_region EXPECT (Country_Region IS NOT NULL) ON VIOLATION FAIL UPDATE
    )
    COMMENT "Formatted and filtered data for analysis."
    AS
    SELECT
        TO_DATE(Last_Update, 'MM/dd/yyyy') as Report_Date,
        Country_Region,
        Confirmed,
        Deaths,
        Recovered
    FROM live.raw_covid_data;
     ```

1. Em uma terceira célula de código, insira o código a seguir que criará uma exibição de dados enriquecida para análise posterior depois que o pipeline for executado.

     ```sql
    CREATE OR REFRESH LIVE TABLE aggregated_covid_data
    COMMENT "Aggregated daily data for the US with total counts."
    AS
    SELECT
        Report_Date,
        sum(Confirmed) as Total_Confirmed,
        sum(Deaths) as Total_Deaths,
        sum(Recovered) as Total_Recovered
    FROM live.processed_covid_data
    GROUP BY Report_Date;
     ```
     
1. Selecione **Delta Live Tables** na barra lateral esquerda e, em seguida, selecione **Criar pipeline**.

1. Na página **Criar pipeline**, crie um novo pipeline com as seguintes configurações:
    - **Nome do pipeline**: `Covid Pipeline`
    - **Edição do produto**: avançado
    - **Modo do pipeline**: acionado
    - **Código-fonte**: *navegue até* o notebook Notebook do Pipeline *na pasta*Users/user@name *.*
    - **Opções de armazenamento**: metastore do Hive
    - **Local de armazenamento**: `dbfs:/pipelines/delta_lab`
    - **Esquema de destino**: *Insira*`default`

1. Selecione **Criar** e depois **Iniciar**. Em seguida, aguarde a execução do pipeline (o que pode levar algum tempo).
 
1. Depois que o pipeline for executado, volte para o notebook *Criar um pipeline com Delta Live tables* que você criou primeiro e execute o seguinte código em uma nova célula para verificar que os arquivos das três novas tabelas foram criados no local de armazenamento especificado:

     ```python
    display(dbutils.fs.ls("dbfs:/pipelines/delta_lab/schemas/default/tables"))
     ```

1. Adicione outra célula de código e execute o seguinte código para verificar se as tabelas foram criadas no banco de dados **padrão**:

     ```sql
    %sql

    SHOW TABLES
     ```

## Exibir resultados como uma visualização

Depois de criar as tabelas, é possível carregá-las em dataframes e visualizar os dados.

1. No notebook *Cirar um pipeline com Delta Live tables*, adicione uma nova célula de código e execute o seguinte código para carregar o `aggregated_covid_data` em um dataframe:

    ```sql
    %sql
    
    SELECT * FROM aggregated_covid_data
    ```

1. Acima da tabela de resultados, selecione **+** e, em seguida, selecione **Visualização** para exibir o editor de visualização e aplique as seguintes opções:
    - **Tipo de visualização**: em linha
    - **Coluna X**: Report_Date
    - **Coluna Y**: *adicione uma nova coluna e selecione***Total_Confirmed**. *Aplicar a* **Agregação** de *soma*.

1. Salve a visualização para exibir o gráfico resultante no notebook.

## Limpar

No portal do Azure Databricks, na página **Computação**, selecione seu cluster e selecione **&#9632; Terminar** para encerrar o processo.

Se você tiver terminado de explorar o Azure Databricks, poderá excluir os recursos que criou para evitar custos desnecessários do Azure e liberar capacidade em sua assinatura.
