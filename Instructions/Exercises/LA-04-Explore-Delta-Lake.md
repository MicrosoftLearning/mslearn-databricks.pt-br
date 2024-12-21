---
lab:
  title: Usar o Delta Lake no Azure Databricks
---

# Usar o Delta Lake no Azure Databricks

O Delta Lake é um projeto de código aberto para criar uma camada de armazenamento de dados transacionais para o Spark sobre um data lake. O Delta Lake adiciona suporte para a semântica relacional em operações de dados em lote e streaming e permite a criação de uma arquitetura de *Lakehouse* na qual o Apache Spark pode ser usado para processar e consultar dados em tabelas baseadas em arquivos subjacentes no data lake.

Este laboratório levará aproximadamente **30** minutos para ser concluído.

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

Agora, vamos criar um notebook Spark e importar os dados com os quais trabalharemos neste exercício.

1. Na barra lateral, use o link **(+) Novo** para criar um **Notebook**.

1. Altere o nome padrão do notebook (**Notebook Sem Título *[data]***) para `Explore Delta Lake` e, na lista suspensa **Conectar**, selecione o cluster, caso ainda não esteja selecionado. Se o cluster não executar, é porque ele pode levar cerca de um minuto para iniciar.

1. Na primeira célula do notebook, insira o código a seguir, que usa os comandos de *shell* para baixar os arquivos de dados do GitHub para o sistema de arquivos usado pelo cluster.

    ```python
    %sh
    rm -r /dbfs/delta_lab
    mkdir /dbfs/delta_lab
    wget -O /dbfs/delta_lab/products.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/products.csv
    ```

1. Use a opção de menu **&#9656; Executar Célula** à esquerda da célula para executá-la. Em seguida, aguarde o término do trabalho do Spark executado pelo código.

1. Na célula de código existente, use o ícone **+ Código** para adicionar uma nova célula de código. Então, na nova célula, insira e execute o código a seguir para carregar os dados do arquivo e exibir as primeiras 10 linhas.

    ```python
   df = spark.read.load('/delta_lab/products.csv', format='csv', header=True)
   display(df.limit(10))
    ```

## Carregar os dados do arquivo em uma tabela Delta

Os dados foram carregados em um dataframe. Vamos manter isso em uma tabela delta.

1. Adicione uma nova célula de código e use-a para executar o código a seguir:

    ```python
   delta_table_path = "/delta/products-delta"
   df.write.format("delta").save(delta_table_path)
    ```

    Os dados de uma tabela delta lake são armazenados no formato Parquet. Um arquivo de log também é criado para acompanhar as modificações feitas nos dados.

1. Adicione uma nova célula de código e use-a para executar os comandos de shell a seguir para exibir o conteúdo da pasta em que os dados delta foram salvos.

    ```
    %sh
    ls /dbfs/delta/products-delta
    ```

1. Os dados do arquivo no formato Delta podem ser carregados em um objeto **DeltaTable**, que você pode usar para exibir e atualizar os dados na tabela. Execute o código a seguir em uma nova célula para atualizar os dados; reduzindo o preço do produto 771 em 10%.

    ```python
   from delta.tables import *
   from pyspark.sql.functions import *
   
   # Create a deltaTable object
   deltaTable = DeltaTable.forPath(spark, delta_table_path)
   # Update the table (reduce price of product 771 by 10%)
   deltaTable.update(
       condition = "ProductID == 771",
       set = { "ListPrice": "ListPrice * 0.9" })
   # View the updated data as a dataframe
   deltaTable.toDF().show(10)
    ```

    A atualização é mantida para os dados na pasta delta e será refletida em qualquer novo dataframe carregado desse local.

1. Execute o seguinte código para criar um novo dataframe com base nos dados da tabela delta:

    ```python
   new_df = spark.read.format("delta").load(delta_table_path)
   new_df.show(10)
    ```

## Explorar o registro em log e de *viagem no tempo*

As modificações de dados são registradas em log, permitindo que você use as recursos de *viagem no tempo* do Delta Lake para exibir versões anteriores dos dados. 

1. Em uma nova célula de código, use o seguinte código para exibir a versão original dos dados do produto:

    ```python
   new_df = spark.read.format("delta").option("versionAsOf", 0).load(delta_table_path)
   new_df.show(10)
    ```

1. O log contém um histórico completo de modificações nos dados. Use o código a seguir para ver um registro das últimas 10 alterações:

    ```python
   deltaTable.history(10).show(10, False, True)
    ```

## Criar tabelas de catálogo

Até agora, você trabalhou com tabelas Delta carregando dados da pasta que contém os arquivos parquet nos quais a tabela se baseia. Você pode definir *tabelas de catálogo* que encapsulam os dados e fornecem uma entidade de tabela nomeada que você pode referenciar no código SQL. O Spark é compatível com dois tipos de tabelas de catálogo para o Delta Lake:

- Tabelas *externas* definidas pelo caminho para os arquivos que contêm os dados da tabela.
- As tabelas *gerenciadas*, que são definidas no metastore.

### Criar uma tabela externa

1. Use o código a seguir para criar um novo banco de dados chamado **AdventureWorks** e, em seguida, cria uma tabela externa chamada **ProductsExternal** nesse banco de dados com base no caminho para os arquivos Delta definidos anteriormente:

    ```python
   spark.sql("CREATE DATABASE AdventureWorks")
   spark.sql("CREATE TABLE AdventureWorks.ProductsExternal USING DELTA LOCATION '{0}'".format(delta_table_path))
   spark.sql("DESCRIBE EXTENDED AdventureWorks.ProductsExternal").show(truncate=False)
    ```

    Observe que a propriedade **Localização** da nova tabela é o caminho especificado.

1. Use o seguinte código para consultar a tabela:

    ```sql
   %sql
   USE AdventureWorks;
   SELECT * FROM ProductsExternal;
    ```

### Criar uma tabela gerenciada

1. Execute o código a seguir para criar (e descrever) uma tabela gerenciada chamada **ProductsManaged** com base no dataframe que você carregou originalmente do arquivo **products.csv** (antes de atualizar o preço do produto 771).

    ```python
   df.write.format("delta").saveAsTable("AdventureWorks.ProductsManaged")
   spark.sql("DESCRIBE EXTENDED AdventureWorks.ProductsManaged").show(truncate=False)
    ```

    Você não especificou um caminho para os arquivos parquet usados pela tabela – ele é gerenciado para você no metastore do Hive e mostrado na propriedade **Localização** na descrição da tabela.

1. Use o seguinte código para consultar a tabela gerenciada, observando que a sintaxe é igual a uma tabela gerenciada:

    ```sql
   %sql
   USE AdventureWorks;
   SELECT * FROM ProductsManaged;
    ```

### Comparar tabelas externas e gerenciadas

1. Use o seguinte código para listar as tabelas no banco de dados **AdventureWorks**:

    ```sql
   %sql
   USE AdventureWorks;
   SHOW TABLES;
    ```

1. Agora, use o seguinte código para ver as pastas nas quais essas tabelas se baseiam:

    ```Bash
    %sh
    echo "External table:"
    ls /dbfs/delta/products-delta
    echo
    echo "Managed table:"
    ls /dbfs/user/hive/warehouse/adventureworks.db/productsmanaged
    ```

1. Use o seguinte código para excluir ambas as tabelas do banco de dados:

    ```sql
   %sql
   USE AdventureWorks;
   DROP TABLE IF EXISTS ProductsExternal;
   DROP TABLE IF EXISTS ProductsManaged;
   SHOW TABLES;
    ```

1. Agora, execute novamente a célula que contém o seguinte código para exibir o conteúdo das pastas delta:

    ```Bash
    %sh
    echo "External table:"
    ls /dbfs/delta/products-delta
    echo
    echo "Managed table:"
    ls /dbfs/user/hive/warehouse/adventureworks.db/productsmanaged
    ```

    Os arquivos da tabela gerenciada são excluídos automaticamente quando a tabela é descartada. No entanto, os arquivos da tabela externa permanecem no local. Remover uma tabela externa remove apenas os metadados da tabela do banco de dados; ele não exclui os arquivos de dados.

1. Use o código a seguir para criar uma nova tabela no banco de dados baseada nos arquivos delta na pasta **products-delta**:

    ```sql
   %sql
   USE AdventureWorks;
   CREATE TABLE Products
   USING DELTA
   LOCATION '/delta/products-delta';
    ```

1. Use o seguinte código para consultar a nova tabela:

    ```sql
   %sql
   USE AdventureWorks;
   SELECT * FROM Products;
    ```

    Como a tabela se baseia nos arquivos delta existentes, que incluem o histórico registrado de alterações, ela reflete as modificações feitas anteriormente nos dados dos produtos.

## Otimizar o layout da tabela

O armazenamento físico dos dados da tabela e dos dados de índice associados pode ser reorganizado para reduzir o espaço de armazenamento e melhorar a eficiência de E/S ao acessar a tabela. Isso é particularmente útil após operações importantes de inserção, atualização ou exclusão em uma tabela.

1. Em uma nova célula de código, use o seguinte código para otimizar o layout e limpar versões antigas de arquivos de dados na tabela delta:

     ```python
    spark.sql("OPTIMIZE Products")
    spark.conf.set("spark.databricks.delta.retentionDurationCheck.enabled", "false")
    spark.sql("VACUUM Products RETAIN 24 HOURS")
     ```

O Delta Lake tem uma verificação de segurança para impedir que você execute um comando VACUUM perigoso. No Databricks Runtime, se você tiver certeza de que não há nenhuma operação em execução nessa tabela que demora mais do que o intervalo de retenção que você planeja especificar, desative essa verificação de segurança definindo a propriedade `spark.databricks.delta.retentionDurationCheck.enabled` da configuração do Spark como `false`.

> **Observação:** se você executar VACUUM em uma tabela Delta, perderá a capacidade de voltar para uma versão anterior ao período de retenção de dados especificado.

## Limpar

No portal do Azure Databricks, na página **Computação**, selecione seu cluster e selecione **&#9632; Terminar** para encerrar o processo.

Se você tiver terminado de explorar o Azure Databricks, poderá excluir os recursos que criou para evitar custos desnecessários do Azure e liberar capacidade em sua assinatura.
