# Exercício 05 – Avaliar modelos de linguagem grandes usando o Azure Databricks e o OpenAI do Azure

## Objetivo
Neste exercício, você aprenderá a avaliar LLMs (modelos de linguagem grande) usando o Azure Databricks e o modelo GPT-4 do OpenAI. Isso inclui configurar o ambiente, definir métricas de avaliação e analisar o desempenho do modelo em tarefas específicas.

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

- Faça login no workspace do Azure Databricks.
- Crie um novo notebook e selecione o cluster padrão.
- Execeute os comandos a seguir para instalar as bibliotecas necessárias do Python:

```python
%pip install openai
%pip install transformers
%pip install datasets
```

- Configurar chave de API OpenAI:
    1. Adicione sua chave de API do OpenAI do Azure ao notebook:

    ```python
    import openai
    openai.api_key = "your-openai-api-key"
    ```

## Etapa 4: Definir métricas de avaliação
- Defina métricas de avaliação comuns:
    1. Nesta etapa, você definirá métricas de avaliação, como Perplexidade, pontuação BLEU, pontuação ROUGE e precisão, dependendo da tarefa.

    ```python
    from datasets import load_metric

    # Example: Load BLEU metric
    bleu_metric = load_metric("bleu")
    rouge_metric = load_metric("rouge")

    def compute_bleu(predictions, references):
        return bleu_metric.compute(predictions=predictions, references=references)

    def compute_rouge(predictions, references):
        return rouge_metric.compute(predictions=predictions, references=references)
    ```

- Defina métricas específicas da tarefa:
    1. Dependendo do caso de uso, defina outras métricas relevantes. Por exemplo, para análise de sentimento, defina precisão:

    ```python
    from sklearn.metrics import accuracy_score

    def compute_accuracy(predictions, references):
        return accuracy_score(references, predictions)
    ```

## Etapa 5: preparar o conjunto de dados
- Carregar um conjunto de dados
    1. Use a biblioteca de conjuntos de dados para carregar um conjunto de dados predefinido. Para este laboratório, você pode usar um conjunto de dados simples, como o conjunto de dados de resenhas de filmes do IMDB para análise de sentimento:

    ```python
    from datasets import load_dataset

    dataset = load_dataset("imdb")
    test_data = dataset["test"]
    ```

- Pré-processar os dados
    1. Tokenize e pré-processe o conjunto de dados para ser compatível com o modelo GPT-4:

    ```python
    from transformers import GPT2Tokenizer

    tokenizer = GPT2Tokenizer.from_pretrained("gpt2")

    def preprocess_function(examples):
        return tokenizer(examples["text"], truncation=True, padding=True)

    tokenized_data = test_data.map(preprocess_function, batched=True)
    ```

## Etapa 6: Avaliar o modelo GPT-4
- Gerar previsões:
    1. Use o modelo GPT-4 para gerar previsões para o conjunto de dados de teste

    ```python
    def generate_predictions(input_texts):
    predictions = []
    for text in input_texts:
        response = openai.Completion.create(
            model="gpt-4",
            prompt=text,
            max_tokens=50
        )
        predictions.append(response.choices[0].text.strip())
    return predictions

    input_texts = tokenized_data["text"]
    predictions = generate_predictions(input_texts)
    ```

- Computar métricas de avaliação
    1. Calcular as métricas de avaliação com base nas previsões geradas pelo modelo GPT-4

    ```python
    # Example: Compute BLEU and ROUGE scores
    bleu_score = compute_bleu(predictions, tokenized_data["text"])
    rouge_score = compute_rouge(predictions, tokenized_data["text"])

    print("BLEU Score:", bleu_score)
    print("ROUGE Score:", rouge_score)
    ```

    2. Se você estiver avaliando uma tarefa específica, como análise de sentimento, calcule a precisão

    ```python
    # Assuming binary sentiment labels (positive/negative)
    actual_labels = test_data["label"]
    predicted_labels = [1 if "positive" in pred else 0 for pred in predictions]

    accuracy = compute_accuracy(predicted_labels, actual_labels)
    print("Accuracy:", accuracy)
    ```

## Etapa 7: Analisar e interpretar os resultados

- Interpretar os resultados
    1. Analise as pontuações BLEU, ROUGE ou de precisão para determinar o desempenho do modelo GPT-4 em sua tarefa.
    2. Discuta os possíveis motivos para quaisquer discrepâncias e considere maneiras de melhorar o desempenho do modelo (por exemplo, ajuste fino, mais pré-processamento de dados).

- Visualizar os resultados
    1. Opcionalmente, você pode visualizar os resultados usando Matplotlib ou qualquer outra ferramenta de visualização.

    ```python
    import matplotlib.pyplot as plt

    # Example: Plot accuracy scores
    plt.bar(["Accuracy"], [accuracy])
    plt.ylabel("Score")
    plt.title("Model Evaluation Metrics")
    plt.show()
    ```

## Etapa 8: Experimentar diferentes cenários

- Experimentar diferentes prompts
    1. Modifique a estrutura do prompt para ver como ela afeta o desempenho do modelo.

- Avaliar em diferentes conjuntos de dados
    1. Tente usar um conjunto de dados diferente para avaliar a versatilidade do modelo GPT-4 em várias tarefas.

- Otimizar as métricas de avaliação
    1. Experimente hiperparâmetros como temperatura, tokens máximos, etc., para otimizar as métricas de avaliação.

## Etapa 9: Limpar os recursos
- Encerrar o cluster:
    1. Volte para a página "Computação", selecione seu cluster e clique em "Encerrar" para interromper o cluster.

- Opcional: exclua o serviço Databricks:
    1. Para evitar cobranças adicionais, considere excluir o workspace do Databricks se esse laboratório não fizer parte de um projeto ou roteiro de aprendizagem maior.

Este exercício orienta você pelo processo de avaliação de um grande modelo de linguagem usando o Azure Databricks e o modelo GPT-4 do OpenAI. Ao concluir este exercício, você obterá insights sobre o desempenho do modelo e entenderá como melhorar e ajustar o modelo para tarefas específicas.