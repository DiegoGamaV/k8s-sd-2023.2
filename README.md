# Roteiro de Kubernetes - Sistemas Distribuídos, 2023.2

Esse é um repositório para roteiro prático de Kubernetes, para disciplina de Sistemas Distribuídos, do curso de Bacharelado em Ciência da Computação, e período letivo de 2023.2.

O roteiro está dividido em X subroteiros pequenos, que exploram diferentes aspectos de Kubernetes para familiarização. É recomendado segui-los em ordem:

1. A [primeira parte](./roteiros/parte-1-comandos-basicos.md), que explica como visualizar os recursos sendo atualmente gerenciados pelo Kubernetes, bem como implantar uma aplicação simples.
2. A [segunda parte](./roteiros/parte-2-aplicacao-web.md), que mostra como implantar uma aplicação web que precisa ser exposta para internet, e que necessita de um banco de dados com volume persistente.

As atividades consideram que você esteja utilizando o **Minikube**, um ambiente simplificado para Kubernetes local, para o propósito de testes e desenvolvimento em um único nó. Para instalar o Minikube em sua versão mais recente, siga o [passo a passo oficial](https://minikube.sigs.k8s.io/docs/start/). Garanta que você também possui uma versão recente de **Docker Engine** instalada, o que pode ser feito seguindo o [passo a passo adequado para seu SO e arquitetura](https://docs.docker.com/engine/install/).

O diretório [manifestos](./manifestos/) contém todos os arquivos YAML necessários para a execução do roteiro.
