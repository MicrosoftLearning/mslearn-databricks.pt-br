# Exercício 02 – Geração Aumentada de Recuperação usando o Azure Databricks

## Objetivo
Este exercício orienta você na configuração de um fluxo de trabalho de RAG (Geração Aumentada de Recuperação) no Azure Databricks. O processo envolve a ingestão de dados, a criação de incorporações vetoriais, o armazenamento dessas incorporações em um banco de dados vetorial e o uso delas para aumentar a entrada de um modelo generativo.

## Requisitos
Uma assinatura ativa do Azure. Se você não tiver uma, poderá se inscrever para uma [avaliação gratuita](https://azure.microsoft.com/en-us/free/).

## Tempo estimado: 40 minutos

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

## Etapa 3: Preparação de dados
- Ingerir Dados
    1. Baixe um conjunto de dados de amostra de artigos da Wikipédia [aqui](https://dumps.wikimedia.org/enwiki/latest/).
    2. Carregue o conjunto de dados no Azure Data Lake Storage ou diretamente no Sistema de Arquivos do Azure Databricks.

- Carregar dados no Azure Databricks
```python
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("RAG-DataPrep").getOrCreate()
raw_data_path = "/mnt/data/wiki_sample.json"  # Adjust the path as necessary

raw_df = spark.read.json(raw_data_path)
raw_df.show(5)
```

- Limpeza e pré-processamento de dados
    1. Limpe e pré-processe os dados para extrair campos de texto relevantes.

    ```python
    from pyspark.sql.functions import col

    clean_df = raw_df.select(col("title"), col("text"))
    clean_df = clean_df.na.drop()
    clean_df.show(5)
    ```
## Etapa 4: Geração de incorporações
- Instalar bibliotecas necessárias
    1. Certifique-se de ter as bibliotecas de transformadores e transformadores de sentença instaladas.

    ```python
    %pip install transformers sentence-transformers
    ```
- Gerar incorporações
    1. Use um modelo pré-treinado para gerar incorporações para o texto.

    ```python
    from sentence_transformers import SentenceTransformer

    model = SentenceTransformer('sentence-transformers/all-MiniLM-L6-v2')

    def embed_text(text):
        return model.encode(text).tolist()

    # Apply the embedding function to the dataset
    from pyspark.sql.functions import udf
    from pyspark.sql.types import ArrayType, FloatType

    embed_udf = udf(embed_text, ArrayType(FloatType()))
    embedded_df = clean_df.withColumn("embeddings", embed_udf(col("text")))
    embedded_df.show(5)
    ```

## Etapa 5: Armazenamento de incorporações
- Armazenar incorporações em tabelas Delta
    1. Salve os dados incorporados em uma tabela Delta para recuperação eficiente.

    ```python
    embedded_df.write.format("delta").mode("overwrite").save("/mnt/delta/wiki_embeddings")
    ```

    2. Criar uma tabela Delta

    ```python
    CREATE TABLE IF NOT EXISTS wiki_embeddings
     LOCATION '/mnt/delta/wiki_embeddings'
    ```
## Etapa 6: Implementação da busca em vetores
- Configurar busca em vetores
    1. Use os recursos de busca em vetores do Databricks ou integre-se a um banco de dados vetorial como Milvus ou Pinecone.

    ```python
    from databricks.feature_store import FeatureStoreClient

    fs = FeatureStoreClient()

    fs.create_table(
        name="wiki_embeddings_vector_store",
        primary_keys=["title"],
        df=embedded_df,
        description="Vector embeddings for Wikipedia articles."
    )
    ```
- Executar a busca em vetores
    1. Implemente uma função para pesquisar documentos relevantes com base em um vetor de consulta.

    ```python
    def search_vectors(query_text, top_k=5):
        query_embedding = model.encode([query_text]).tolist()
        query_df = spark.createDataFrame([(query_text, query_embedding)], ["query_text", "query_embedding"])
        
        results = fs.search_table(
            name="wiki_embeddings_vector_store",
            vector_column="embeddings",
            query=query_df,
            top_k=top_k
        )
        return results

    query = "Machine learning applications"
    search_results = search_vectors(query)
    search_results.show()
    ```

## Etapa 7: Aumento generativo
- Prompts de aumento com dados recuperados:
    1. Combine os dados recuperados com a consulta do usuário para criar um prompt avançado para o LLM.

    ```python
    def augment_prompt(query_text):
        search_results = search_vectors(query_text)
        context = " ".join(search_results.select("text").rdd.flatMap(lambda x: x).collect())
        return f"Query: {query_text}\nContext: {context}"

    prompt = augment_prompt("Explain the significance of the Turing test")
    print(prompt)
    ```

- Gerar respostas com LLM:
    2. Use um LLM como GPT-3 ou modelos semelhantes da Hugging Face para gerar respostas.

    ```python
    from transformers import GPT2LMHeadModel, GPT2Tokenizer

    tokenizer = GPT2Tokenizer.from_pretrained("gpt2")
    model = GPT2LMHeadModel.from_pretrained("gpt2")

    inputs = tokenizer(prompt, return_tensors="pt")
    outputs = model.generate(inputs["input_ids"], max_length=500, num_return_sequences=1)
    response = tokenizer.decode(outputs[0], skip_special_tokens=True)

    print(response)
    ```

## Etapa 8: Avaliação e otimização
- Avalie a qualidade das respostas geradas:
    1. Avalie a relevância, coerência e precisão das respostas geradas.
    2. Colete comentários do usuário e itere no processo de aumento de prompt.

- Otimizar o fluxo de trabalho RAG:
    1. Experimente diferentes modelos de incorporação, tamanhos de partes e parâmetros de recuperação para otimizar o desempenho.
    2. Monitore o desempenho do sistema e faça ajustes para melhorar a precisão e a eficiência.

## Etapa 9: Limpar os recursos
- Encerrar o cluster:
    1. Volte para a página "Computação", selecione seu cluster e clique em "Encerrar" para interromper o cluster.

- Opcional: exclua o serviço Databricks:
    1. Para evitar cobranças adicionais, considere excluir o workspace do Databricks se esse laboratório não fizer parte de um projeto ou roteiro de aprendizagem maior.

Seguindo estas etapas, você terá implementado um sistema RAG (Geração Aumentada de Recuperação) usando o Azure Databricks. Este laboratório demonstra como pré-processar dados, gerar incorporações, armazená-los com eficiência, realizar buscas em vetores e usar modelos generativos para criar respostas enriquecidas. A abordagem pode ser adaptada a vários domínios e conjuntos de dados para aprimorar os recursos dos aplicativos orientados por IA.