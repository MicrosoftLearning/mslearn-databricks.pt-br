# Exercício 03 – Raciocínio em vários estágios com LangChain usando Azure Databricks e GPT-4

## Objetivo
Este exercício tem como objetivo orientar você na criação de um sistema de raciocínio de vários estágios usando o LangChain no Azure Databricks. Você aprenderá como criar um índice vetorial, armazenar inserções, criar uma cadeia com base em recuperador, construir uma cadeia de geração de imagens e, finalmente, combiná-las em um sistema de várias cadeias usando o modelo GPT-4 OpenAI.

## Requisitos
Uma assinatura ativa do Azure. Se você não tiver uma, poderá se inscrever para uma [avaliação gratuita](https://azure.microsoft.com/en-us/free/).

## Etapa 1: Provisionar o Azure Databricks
- Fazer logon no portal do Azure:
    1. Vá para o portal do Azure e entre com suas credenciais.
- Criar serviço do Databricks:
    1. Navegue até "Criar um recurso" > "Análise" > "Azure Databricks".
    2. Insira os detalhes necessários, como nome do espaço de trabalho, assinatura, grupo de recursos (crie um novo ou selecione um já existente) e local.
    3. Selecione o tipo de preço (escolha padrão para este laboratório).
    4. Clique em "Examinar + criar" e "Criar" depois que a validação for aprovada.

## Etapa 2: Iniciar o espaço de trabalho e criar um cluster
- Iniciar o workspace do Databricks:
    1. Quando a implantação estiver concluída, vá para o recurso e clique em "Iniciar espaço de trabalho".
- Criar um Cluster do Spark:
    1. No workspace do Databricks, clique em "Computação" na barra lateral e, em seguida, em "Criar computação".
    2. Especifique o nome do cluster e selecione uma versão de runtime do Spark.
    3. Escolha o tipo de trabalhador como "Padrão" e o tipo de nó com base nas opções disponíveis (escolha nós menores para eficiência de custo).
    4. Clique em "Criar computação".

## Etapa 3: Instalar bibliotecas necessárias

- Abra um novo notebook em seu espaço de trabalho.
- Instale as bibliotecas necessárias usando os seguintes comandos:

```python
%pip install langchain openai faiss-cpu
```

- Configurar a API OpenAI

```python
import os
os.environ["OPENAI_API_KEY"] = "your-openai-api-key"
```

## Etapa 4: Criar um índice vetorial e armazenar incorporações

- Carregar o conjunto de dados
    1. Carregue um conjunto de dados de amostra para o qual você deseja gerar incorporações. Para este laboratório, usaremos um pequeno conjunto de dados de texto.

    ```python
    sample_texts = [
        "Azure Databricks is a fast, easy, and collaborative Apache Spark-based analytics platform.",
        "LangChain is a framework designed to simplify the creation of applications using large language models.",
        "GPT-4 is a powerful language model developed by OpenAI."
    ]
    ```
- Gerar incorporações
    1. Use o modelo OpenAI GPT-4 para gerar incorporações para esses textos.

    ```python
    from langchain.embeddings.openai import OpenAIEmbeddings

    embeddings_model = OpenAIEmbeddings()
    embeddings = embeddings_model.embed_documents(sample_texts)
    ``` 

- Armazene as incorporações usando FAISS
    1. Use FAISS para criar um índice vetorial para recuperação eficiente.

    ```python
    import faiss
    import numpy as np

    dimension = len(embeddings[0])
    index = faiss.IndexFlatL2(dimension)
    index.add(np.array(embeddings))
    ```

## Etapa 5: Criar uma cadeia com base no recuperador
- Definir um recuperador
    1. Crie um recuperador que possa pesquisar no índice vetorial os textos mais semelhantes.

    ```python
    from langchain.chains import RetrievalQA
    from langchain.vectorstores.faiss import FAISS

    vector_store = FAISS(index, embeddings_model)
    retriever = vector_store.as_retriever()  
    ```

- Criar a cadeia RetrievalQA
    1. Crie um sistema de controle de qualidade usando o recuperador e o modelo GPT-4.
    
    ```python
    from langchain.llms import OpenAI
    from langchain.chains.question_answering import load_qa_chain

    llm = OpenAI(model_name="gpt-4")
    qa_chain = load_qa_chain(llm, retriever)
    ```

- Testar o sistema de QA
    1. Faça uma pergunta relacionada aos textos que você incorporou

    ```python
    result = qa_chain.run("What is Azure Databricks?")
    print(result)
    ```

## Etapa 6: Criar uma cadeia de geração de imagens

- Configurar o Modelo de geração de imagens
    1. Configure os recursos de geração de imagens usando GPT-4.

    ```python
    from langchain.chains import SimpleChain

    def generate_image(prompt):
        # Assuming you have an endpoint or a tool to generate images from text.
        return f"Generated image for prompt: {prompt}"

    image_generation_chain = SimpleChain(input_variables=["prompt"], output_variables=["image"], transform=generate_image)
    ```

- Testar a cadeia de geração de imagens
    1. Gere uma imagem com base em um prompt de texto.

    ```python
    prompt = "A futuristic city with flying cars"
    image_result = image_generation_chain.run(prompt=prompt)
    print(image_result)
    ```

## Etapa 7: Combine correntes em um sistema de várias cadeias
- Combinar correntes
    1. Integre a cadeia de QA com base no recuperador e a cadeia de geração de imagens em um sistema de várias cadeias.

    ```python
    from langchain.chains import MultiChain

    multi_chain = MultiChain(
        chains=[
            {"name": "qa", "chain": qa_chain},
            {"name": "image_generation", "chain": image_generation_chain}
        ]
    )
    ```

- Executar o sistema de várias cadeias
    1. Passe uma tarefa que envolva recuperação de texto e geração de imagens.

    ```python
    multi_task_input = {
        "qa": {"question": "Tell me about LangChain."},
        "image_generation": {"prompt": "A conceptual diagram of LangChain in use"}
    }

    multi_task_output = multi_chain.run(multi_task_input)
    print(multi_task_output)
    ```

## Etapa 8: Limpar os recursos
- Encerrar o cluster:
    1. Volte para a página "Computação", selecione seu cluster e clique em "Encerrar" para interromper o cluster.

- Opcional: exclua o serviço Databricks:
    1. Para evitar cobranças adicionais, considere excluir o workspace do Databricks se esse laboratório não fizer parte de um projeto ou roteiro de aprendizagem maior.

Isso conclui o exercício sobre raciocínio de vários etapas com o LangChain usando o Azure Databricks.