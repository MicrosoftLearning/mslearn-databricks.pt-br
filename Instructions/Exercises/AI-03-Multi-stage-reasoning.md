---
lab:
  title: Análise em vários estágios com LangChain usando o Azure Databricks e o Azure OpenAI
---

# Análise em vários estágios com LangChain usando o Azure Databricks e o Azure OpenAI

A análise em vários estágios é uma abordagem de ponta em IA que envolve a divisão de problemas complexos em estágios menores e mais gerenciáveis. O LangChain, uma estrutura de software, facilita a criação de aplicativos que utilizam modelos de linguagem grandes (LLMs). Quando integrado ao Azure Databricks, o LangChain permite o carregamento contínuo de dados, o encapsulamento de modelos e o desenvolvimento de agentes de IA sofisticados. Essa combinação é particularmente poderosa para lidar com tarefas complexas que demandam uma compreensão profunda do contexto e a capacidade de analisar em várias etapas.

Este laboratório levará aproximadamente **30** minutos para ser concluído.

## Antes de começar

É necessário ter uma [assinatura do Azure](https://azure.microsoft.com/free) com acesso de nível administrativo.

## Provisionar um recurso de OpenAI do Azure

Se ainda não tiver um, provisione um recurso OpenAI do Azure na sua assinatura do Azure.

1. Entre no **portal do Azure** em `https://portal.azure.com`.
2. Crie um recurso do **OpenAI do Azure** com as seguintes configurações:
    - **Assinatura**: *Selecione uma assinatura do Azure que tenha sido aprovada para acesso ao serviço Azure OpenAI*
    - **Grupo de recursos**: *escolher ou criar um grupo de recursos*
    - **Região**: *faça uma escolha **aleatória** de uma das regiões a seguir*\*
        - Leste da Austrália
        - Leste do Canadá
        - Leste dos EUA
        - Leste dos EUA 2
        - França Central
        - Leste do Japão
        - Centro-Norte dos EUA
        - Suécia Central
        - Norte da Suíça
        - Sul do Reino Unido
    - **Nome**: *um nome exclusivo de sua preferência*
    - **Tipo de preço**: Standard S0

> \* Os recursos do OpenAI do Azure são restritos por cotas regionais. As regiões listadas incluem a cota padrão para os tipos de modelos usados neste exercício. A escolha aleatória de uma região reduz o risco de uma só região atingir o limite de cota em cenários nos quais você compartilha uma assinatura com outros usuários. No caso de um limite de cota ser atingido mais adiante no exercício, há a possibilidade de você precisar criar outro recurso em uma região diferente.

3. Aguarde o fim da implantação. Em seguida, vá para o recurso OpenAI do Azure implantado no portal do Azure.

4. No painel esquerdo, em **Gerenciamento de recursos**, selecione **Chaves e Ponto de Extremidade**.

5. Copie o ponto de extremidade e uma das chaves disponíveis para usar posteriormente neste exercício.

## Implantar os modelos necessários

O Azure fornece um portal baseado na Web chamado **Estúdio de IA do Azure**, que você pode usar para implantar, gerenciar e explorar modelos. Você iniciará sua exploração do OpenAI do Azure usando o Estúdio de IA do Azure para implantar um modelo.

> **Observação**: À medida que você usa o Estúdio de IA do Azure, podem ser exibidas caixas de mensagens sugerindo tarefas para você executar. Você pode fechá-los e seguir as etapas desse exercício.

1. No portal do Azure, na página **Visão geral** do recurso OpenAI do Azure, role para baixo até a seção **Introdução** e clique no botão para abrir o **Estúdio de IA do Azure**.
   
1. No Estúdio de IA do Azure, no painel à esquerda, selecione a página **Implantações** e visualize as implantações de modelo existentes. Se você ainda não tiver uma implantação, crie uma nova implantação do modelo **gpt-35-turbo-16k** com as seguintes configurações:
    - **Nome da implantação**: *gpt-35-turbo-16k*
    - **Modelo**: gpt-35-turbo-16k *(se o modelo 16k não estiver disponível, escolha gpt-35-turbo e nomeie a implantação de acordo)*
    - **Versão do modelo**: *usar a versão padrão*
    - **Tipo de implantação**: Padrão
    - **Limite de taxa de tokens por minuto**: 5K\*
    - **Filtro de conteúdo**: Padrão
    - **Habilitar cota dinâmica**: Desabilitado
    
1. Volte para a página **Implantações** e crie uma nova implantação do modelo **text-embedding-ada-002** com as seguintes configurações:
    - **Nome da implantação**: *text-embedding-ada-002*
    - **Modelo**: text-embedding-ada-002
    - **Versão do modelo**: *usar a versão padrão*
    - **Tipo de implantação**: Padrão
    - **Limite de taxa de tokens por minuto**: 5K\*
    - **Filtro de conteúdo**: Padrão
    - **Habilitar cota dinâmica**: Desabilitado

> \* Um limite de taxa de 5.000 tokens por minuto é mais do que adequado para concluir este exercício, deixando capacidade para outras pessoas que usam a mesma assinatura.

## Provisionar um workspace do Azure Databricks

> **Dica**: Se você já tem um workspace do Azure Databricks, pode ignorar esse procedimento e usar o workspace existente.

1. Entre no **portal do Azure** em `https://portal.azure.com`.
2. Crie um recurso do **Azure Databricks** com as seguintes configurações:
    - **Assinatura**: *selecione a mesma assinatura do Azure usada para criar o recurso do OpenAI do Azure*
    - **Grupo de recursos**: *o grupo de recursos em que você criou o recurso do OpenAI do Azure*
    - **Região**: *a mesma região onde você criou seu recurso do OpenAI do Azure*
    - **Nome**: *um nome exclusivo de sua preferência*
    - **Tipo de preço**: *premium* ou *avaliação*

3. Selecione **Revisar + criar** e aguarde a conclusão da implantação. Em seguida, vá para o recurso e inicie o workspace.

## Criar um cluster

O Azure Databricks é uma plataforma de processamento distribuído que usa *clusters* do Apache Spark para processar dados em paralelo em vários nós. Cada cluster consiste em um nó de driver para coordenar o trabalho e nós de trabalho para executar tarefas de processamento. Neste exercício, você criará um cluster de *nó único* para minimizar os recursos de computação usados no ambiente de laboratório (no qual os recursos podem ser restritos). Em um ambiente de produção, você normalmente criaria um cluster com vários nós de trabalho.

> **Dica**: Se você já tiver um cluster com uma versão de runtime 13.3 LTS **<u>ML</u>** ou superior em seu workspace do Azure Databricks, poderá usá-lo para concluir este exercício e ignorar este procedimento.

1. No portal do Azure, navegue até o grupo de recursos em que o workspace do Azure Databricks foi criado.
2. Clique no recurso de serviço do Azure Databricks.
3. Na página **Visão geral** do seu workspace, use o botão **Iniciar workspace** para abrir seu workspace do Azure Databricks em uma nova guia do navegador, fazendo o logon se solicitado.

> **Dica**: ao usar o portal do workspace do Databricks, várias dicas e notificações podem ser exibidas. Dispense-as e siga as instruções fornecidas para concluir as tarefas neste exercício.

4. Na barra lateral à esquerda, selecione a tarefa **(+) Novo** e, em seguida, selecione **Cluster**.
5. Na página **Novo Cluster**, crie um novo cluster com as seguintes configurações:
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

6. Aguarde a criação do cluster. Isso pode levar alguns minutos.

> **Observação**: se o cluster não for iniciado, sua assinatura pode ter cota insuficiente na região onde seu workspace do Azure Databricks está provisionado. Consulte [Limite de núcleo da CPU impede a criação do cluster](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit) para obter detalhes. Se isso acontecer, você pode tentar excluir seu workspace e criar um novo workspace em uma região diferente.

## Instalar as bibliotecas necessárias

1. No workspace do Databricks, vá para a seção **Espaço de trabalho**.

2. Selecione **Criar** e, em seguida, selecione **Notebook**.

3. Dê um nome ao notebook e selecione `Python` como a linguagem.

4. Na primeira célula de código, insira e execute o seguinte código para instalar as bibliotecas necessárias:
   
     ```python
    %pip install langchain openai langchain_openai faiss-cpu
     ```

5. Após a conclusão da instalação, reinicie o kernel em uma nova célula:

     ```python
    %restart_python
     ```

6. Em uma nova célula, defina os parâmetros de autenticação que serão usados para inicializar os modelos do OpenAI, substituindo `your_openai_endpoint` e `your_openai_api_key` pelo endpoint e pela chave copiados anteriormente do seu recurso do OpenAI:

     ```python
    endpoint = "your_openai_endpoint"
    key = "your_openai_api_key"
     ```
     
## Criar um índice vetorial e armazenar incorporações

Um índice vetorial é uma estrutura de dados especializada que permite o armazenamento e a recuperação eficientes de dados vetoriais de alta dimensão, o que é crucial para realizar pesquisas rápidas de similaridade e consultas de vizinhos mais próximos. As incorporações, por outro lado, são representações numéricas de objetos que capturam seu significado em forma vetorial, permitindo que as máquinas processem e entendam vários tipos de dados, incluindo texto e imagens.

1. Em uma nova célula, execute o código a seguir para carregar um conjunto de dados de amostra:

     ```python
    from langchain_core.documents import Document

    documents = [
         Document(page_content="Azure Databricks is a fast, easy, and collaborative Apache Spark-based analytics platform.", metadata={"date_created": "2024-08-22"}),
         Document(page_content="LangChain is a framework designed to simplify the creation of applications using large language models.", metadata={"date_created": "2024-08-22"}),
         Document(page_content="GPT-4 is a powerful language model developed by OpenAI.", metadata={"date_created": "2024-08-22"})
    ]
    ids = ["1", "2", "3"]
     ```
     
1. Em uma nova célula, execute o seguinte código para gerar incorporações usando o modelo `text-embedding-ada-002`:

     ```python
    from langchain_openai import AzureOpenAIEmbeddings
     
    embedding_function = AzureOpenAIEmbeddings(
        deployment="text-embedding-ada-002",
        model="text-embedding-ada-002",
        azure_endpoint=endpoint,
        openai_api_key=key,
        chunk_size=1
    )
     ```
     
1. Em uma nova célula, execute o seguinte código para criar um índice de vetor usando a primeira amostra de texto como referência para a dimensão de vetor:

     ```python
    import faiss
      
    index = faiss.IndexFlatL2(len(embedding_function.embed_query("Azure Databricks is a fast, easy, and collaborative Apache Spark-based analytics platform.")))
     ```

## Criar uma cadeia baseada no recuperador

Um componente recuperador busca documentos ou dados relevantes com base em uma consulta. Isso é particularmente útil em aplicativos que exigem a integração de grandes quantidades de dados para análise, como em sistemas de geração aumentada por recuperação.

1. Em uma nova célula, execute o código a seguir para criar um recuperador que possa pesquisar o índice de vetor dos textos mais semelhantes.

     ```python
    from langchain.vectorstores import FAISS
    from langchain_core.vectorstores import VectorStoreRetriever
    from langchain_community.docstore.in_memory import InMemoryDocstore

    vector_store = FAISS(
        embedding_function=embedding_function,
        index=index,
        docstore=InMemoryDocstore(),
        index_to_docstore_id={}
    )
    vector_store.add_documents(documents=documents, ids=ids)
    retriever = VectorStoreRetriever(vectorstore=vector_store)
     ```

1. Em uma nova célula, execute o seguinte código para criar um sistema de controle de qualidade usando o recuperador e o modelo `gpt-35-turbo-16k`:
    
     ```python
    from langchain_openai import AzureChatOpenAI
    from langchain_core.prompts import ChatPromptTemplate
    from langchain.chains.combine_documents import create_stuff_documents_chain
    from langchain.chains import create_retrieval_chain
     
    llm = AzureChatOpenAI(
        deployment_name="gpt-35-turbo-16k",
        model_name="gpt-35-turbo-16k",
        azure_endpoint=endpoint,
        api_version="2023-03-15-preview",
        openai_api_key=key,
    )

    system_prompt = (
        "Use the given context to answer the question. "
        "If you don't know the answer, say you don't know. "
        "Use three sentences maximum and keep the answer concise. "
        "Context: {context}"
    )

    prompt1 = ChatPromptTemplate.from_messages([
        ("system", system_prompt),
        ("human", "{input}")
    ])

    chain = create_stuff_documents_chain(llm, prompt)

    qa_chain1 = create_retrieval_chain(retriever, chain)
     ```

1. Em uma nova célula, execute o seguinte código para testar o sistema de controle de qualidade:

     ```python
    result = qa_chain1.invoke({"input": "What is Azure Databricks?"})
    print(result)
     ```

A saída do resultado deve mostrar uma resposta com base no documento relevante presente no conjunto de dados de amostra mais o texto generativo produzido pelo LLM.

## Combine correntes em um sistema de várias cadeias

Langchain é uma ferramenta versátil que permite a combinação de várias cadeias em um sistema de várias cadeias, aprimorando os recursos dos modelos de linguagem. Esse processo envolve a junção de vários componentes que podem processar entradas em paralelo ou em sequência, sintetizando uma resposta final.

1. Em uma nova célula, execute o código a seguir para criar uma segunda cadeia

     ```python
    from langchain_core.prompts import ChatPromptTemplate
    from langchain_core.output_parsers import StrOutputParser

    prompt2 = ChatPromptTemplate.from_template("Create a social media post based on this summary: {summary}")

    qa_chain2 = ({"summary": qa_chain1} | prompt2 | llm | StrOutputParser())
     ```

1. Em uma nova célula, execute o seguinte código para chamar uma cadeia de vários estágios com uma determinada entrada:

     ```python
    result = qa_chain2.invoke({"input": "How can we use LangChain?"})
    print(result)
     ```

A primeira cadeia fornece uma resposta à entrada com base no conjunto de dados de amostra fornecido, enquanto a segunda cadeia cria uma postagem de mídia social com base na saída da primeira cadeia. Essa abordagem permite que você lide com tarefas de processamento de texto mais complexas encadeando várias etapas.

## Limpar

Quando terminar o recurso do OpenAI do Azure, lembre-se de excluir a implantação ou todo o recurso no **portal do Azure** em `https://portal.azure.com`.

No portal do Azure Databricks, na página **Computação**, selecione seu cluster e selecione **&#9632; Terminar** para encerrar o processo.

Se você tiver terminado de explorar o Azure Databricks, poderá excluir os recursos que criou para evitar custos desnecessários do Azure e liberar capacidade em sua assinatura.