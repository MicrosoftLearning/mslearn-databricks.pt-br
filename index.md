---
title: Instruções online hospedadas
permalink: index.html
layout: home
---

# Exercícios do Azure Databricks

Esses exercícios foram projetados para dar suporte ao seguinte conteúdo de treinamento no Microsoft Learn:

- [Implementar uma solução de análise de dados com o Azure Databricks](https://learn.microsoft.com/training/paths/data-engineer-azure-databricks/)
- [Implementar uma solução de Machine Learning com o Azure Databricks](https://learn.microsoft.com/training/paths/build-operate-machine-learning-solutions-azure-databricks/)

Você precisará de uma assinatura do Azure na qual tenha acesso administrativo para concluir esses exercícios.

{% assign exercises = site.pages | where_exp:"page", "page.url contains '/Instructions'" %} {% for activity in exercises  %}
- [{{ activity.lab.title }}]({{ site.github.url }}{{ activity.url }}) | {% endfor %}