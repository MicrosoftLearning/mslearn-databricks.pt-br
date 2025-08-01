---
lab:
  title: Explorar modelos de linguagem grandes com o Azure Databricks
---

# Explorar modelos de linguagem grandes com o Azure Databricks

Os LLMs (Modelos de linguagem grande) podem ser um ativo poderoso para tarefas de NLP (processamento de linguagem natural) quando integrados ao Azure Databricks e ao Hugging Face Transformers. O Azure Databricks fornece uma plataforma perfeita para acessar, ajustar e implantar LLMs, incluindo modelos pré-treinados da extensa biblioteca da Hugging Face. Para inferência de modelo, a classe pipelines da Hugging Face simplifica o uso de modelos pré-treinados, aceitando uma ampla variedade de tarefas de NLP diretamente no ambiente do Databricks.

Este laboratório levará aproximadamente **30** minutos para ser concluído.

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

## Carregar modelos pré-treinado

1. No workspace do Databricks, vá para a seção **Espaço de trabalho**.

2. Selecione **Criar** e, em seguida, selecione **Notebook**.

3. Dê um nome ao seu notebook e verifique se `Python` está selecionado como linguagem.

4. No menu suspenso **Conectar**, selecione o recurso de computação criado anteriormente.

5. Na primeira célula do código, insira e execute o código a seguir:

    ```python
   from transformers import pipeline

   # Load the summarization model with PyTorch weights
   summarizer = pipeline("summarization", model="facebook/bart-large-cnn", framework="pt")

   # Load the sentiment analysis model
   sentiment_analyzer = pipeline("sentiment-analysis", model="distilbert/distilbert-base-uncased-finetuned-sst-2-english", revision="714eb0f")

   # Load the translation model
   translator = pipeline("translation_en_to_fr", model="google-t5/t5-base", revision="a9723ea")

   # Load a general purpose model for zero-shot classification and few-shot learning
   classifier = pipeline("zero-shot-classification", model="facebook/bart-large-mnli", revision="d7645e1") 
    ```
     
    Isso carregará todos os modelos necessários para as tarefas de PNL apresentadas neste exercício.

### Resumir texto

Um pipeline de resumo gera resumos concisos de textos mais longos. Ao especificar um intervalo de comprimento (`min_length`, `max_length`) e se ele usará amostragem ou não (`do_sample`), podemos determinar o quão preciso ou criativo será o resumo gerado. 

1. Na nova célula de código, insira o código a seguir:

     ```python
    text = "Large language models (LLMs) are advanced AI systems capable of understanding and generating human-like text by learning from vast datasets. These models, which include OpenAI's GPT series and Google's BERT, have transformed the field of natural language processing (NLP). They are designed to perform a wide range of tasks, from translation and summarization to question-answering and creative writing. The development of LLMs has been a significant milestone in AI, enabling machines to handle complex language tasks with increasing sophistication. As they evolve, LLMs continue to push the boundaries of what's possible in machine learning and artificial intelligence, offering exciting prospects for the future of technology."
    summary = summarizer(text, max_length=75, min_length=25, do_sample=False)
    print(summary)
     ```

2. Execute a célula para ver o texto resumido.

### Analisar sentimento

O pipeline de análise de sentimento determina o sentimento de um determinado texto. Ele classifica o texto em categorias como positivo, negativo ou neutro.

1. Na nova célula de código, insira o código a seguir:

     ```python
    text = "I love using Azure Databricks for NLP tasks!"
    sentiment = sentiment_analyzer(text)
    print(sentiment)
     ```

2. Execute a célula para ver o resultado da análise de sentimento.

### Traduzir o texto

O pipeline de tradução converte o texto de um idioma para outro. Neste exercício, a tarefa usada foi `translation_en_to_fr`, o que significa que ele traduzirá qualquer texto do inglês para o francês.

1. Na nova célula de código, insira o código a seguir:

     ```python
    text = "Hello, how are you?"
    translation = translator(text)
    print(translation)
     ```

2. Execute a célula para ver o texto traduzido em francês.

### Classificar textos

O pipeline de classificação zero-shot permite que um modelo classifique o texto em categorias que não viu durante o treinamento. Portanto, requer rótulos predefinidos como um parâmetro `candidate_labels`.

1. Na nova célula de código, insira o código a seguir:

     ```python
    text = "Azure Databricks is a powerful platform for big data analytics."
    labels = ["technology", "health", "finance"]
    classification = classifier(text, candidate_labels=labels)
    print(classification)
     ```

2. Execute a célula para ver os resultados da classificação zero-shot.

## Limpar

No portal do Azure Databricks, na página **Computação**, selecione seu cluster e selecione **&#9632; Terminar** para encerrar o processo.

Se você tiver terminado de explorar o Azure Databricks, poderá excluir os recursos que criou para evitar custos desnecessários do Azure e liberar capacidade em sua assinatura.
