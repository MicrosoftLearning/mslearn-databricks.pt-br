---
lab:
  title: Geração Aumentada de Recuperação usando o Azure Databricks
---

# Geração Aumentada de Recuperação usando o Azure Databricks

A RAG (Geração Aumentada de Recuperação) é uma abordagem de ponta em IA que aprimora grandes modelos de linguagem integrando fontes de conhecimento externas. O Azure Databricks oferece uma plataforma robusta para o desenvolvimento de aplicativos RAG, permitindo a transformação de dados não estruturados em um formato adequado para recuperação e geração de respostas. Esse processo envolve uma série de etapas, incluindo a compreensão da consulta do usuário, a recuperação de dados relevantes e a geração de uma resposta usando um modelo de linguagem. A estrutura fornecida pelo Azure Databricks dá suporte à iteração e implantação rápidas de aplicativos RAG, garantindo respostas específicas de domínio de alta qualidade que podem incluir informações atualizadas e conhecimento proprietário.

Este laboratório levará aproximadamente **40** minutos para ser concluído.

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

    ```powershell
   rm -r mslearn-databricks -f
   git clone https://github.com/MicrosoftLearning/mslearn-databricks
    ```

5. Depois que o repositório tiver sido clonado, insira o seguinte comando para executar **setup.ps1** do script, que provisiona um workspace do Azure Databricks em uma região disponível:

    ```powershell
   ./mslearn-databricks/setup.ps1
    ```

6. Se solicitado, escolha qual assinatura você deseja usar (isso só acontecerá se você tiver acesso a várias assinaturas do Azure).

7. Aguarde a conclusão do script – isso normalmente leva cerca de 5 minutos, mas em alguns casos pode levar mais tempo.

## Criar um cluster

O Azure Databricks é uma plataforma de processamento distribuído que usa *clusters* do Apache Spark para processar dados em paralelo em vários nós. Cada cluster consiste em um nó de driver para coordenar o trabalho e nós de trabalho para executar tarefas de processamento. Neste exercício, você criará um cluster de *nó único* para minimizar os recursos de computação usados no ambiente de laboratório (no qual os recursos podem ser restritos). Em um ambiente de produção, você normalmente criaria um cluster com vários nós de trabalho.

> **Dica**: Se você já tiver um cluster com a versão 16.4 LTS **<u>ML</u>** ou superior do runtime no workspace do Azure Databricks, poderá usá-lo para concluir este exercício e pular este procedimento.

1. No portal do Azure, navegue até o grupo de recursos **msl-*xxxxxxx*** criado pelo script (ou o grupo de recursos que contém seu workspace do Azure Databricks existente)
1. Selecione o recurso Serviço do Azure Databricks (chamado **databricks-*xxxxxxx*** se você usou o script de instalação para criá-lo).
1. Na página **Visão geral** do seu workspace, use o botão **Iniciar workspace** para abrir seu workspace do Azure Databricks em uma nova guia do navegador, fazendo o logon se solicitado.

    > **Dica**: ao usar o portal do workspace do Databricks, várias dicas e notificações podem ser exibidas. Dispense-as e siga as instruções fornecidas para concluir as tarefas neste exercício.

1. Na barra lateral à esquerda, selecione a tarefa **(+) Novo** e, em seguida, selecione **Cluster**.
1. Na página **Novo Cluster**, crie um novo cluster com as seguintes configurações:
    - **Nome do cluster**: cluster *Nome do Usuário* (o nome do cluster padrão)
    - **Política**: Sem restrições
    - **Machine learning**: Habilitado
    - **Databricks Runtime**: 16.4-LTS
    - **Usa a Aceleração do Photon**: <u>Não</u> selecionado
    - **Tipo de trabalho**: Standard_D4ds_v5
    - **Nó único**: Marcado

1. Aguarde a criação do cluster. Isso pode levar alguns minutos.

> **Observação**: se o cluster não for iniciado, sua assinatura pode ter cota insuficiente na região onde seu workspace do Azure Databricks está provisionado. Consulte [Limite de núcleo da CPU impede a criação do cluster](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit) para obter detalhes. Se isso acontecer, você pode tentar excluir seu workspace e criar um novo workspace em uma região diferente. Você pode especificar uma região como um parâmetro para o script de instalação da seguinte maneira: `./mslearn-databricks/setup.ps1 eastus`

## Instalar as bibliotecas necessárias

1. Na página do cluster, selecione a guia **Bibliotecas**.

2. Selecione **Instalar novo**.

3. Selecione **PyPI** como a origem da biblioteca e digite `transformers==4.53.0` no campo **pacote**.

4. Selecione **Instalar**.

5. Repita as etapas acima para instalar `databricks-vectorsearch==0.56`também.
   
## Criar um notebook e ingerir dados

1. Na barra lateral, use o link **(+) Novo** para criar um **Notebook**. Na lista suspensa **Conectar**, selecione o cluster caso ainda não esteja selecionado. Se o cluster não executar, é porque ele pode levar cerca de um minuto para iniciar.

1. Na primeira célula do notebook, insira a seguinte consulta SQL para criar um novo volume que será usado para armazenar os dados deste exercício dentro do seu catálogo padrão:

    ```python
   %sql 
   CREATE VOLUME <catalog_name>.default.RAG_lab;
    ```

1. Substitua `<catalog_name>` pelo nome do workspace, pois o Azure Databricks cria automaticamente um catálogo padrão com esse nome.
1. Use a opção de menu **&#9656; Executar Célula** à esquerda da célula para executá-la. Em seguida, aguarde o término do trabalho do Spark executado pelo código.
1. Em uma nova célula, execute o código a seguir que usa um comando *shell* para baixar dados do GitHub para o seu catálogo do Unity.

    ```python
   %sh
   wget -O /Volumes/<catalog_name>/default/RAG_lab/enwiki-latest-pages-articles.xml https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/enwiki-latest-pages-articles.xml
    ```

1. Em uma nova célula, execute o seguinte código para criar um dataframe a partir dos dados brutos:

    ```python
   from pyspark.sql import SparkSession

   # Create a Spark session
   spark = SparkSession.builder \
       .appName("RAG-DataPrep") \
       .getOrCreate()

   # Read the XML file
   raw_df = spark.read.format("xml") \
       .option("rowTag", "page") \
       .load("/Volumes/<catalog_name>/default/RAG_lab/enwiki-latest-pages-articles.xml")

   # Show the DataFrame
   raw_df.show(5)

   # Print the schema of the DataFrame
   raw_df.printSchema()
    ```

1. Em uma nova célula, execute o seguinte código, substituindo `<catalog_name>` pelo nome do catálogo do Unity, para limpar e pré-processar os dados e extrair os campos de texto relevantes:

    ```python
   from pyspark.sql.functions import col

   clean_df = raw_df.select(col("title"), col("revision.text._VALUE").alias("text"))
   clean_df = clean_df.na.drop()
   clean_df.write.format("delta").mode("overwrite").saveAsTable("<catalog_name>.default.wiki_pages")
   clean_df.show(5)
    ```

    Se você abrir o explorador de **Catálogo (CTRL + Alt + C)** e atualizar seu painel, verá a tabela Delta criada em seu catálogo padrão do Unity.

## Gerar incorporações e implementar a busca em vetores

O Mosaic AI Vector Search do Databricks é uma solução de banco de dados vetorial integrada à Plataforma Azure Databricks. Ele otimiza o armazenamento e a recuperação de incorporações utilizando o algoritmo HNSW (Hierarchical Navigable Small World). Ele permite pesquisas eficientes de vizinhos mais próximos, e seu recurso de pesquisa híbrida de similaridade de palavras-chave fornece resultados mais relevantes combinando técnicas de pesquisa baseadas em vetores e palavras-chave.

1. Em uma nova célula, execute a consulta SQL a seguir para habilitar o recurso Alterar Feed de Dados na tabela de origem antes de criar um índice de sincronização delta.

    ```python
   %sql
   ALTER TABLE <catalog_name>.default.wiki_pages SET TBLPROPERTIES (delta.enableChangeDataFeed = true)
    ```

2. Em uma nova célula, execute o código a seguir para criar o índice de busca em vetores.

    ```python
   from databricks.vector_search.client import VectorSearchClient

   client = VectorSearchClient()

   client.create_endpoint(
       name="vector_search_endpoint",
       endpoint_type="STANDARD"
   )

   index = client.create_delta_sync_index(
     endpoint_name="vector_search_endpoint",
     source_table_name="<catalog_name>.default.wiki_pages",
     index_name="<catalog_name>.default.wiki_index",
     pipeline_type="TRIGGERED",
     primary_key="title",
     embedding_source_column="text",
     embedding_model_endpoint_name="databricks-gte-large-en"
    )
    ```
     
Se você abrir o explorador de **Catálogo (CTRL + Alt + C)** e atualizar seu painel, verá o índice criado em seu catálogo padrão do Unity.

> **Observação:** antes de executar a próxima célula de código, verifique se o índice foi criado com êxito. Para fazer isso, clique com o botão direito do mouse no índice no painel Catálogo e selecione **Abrir no Explorador do Catálogo**. Aguarde até que o status do índice seja **Online**.

3. Em uma nova célula, execute o código a seguir para pesquisar documentos relevantes com base em um vetor de consulta.

    ```python
   results_dict=index.similarity_search(
       query_text="Anthropology fields",
       columns=["title", "text"],
       num_results=1
   )

   display(results_dict)
    ```

Verifique se a saída localiza a página Wiki correspondente relacionada ao prompt de consulta.

## Aumentar prompts com dados recuperados

Agora podemos aprimorar os recursos de grandes modelos de linguagem, fornecendo-lhes contexto adicional de fontes de dados externos. Ao fazer isso, os modelos podem gerar respostas mais precisas e contextualmente relevantes.

1. Em uma nova célula, execute o código a seguir para combinar os dados recuperados com a consulta do usuário para criar um prompt avançado para o LLM.

    ```python
   # Convert the dictionary to a DataFrame
   results = spark.createDataFrame([results_dict['result']['data_array'][0]])

   from transformers import pipeline

   # Load the summarization model
   summarizer = pipeline("summarization", model="facebook/bart-large-cnn", framework="pt")

   # Extract the string values from the DataFrame column
   text_data = results.select("_2").rdd.flatMap(lambda x: x).collect()

   # Pass the extracted text data to the summarizer function
   summary = summarizer(text_data, max_length=512, min_length=100, do_sample=True)

   def augment_prompt(query_text):
       context = " ".join([item['summary_text'] for item in summary])
       return f"Query: {query_text}\nContext: {context}"

   prompt = augment_prompt("Explain the significance of Anthropology")
   print(prompt)
    ```

3. Em uma nova célula, execute o código a seguir para usar um LLM para gerar respostas.

    ```python
   from transformers import GPT2LMHeadModel, GPT2Tokenizer

   tokenizer = GPT2Tokenizer.from_pretrained("gpt2")
   model = GPT2LMHeadModel.from_pretrained("gpt2")

   inputs = tokenizer(prompt, return_tensors="pt")
   outputs = model.generate(
       inputs["input_ids"], 
       max_length=300, 
       num_return_sequences=1, 
       repetition_penalty=2.0, 
       top_k=50, 
       top_p=0.95, 
       temperature=0.7,
       do_sample=True
   )
   response = tokenizer.decode(outputs[0], skip_special_tokens=True)

   print(response)
    ```

## Limpar

No portal do Azure Databricks, na página **Computação**, selecione seu cluster e selecione **&#9632; Terminar** para encerrar o processo.

Se você tiver terminado de explorar o Azure Databricks, poderá excluir os recursos que criou para evitar custos desnecessários do Azure e liberar capacidade em sua assinatura.
