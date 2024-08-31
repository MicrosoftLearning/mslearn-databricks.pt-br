# Exercício 04 – Ajustar modelos de linguagem grande usando o Azure Databricks e o OpenAI do Azure

## Objetivo
Este exercício orientará você pelo processo de ajuste fino de um LLM (modelo de linguagem grande) usando o Azure Databricks e o OpenAI do Azure. Você aprenderá como configurar o ambiente, pré-processar dados e ajustar um LLM em dados personalizados para realizar tarefas específicas de NLP.

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

## Etapa 3: Instalar as bibliotecas necessárias
- Na guia "Bibliotecas" do seu cluster, clique em "Instalar novo".
- Instale os seguintes pacotes do Python:
    1. transformers
    2. conjuntos de dados
    3. azure-ai-openai
- Opcionalmente, você também pode instalar quaisquer outros pacotes necessários, como torch.

### Criar novo notebook
- Vá para a seção "Espaço de trabalho" e clique em "Criar" > "Notebook".
- Nomeie seu notebook (por exemplo, Ajuste-Fino-GPT4) e escolha Python como a linguagem padrão.
- Anexe o notebook ao seu cluster.

## Etapa 4 – Preparar o conjunto de dados

- Carregar o conjunto de dados
    1. Você pode usar qualquer conjunto de dados de texto adequado para sua tarefa de ajuste fino. Por exemplo, vamos usar o conjunto de dados do IMDB para análise de sentimento.
    2. Em seu notebook, execute o código a seguir para carregar o conjunto de dados

    ```python
    from datasets import load_dataset

    dataset = load_dataset("imdb")
    ```

- Pré-processar o conjunto de dados
    1. Tokenize os dados de texto usando o tokenizador da biblioteca de transformadores.
    2. No seu notebook, adicione o código a seguir:

    ```python
    from transformers import GPT2Tokenizer

    tokenizer = GPT2Tokenizer.from_pretrained("gpt2")

    def tokenize_function(examples):
        return tokenizer(examples["text"], padding="max_length", truncation=True)

    tokenized_datasets = dataset.map(tokenize_function, batched=True)
    ```

- Preparar dados para ajuste fino
    1. Divida os dados em conjuntos de treinamento e validação.
    2. Em seu notebook, adicione:

    ```python
    small_train_dataset = tokenized_datasets["train"].shuffle(seed=42).select(range(1000))
    small_eval_dataset = tokenized_datasets["test"].shuffle(seed=42).select(range(500))
    ```

## Etapa 5 – Ajustar o modelo GPT-4

- Configurar a API OpenAI
    1. Você precisará da chave de API e do ponto de extremidade do OpenAI do Azure.
    2. Em seu notebook, configure as credenciais da API:

    ```python
    import openai

    openai.api_type = "azure"
    openai.api_key = "YOUR_AZURE_OPENAI_API_KEY"
    openai.api_base = "YOUR_AZURE_OPENAI_ENDPOINT"
    openai.api_version = "2023-05-15"
    ```
- Ajustar um modelo
    1. O ajuste fino do GPT-4 é realizado ajustando os hiperparâmetros e continuando o processo de treinamento em seu conjunto de dados específico.
    2. O ajuste fino pode ser mais complexo e pode exigir dados em lote, personalização de loops de treinamento, etc.
    3. Use as informações a seguir como um modelo básico:

    ```python
    from transformers import GPT2LMHeadModel, Trainer, TrainingArguments

    model = GPT2LMHeadModel.from_pretrained("gpt2")

    training_args = TrainingArguments(
        output_dir="./results",
        evaluation_strategy="epoch",
        learning_rate=2e-5,
        per_device_train_batch_size=2,
        per_device_eval_batch_size=2,
        num_train_epochs=3,
        weight_decay=0.01,
    )

    trainer = Trainer(
        model=model,
        args=training_args,
        train_dataset=small_train_dataset,
        eval_dataset=small_eval_dataset,
    )

    trainer.train()
    ```
    4. Este código fornece uma estrutura básica para treinamento. Os parâmetros e conjuntos de dados precisariam ser adaptados para casos específicos.

- Monitorar o processo de treinamento
    1. O Databricks permite monitorar o processo de treinamento por meio da interface do notebook e de ferramentas integradas como o MLflow para acompanhamento.

## Etapa 6: Avaliar o modelo ajustado

- Gerar previsões
    1. Após o ajuste fino, gere previsões no conjunto de dados de avaliação.
    2. Em seu notebook, adicione:

    ```python
    predictions = trainer.predict(small_eval_dataset)
    print(predictions)
    ```

- Avaliar desempenho do modelo
    1. Use métricas como precisão, recall e pontuação F1 para avaliar o modelo.
    2. Exemplo:

    ```python
    from sklearn.metrics import accuracy_score

    preds = predictions.predictions.argmax(-1)
    labels = predictions.label_ids
    accuracy = accuracy_score(labels, preds)
    print(f"Accuracy: {accuracy}")
    ```

- Salvar o modelo ajustado
    1. Salve o modelo ajustado em seu ambiente do Azure Databricks ou no armazenamento do Azure para uso futuro.
    2. Exemplo:

    ```python
    model.save_pretrained("/dbfs/mnt/fine-tuned-gpt4/")
    ```

## Etapa 7: Implantação do modelo ajustado
- Empacotar o modelo para implantação
    1. Converta o modelo em um formato compatível com o OpenAI do Azure ou outro serviço de implantação.

- Implantar o modelo
    1. Use o OpenAI do Azure para implantação registrando o modelo por meio do Azure Machine Learning ou diretamente com o ponto de extremidade do OpenAI.

- Testar o modelo implantado
    1. Execute testes para garantir que o modelo implantado se comporte conforme o esperado e se integre perfeitamente aos aplicativos.

## Etapa 8: Limpar os recursos
- Encerrar o cluster:
    1. Volte para a página "Computação", selecione seu cluster e clique em "Encerrar" para interromper o cluster.

- Opcional: exclua o serviço Databricks:
    1. Para evitar cobranças adicionais, considere excluir o workspace do Databricks se esse laboratório não fizer parte de um projeto ou roteiro de aprendizagem maior.

Este exercício forneceu um guia abrangente sobre como ajustar modelos de linguagem grande, como GPT-4, usando o Azure Databricks e o OpenAI do Azure. Seguindo essas etapas, você poderá ajustar modelos para tarefas específicas, avaliar seu desempenho e implantá-los para aplicativos do mundo real.

