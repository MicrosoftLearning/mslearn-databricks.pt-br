---
lab:
  title: Implementação da privacidade e da governança de dados usando o Microsoft Purview e o catálogo do Unity com o Azure Databricks
---

# Implementação da privacidade e da governança de dados usando o Microsoft Purview e o catálogo do Unity com o Azure Databricks

O Microsoft Purview permite uma governança de dados abrangente em todo o seu patrimônio de dados, integrando-se perfeitamente ao Azure Databricks para gerenciar dados do Lakehouse e trazer metadados para o Mapa de dados. O catálogo do Unity aprimora isso fornecendo gerenciamento e governança de dados centralizados, simplificando a segurança e a conformidade nos workspaces do Databricks.

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
    - **Tipo de nó**: Standard_DS3_v2
    - **Encerra após** *20* **minutos de inatividade**

1. Aguarde a criação do cluster. Isso pode levar alguns minutos.

    > **Observação**: se o cluster não for iniciado, sua assinatura pode ter cota insuficiente na região onde seu workspace do Azure Databricks está provisionado. Consulte [Limite de núcleo da CPU impede a criação do cluster](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit) para obter detalhes. Se isso acontecer, você pode tentar excluir seu workspace e criar um novo workspace em uma região diferente. Você pode especificar uma região como um parâmetro para o script de instalação da seguinte maneira: `./mslearn-databricks/setup.ps1 eastus`

## Configurar o Catálogo do Unity

Os metastores do Catálogo do Unity registram metadados sobre objetos protegíveis (como tabelas, volumes, locais externos e compartilhamentos) e as permissões que regem o acesso a eles. Cada metastore expõe um namespace de três níveis (`catalog`.`schema`.`table`) pelo qual os dados podem ser organizados. Você deve ter um metastore para cada região em que sua organização opera. Para trabalhar com o Catálogo do Unity, os usuários devem estar em um workspace anexado a um metastore em sua região.

1. Na barra lateral, selecione **Catálogo**.

2. No Explorador do Catálogo, um Catálogo padrão do Unity com o nome do workspace (**databricks-*xxxxxxx*** se você usou o script de instalação para criá-lo) deve estar presente. Selecione o catálogo e, na parte superior do painel direito, selecione **Criar esquema**.

3. Nomeie o novo esquema de **comércio eletrônico**, escolha o local de armazenamento criado com seu workspace e selecione **Criar**.

4. Selecione o catálogo e, no painel direito, selecione a guia **Workspaces**. Verifique se o workspace tem acesso `Read & Write` a ele.

## Ingerir dados de amostra no Azure Databricks

1. Baixe os arquivos de dados de amostra:
   * [customers.csv](https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/DE-05/customers.csv)
   * [products.csv](https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/DE-05/products.csv)
   * [sales.csv](https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/DE-05/sales.csv)

2. No workspace do Azure Databricks, na parte superior do explorador de catálogos, selecione **+** e **Adicionar dados**.

3. Na nova janela, selecione **Carregar arquivos no volume**.

4. Na nova janela, vá até o esquema `ecommerce`, expanda-o e selecione **Criar um volume**.

5. Nomeie o novo volume **sample_data** e selecione **Criar**.

6. Selecione o novo volume e faça o upload dos arquivos `customers.csv`, `products.csv` e `sales.csv`. Escolha **Carregar**.

7. Na barra lateral, use o link **(+) Novo** para criar um **Notebook**. Na lista suspensa **Conectar**, selecione o cluster caso ainda não esteja selecionado. Se o cluster não executar, é porque ele pode levar cerca de um minuto para iniciar.

8. Na primeira célula do notebook, digite o seguinte código para criar tabelas a partir dos arquivos CSV:

     ```python
    # Load Customer Data
    customers_df = spark.read.format("csv").option("header", "true").load("/Volumes/databricksxxxxxxx/ecommerce/sample_data/customers.csv")
    customers_df.write.saveAsTable("ecommerce.customers")

    # Load Sales Data
    sales_df = spark.read.format("csv").option("header", "true").load("/Volumes/databricksxxxxxxx/ecommerce/sample_data/sales.csv")
    sales_df.write.saveAsTable("ecommerce.sales")

    # Load Product Data
    products_df = spark.read.format("csv").option("header", "true").load("/Volumes/databricksxxxxxxx/ecommerce/sample_data/products.csv")
    products_df.write.saveAsTable("ecommerce.products")
     ```

>**Observação:** no caminho do arquivo `.load`, substitua `databricksxxxxxxx` pelo nome do catálogo.

9. No explorador de catálogos, vá até o volume `sample_data` e verifique se as novas tabelas estão dentro dele.
    
## Configurar o Microsoft Purview

O Microsoft Purview é um serviço unificado de governança de dados que ajuda as organizações a gerenciar e proteger seus dados em vários ambientes. Com recursos como prevenção contra perda de dados, proteção de informações e gerenciamento de conformidade, o Microsoft Purview oferece ferramentas para entender, gerenciar e proteger dados em todo o ciclo de vida.

1. Navegue até o [Portal do Azure](https://portal.azure.com/).

2. Selecione **Criar um recurso** e pesquise por **Microsoft Purview**.

3. Crie um recurso **Microsoft Purview** com as seguintes configurações:
    - **Assinatura**: *Selecione sua assinatura do Azure*
    - **Grupo de recursos**: *escolha o mesmo grupo de recursos que seu workspace do Azure Databricks*
    - **Nome da conta do Microsoft Purview**: *um nome exclusivo de sua escolha*
    - **Local**: *selecione a mesma região que o workspace do Azure Databricks*

4. Selecione **Examinar + criar**. Aguarde a validação e selecione **Criar**.

5. Aguarde o fim da implantação. Em seguida, vá para o recurso Microsoft Purview implantado no portal do Azure.

6. No portal de governança do Microsoft Purview, vá até a seção **Mapa de dados** na barra lateral.

7. No painel **Fontes de dados**, selecione **Registrar**.

8. Na janela **Registrar fonte de dados**, pesquise por e selecione o **Azure Databricks**. Selecione **Continuar**.

9. Dê um nome exclusivo à fonte de dados e selecione o workspace do Azure Databricks. Selecione **Registrar**.

## Implementar políticas de privacidade e governança de dados

1. Na seção **Mapa de dados** da barra lateral, selecione **Classificações**.

2. No painel **Classificações**, selecione **+ Novo** e crie uma nova classificação chamada **PII** (Informações de identificação pessoal). Selecione **OK**.

3. Selecione **Catálogo de Dados** na barra lateral e vá até a tabela de **clientes**.

4. Aplique a classificação de PII às colunas de email e telefone.

5. Acesse o Azure Databricks e abra o notebook criado anteriormente.
 
6. Em uma nova célula, execute o código a seguir para criar uma política de acesso a dados para restringir o acesso a dados PII.

     ```sql
    CREATE OR REPLACE TABLE ecommerce.customers (
      customer_id STRING,
      name STRING,
      email STRING,
      phone STRING,
      address STRING,
      city STRING,
      state STRING,
      zip_code STRING,
      country STRING
    ) TBLPROPERTIES ('data_classification'='PII');

    GRANT SELECT ON TABLE ecommerce.customers TO ROLE data_scientist;
    REVOKE SELECT (email, phone) ON TABLE ecommerce.customers FROM ROLE data_scientist;
     ```

7. Tente consultar a tabela de clientes como um usuário com a função data_scientist. Verifique se o acesso às colunas de PII (email e telefone) é restrito.

## Limpar

No portal do Azure Databricks, na página **Computação**, selecione seu cluster e selecione **&#9632; Terminar** para encerrar o processo.

Se você tiver terminado de explorar o Azure Databricks, poderá excluir os recursos que criou para evitar custos desnecessários do Azure e liberar capacidade em sua assinatura.
