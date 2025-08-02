---
lab:
  title: Ajustar modelos de linguagem grande usando o Azure Databricks e o OpenAI do Azure
---

# Ajustar modelos de linguagem grande usando o Azure Databricks e o OpenAI do Azure

Com o Azure Databricks, os usuários agora podem usar o poder dos LLMs para tarefas especializadas ajustando-os com seus próprios dados, melhorando o desempenho específico do domínio. Para ajustar um modelo de linguagem usando o Azure Databricks, você pode utilizar a interface de Treinamento de Modelo de IA do Mosaic, que simplifica o processo de ajuste completo do modelo. Esse recurso permite que você ajuste um modelo com seus dados personalizados, com pontos de verificação salvos no MLflow, garantindo que você mantenha o controle total sobre o modelo ajustado.

Este laboratório levará aproximadamente **60** minutos para ser concluído.

> **Observação**: a interface do usuário do Azure Databricks está sujeita a melhorias contínuas. A interface do usuário pode ter sido alterada desde que as instruções neste exercício foram escritas.

## Antes de começar

É necessário ter uma [assinatura do Azure](https://azure.microsoft.com/free) com acesso de nível administrativo.

## Provisionar um recurso de OpenAI do Azure

Se ainda não tiver um, provisione um recurso OpenAI do Azure na sua assinatura do Azure.

1. Entre no **portal do Azure** em `https://portal.azure.com`.
2. Crie um recurso do **OpenAI do Azure** com as seguintes configurações:
    - **Assinatura**: *Selecione uma assinatura do Azure que tenha sido aprovada para acesso ao serviço Azure OpenAI*
    - **Grupo de recursos**: *escolher ou criar um grupo de recursos*
    - **Região**: *faça uma escolha **aleatória** de uma das regiões a seguir*\*
        - Leste dos EUA 2
        - Centro-Norte dos EUA
        - Suécia Central
        - Oeste da Suíça
    - **Nome**: *um nome exclusivo de sua preferência*
    - **Tipo de preço**: Standard S0

> \* Os recursos do OpenAI do Azure são restritos por cotas regionais. As regiões listadas incluem a cota padrão para os tipos de modelos usados neste exercício. A escolha aleatória de uma região reduz o risco de uma só região atingir o limite de cota em cenários nos quais você compartilha uma assinatura com outros usuários. No caso de um limite de cota ser atingido mais adiante no exercício, há a possibilidade de você precisar criar outro recurso em uma região diferente.

3. Aguarde o fim da implantação. Em seguida, vá para o recurso OpenAI do Azure implantado no portal do Azure.

4. No painel esquerdo, em **Gerenciamento de recursos**, selecione **Chaves e Ponto de Extremidade**.

5. Copie o ponto de extremidade e uma das chaves disponíveis para usar posteriormente neste exercício.

6. Inicie o Cloud Shell e execute `az account get-access-token` para receber um token de autorização temporário para teste de API. Mantenha-o junto com o ponto de extremidade e a chave copiados anteriormente.

    >**Observação**: Você só precisa copiar o valor do campo `accessToken` e **não** toda a saída JSON.

## Implantar o modelo necessário

O Azure fornece um portal baseado na Web chamado **Fábrica de IA do Azure**, que você pode usar para implantar, gerenciar e explorar modelos. Você irá iniciar a exploração do OpenAI do Azure usando o portal da Fábrica de IA do Azure para implantar um modelo.

> **Observação**: À medida que você usar o portal da Fábrica de IA do Azure, poderão ser exibidas caixas de mensagens sugerindo tarefas para você executar. Você pode fechá-los e seguir as etapas desse exercício.

1. No portal do Azure, na página **Visão geral** do recurso OpenAI do Azure, role para baixo até a seção **Introdução** e clique no botão para abrir a **Fábrica de IA do Azure**.
   
1. No portal da Fábrica de IA do Azure, no painel à esquerda, selecione a página **Implantações** e visualize as implantações de modelo existentes. Se você ainda não tiver uma, crie uma nova implantação do modelo **gpt-4o** com as seguintes configurações:
    - **Nome da implantação**: *gpt-4o*
    - **Tipo de implantação**: Padrão
    - **Versão do modelo**: *usar a versão padrão*
    - **Limite de taxa de tokens por minuto**: 10 MIL\*
    - **Filtro de conteúdo**: Padrão
    - **Habilitar cota dinâmica**: Desabilitado
    
> \* Um limite de 10.000 tokens por minuto é mais do que suficiente para concluir este exercício, mantendo capacidade disponível para outras pessoas que usam a mesma assinatura.

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

> **Dica**: Se você já tiver um cluster com a versão 16.4 LTS **<u>ML</u>** ou superior do runtime no workspace do Azure Databricks, poderá usá-lo para concluir este exercício e pular este procedimento.

1. No portal do Azure, navegue até o grupo de recursos em que o workspace do Azure Databricks foi criado.
2. Clique no recurso de serviço do Azure Databricks.
3. Na página **Visão geral** do seu workspace, use o botão **Iniciar workspace** para abrir seu workspace do Azure Databricks em uma nova guia do navegador, fazendo o logon se solicitado.

> **Dica**: ao usar o portal do workspace do Databricks, várias dicas e notificações podem ser exibidas. Dispense-as e siga as instruções fornecidas para concluir as tarefas neste exercício.

4. Na barra lateral à esquerda, selecione a tarefa **(+) Novo** e, em seguida, selecione **Cluster**.
5. Na página **Novo Cluster**, crie um novo cluster com as seguintes configurações:
    - **Nome do cluster**: cluster *Nome do Usuário* (o nome do cluster padrão)
    - **Política**: Sem restrições
    - **Machine learning**: Habilitado
    - **Databricks Runtime**: 16.4-LTS
    - **Usa a Aceleração do Photon**: <u>Não</u> selecionado
    - **Tipo de trabalho**: Standard_D4ds_v5
    - **Nó único**: Marcado

6. Aguarde a criação do cluster. Isso pode levar alguns minutos.

> **Observação**: se o cluster não for iniciado, sua assinatura pode ter cota insuficiente na região onde seu workspace do Azure Databricks está provisionado. Consulte [Limite de núcleo da CPU impede a criação do cluster](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit) para obter detalhes. Se isso acontecer, você pode tentar excluir seu workspace e criar um novo workspace em uma região diferente.

## Criar um notebook e ingerir dados

1. Na barra lateral, use o link **(+) Novo** para criar um **Notebook**. Na lista suspensa **Conectar**, selecione o cluster caso ainda não esteja selecionado. Se o cluster não executar, é porque ele pode levar cerca de um minuto para iniciar.

1. Na primeira célula do notebook, insira a seguinte consulta SQL para criar um novo volume que será usado para armazenar os dados deste exercício dentro do seu catálogo padrão:

    ```python
   %sql 
   CREATE VOLUME <catalog_name>.default.fine_tuning;
    ```

1. Substitua `<catalog_name>` pelo nome do seu catálogo padrão. Você pode verificar o nome selecionando **Catálogo** na barra lateral.
1. Use a opção de menu **&#9656; Executar Célula** à esquerda da célula para executá-la. Em seguida, aguarde o término do trabalho do Spark executado pelo código.
1. Em uma nova célula, execute o código a seguir que usa um comando *shell* para baixar dados do GitHub para o seu catálogo do Unity.

    ```python
   %sh
   wget -O /Volumes/<catalog_name>/default/fine_tuning/training_set.jsonl https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/training_set.jsonl
   wget -O /Volumes/<catalog_name>/default/fine_tuning/validation_set.jsonl https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/validation_set.jsonl
    ```

3. Em uma nova célula, execute o seguinte código com as informações de acesso copiadas no início deste exercício para atribuir variáveis de ambiente persistentes para autenticação ao usar recursos do OpenAI do Azure:

    ```python
   import os

   os.environ["AZURE_OPENAI_API_KEY"] = "your_openai_api_key"
   os.environ["AZURE_OPENAI_ENDPOINT"] = "your_openai_endpoint"
   os.environ["TEMP_AUTH_TOKEN"] = "your_access_token"
    ```
     
## Validar contagens de tokens

Ambos `training_set.jsonl` e `validation_set.jsonl` são feitos de diferentes exemplos de conversação entre `user` e `assistant` que servirão como pontos de dados para treinar e validar o modelo ajustado. Embora os conjuntos de dados deste exercício sejam considerados pequenos, é importante ter em mente, ao trabalhar com conjuntos de dados maiores, que os LLMs possuem um comprimento máximo de contexto em termos de tokens. Portanto, você pode verificar a contagem de tokens dos seus conjuntos de dados antes de treinar seu modelo e revisá-los, se necessário. 

1. Em uma nova célula, execute o seguinte código para validar as contagens de tokens para cada arquivo:

    ```python
   import json
   import tiktoken
   import numpy as np
   from collections import defaultdict

   encoding = tiktoken.get_encoding("cl100k_base")

   def num_tokens_from_messages(messages, tokens_per_message=3, tokens_per_name=1):
       num_tokens = 0
       for message in messages:
           num_tokens += tokens_per_message
           for key, value in message.items():
               num_tokens += len(encoding.encode(value))
               if key == "name":
                   num_tokens += tokens_per_name
       num_tokens += 3
       return num_tokens

   def num_assistant_tokens_from_messages(messages):
       num_tokens = 0
       for message in messages:
           if message["role"] == "assistant":
               num_tokens += len(encoding.encode(message["content"]))
       return num_tokens

   def print_distribution(values, name):
       print(f"\n##### Distribution of {name}:")
       print(f"min / max: {min(values)}, {max(values)}")
       print(f"mean / median: {np.mean(values)}, {np.median(values)}")

   files = ['/Volumes/<catalog_name>/default/fine_tuning/training_set.jsonl', '/Volumes/<catalog_name>/default/fine_tuning/validation_set.jsonl']

   for file in files:
       print(f"File: {file}")
       with open(file, 'r', encoding='utf-8') as f:
           dataset = [json.loads(line) for line in f]

       total_tokens = []
       assistant_tokens = []

       for ex in dataset:
           messages = ex.get("messages", {})
           total_tokens.append(num_tokens_from_messages(messages))
           assistant_tokens.append(num_assistant_tokens_from_messages(messages))

       print_distribution(total_tokens, "total tokens")
       print_distribution(assistant_tokens, "assistant tokens")
       print('*' * 75)
    ```

Como referência, o modelo usado neste exercício, o GPT-4o, possui um limite de contexto (número total de tokens no prompt de entrada e na resposta gerada combinados) de 128 mil tokens.

## Carregar arquivos de ajuste fino no OpenAI do Azure

Antes de começar a ajustar o modelo, você precisa inicializar um cliente OpenAI e adicionar os arquivos de ajuste fino ao respectivo ambiente, gerando IDs de arquivo que serão usadas para inicializar o trabalho.

1. Execute o código a seguir em uma nova célula:

     ```python
    import os
    from openai import AzureOpenAI

    client = AzureOpenAI(
      azure_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT"),
      api_key = os.getenv("AZURE_OPENAI_API_KEY"),
      api_version = "2024-05-01-preview"  # This API version or later is required to access seed/events/checkpoint features
    )

    training_file_name = '/Volumes/<catalog_name>/default/fine_tuning/training_set.jsonl'
    validation_file_name = '/Volumes/<catalog_name>/default/fine_tuning/validation_set.jsonl'

    training_response = client.files.create(
        file = open(training_file_name, "rb"), purpose="fine-tune"
    )
    training_file_id = training_response.id

    validation_response = client.files.create(
        file = open(validation_file_name, "rb"), purpose="fine-tune"
    )
    validation_file_id = validation_response.id

    print("Training file ID:", training_file_id)
    print("Validation file ID:", validation_file_id)
     ```

## Enviar trabalho de ajuste fino

Agora que os arquivos de ajuste fino foram carregados, você pode enviar seu trabalho de treinamento de ajuste fino. Não é incomum que o treinamento leve mais de uma hora para ser concluído. Após a conclusão do treinamento, você pode visualizar os resultados na Fábrica de IA do Azure selecionando a opção **Ajuste fino** no painel esquerdo.

1. Em uma nova célula, execute o seguinte código para iniciar o trabalho de treinamento de ajuste fino:

    ```python
   response = client.fine_tuning.jobs.create(
       training_file = training_file_id,
       validation_file = validation_file_id,
       model = "gpt-4o",
       seed = 105 # seed parameter controls reproducibility of the fine-tuning job. If no seed is specified one will be generated automatically.
   )

   job_id = response.id
    ```

O parâmetro `seed` controla a reprodutibilidade do trabalho de ajuste fino. Passar os mesmos parâmetros iniciais e de trabalho deve produzir os mesmos resultados, mas pode diferir em casos raros. Se nenhuma semente for especificada, uma será gerada automaticamente.

2. Em uma nova célula, é possível executar o seguinte código para monitorar o status do trabalho de ajuste fino:

    ```python
   print("Job ID:", response.id)
   print("Status:", response.status)
    ```

>**Observação**: Você também pode monitorar o status do trabalho na Fábrica de IA selecionando **Ajuste fino** na barra lateral esquerda.

3. Depois que o status do trabalho for alterado para `succeeded`, execute o seguinte código para obter os resultados finais:

    ```python
   response = client.fine_tuning.jobs.retrieve(job_id)

   print(response.model_dump_json(indent=2))
   fine_tuned_model = response.fine_tuned_model
    ```
   
## Implantar modelo ajustado

Agora que você possui um modelo ajustado, pode implantá-lo como um modelo personalizado e usá-lo como qualquer outro modelo implantado, seja no Playground de **Chat** da Fábrica de IA do Azure ou por meio da API de conclusão de chat.

1. Em uma nova célula, execute o seguinte código para implementar o modelo ajustado:
   
    ```python
   import json
   import requests

   token = os.getenv("TEMP_AUTH_TOKEN")
   subscription = "<YOUR_SUBSCRIPTION_ID>"
   resource_group = "<YOUR_RESOURCE_GROUP_NAME>"
   resource_name = "<YOUR_AZURE_OPENAI_RESOURCE_NAME>"
   model_deployment_name = "gpt-4o-ft"

   deploy_params = {'api-version': "2023-05-01"}
   deploy_headers = {'Authorization': 'Bearer {}'.format(token), 'Content-Type': 'application/json'}

   deploy_data = {
       "sku": {"name": "standard", "capacity": 1},
       "properties": {
           "model": {
               "format": "OpenAI",
               "name": "<YOUR_FINE_TUNED_MODEL>",
               "version": "1"
           }
       }
   }
   deploy_data = json.dumps(deploy_data)

   request_url = f'https://management.azure.com/subscriptions/{subscription}/resourceGroups/{resource_group}/providers/Microsoft.CognitiveServices/accounts/{resource_name}/deployments/{model_deployment_name}'

   print('Creating a new deployment...')

   r = requests.put(request_url, params=deploy_params, headers=deploy_headers, data=deploy_data)

   print(r)
   print(r.reason)
   print(r.json())
    ```

2. Em uma nova célula, execute o seguinte código para usar o modelo personalizado em uma chamada de conclusão de chat:
   
    ```python
   import os
   from openai import AzureOpenAI

   client = AzureOpenAI(
     azure_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT"),
     api_key = os.getenv("AZURE_OPENAI_API_KEY"),
     api_version = "2024-02-01"
   )

   response = client.chat.completions.create(
       model = "gpt-4o-ft", # model = "Custom deployment name you chose for your fine-tuning model"
       messages = [
           {"role": "system", "content": "You are a helpful assistant."},
           {"role": "user", "content": "Does Azure OpenAI support customer managed keys?"},
           {"role": "assistant", "content": "Yes, customer managed keys are supported by Azure OpenAI."},
           {"role": "user", "content": "Do other Azure AI services support this too?"}
       ]
   )

   print(response.choices[0].message.content)
    ```
 
## Limpar

Quando terminar o recurso do OpenAI do Azure, lembre-se de excluir a implantação ou todo o recurso no **portal do Azure** em `https://portal.azure.com`.

No portal do Azure Databricks, na página **Computação**, selecione seu cluster e selecione **&#9632; Terminar** para encerrar o processo.

Se você tiver terminado de explorar o Azure Databricks, poderá excluir os recursos que criou para evitar custos desnecessários do Azure e liberar capacidade em sua assinatura.
