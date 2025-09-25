# Documentação Técnica: Automação n8n para Upload de Arquivos no SharePoint

## 1. Introdução

Este documento detalha o processo de desenvolvimento de uma automação no n8n para gerenciar o upload de múltiplos arquivos para o Microsoft SharePoint. A automação foi projetada para lidar com dados dinâmicos provenientes de um webhook, garantindo que os arquivos sejam armazenados em pastas específicas e renomeados de forma única para evitar sobrescritas. A jornada de desenvolvimento foi marcada por desafios técnicos específicos da plataforma n8n e da integração com a API do SharePoint, exigindo uma abordagem iterativa e aprofundada na depuração. O objetivo final é fornecer um guia compreensivo sobre as soluções implementadas e as melhores práticas aprendidas, servindo como referência para futuros projetos de automação.

## 2. Declaração do Problema

A necessidade central era automatizar o recebimento de arquivos via webhook e seu subsequente upload para o SharePoint. Os requisitos incluíam:

*   **Recebimento Dinâmico:** O webhook deveria aceitar um número variável de arquivos PDF, juntamente com metadados JSON (como `documentType`, `employeeCAD`, `employeeName`, `sector`, `protocolId`).
*   **Organização no SharePoint:** Os arquivos deveriam ser armazenados em uma estrutura de pastas dinâmica no SharePoint, baseada nos metadados recebidos (ex: `Documentos Compartilhados/CONTRATOS/[SETOR]/[NOME_FUNCIONARIO] - MAT [CAD_FUNCIONARIO]`).
*   **Renomeação Inteligente:** Os arquivos deveriam ser renomeados no momento do upload para um formato padronizado (ex: `[CAD_FUNCIONARIO] - [TIPO_DOCUMENTO].[EXTENSAO]`).
*   **Prevenção de Sobrescrita:** Em caso de múltiplos uploads para a mesma pasta com o mesmo nome base, a automação deveria adicionar um sufixo numérico (ex: `123 - ADMISSIONAIS 1.pdf`, `123 - ADMISSIONAIS 2.pdf`) para garantir a unicidade e evitar a perda de dados.

Os desafios surgiram principalmente da interação entre a estrutura de dados do n8n (especialmente a separação entre dados JSON e binários), as particularidades da API do SharePoint e as diferenças de comportamento entre as versões do n8n, que exigiram uma depuração cuidadosa e a adaptação de soluções propostas inicialmente para o ambiente específico do usuário.



## 3. Metodologia de Desenvolvimento e Desafios Iniciais

A metodologia adotada foi iterativa e baseada em depuração contínua, dada a complexidade da interação entre os componentes do n8n e a API externa. Os desafios iniciais foram significativos e frequentemente relacionados à interpretação da estrutura de dados interna do n8n e às diferenças de comportamento entre as versões da plataforma.

### 3.1. Compreensão da Estrutura de Dados do Webhook

O primeiro passo foi analisar a estrutura do JSON recebido pelo webhook. Observou-se que o webhook recebia um único item que continha tanto os metadados JSON (`body.documentType`, `body.employeeCAD`, etc.) quanto os arquivos binários anexados, representados por chaves como `binary.files0`, `binary.files1`, etc. [1].

> **Dica de Mercado:** Em automações com dados complexos, a primeira etapa crucial é sempre a inspeção detalhada da carga útil (payload) de entrada. Ferramentas como o painel de execução do n8n ou um serviço como o `webhook.site` são indispensáveis para visualizar a estrutura exata dos dados.

### 3.2. Desafios com o Nó 'Code' e Versões do n8n

Inicialmente, tentou-se usar o nó `Code` para manipular os dados e separar os arquivos. No entanto, uma série de erros foram encontrados, revelando inconsistências na forma como o n8n lida com as variáveis `$items`, `$input`, e a propriedade `binary` em diferentes contextos e versões:

*   **`Cannot read properties of undefined (reading 'binary')`**: Este erro indicava que o código tentava acessar a propriedade `binary` de um objeto que não a possuía. Isso ocorria quando o item de entrada não continha dados binários ou quando a forma de acesso estava incorreta [2].
*   **`Cannot read properties of undefined (reading 'json')`**: Similar ao erro anterior, mas para a propriedade `json`. Apontava que o item de entrada não possuía a estrutura `json` esperada [3].
*   **`$items.find is not a function` e `$items is not iterable`**: Estes erros foram particularmente reveladores. Eles indicavam que a variável global `$items`, que em versões mais recentes do n8n é um array de itens, na versão do usuário (`1.79.3`) não se comportava como um array iterável. Isso invalidou abordagens que dependiam de `$items` ser um array padrão [4].

> **Dica de Mercado:** A compatibilidade de versões é um fator crítico em ambientes de desenvolvimento de automação. Sempre verifique a documentação específica da versão em uso e esteja ciente de que exemplos e tutoriais podem não ser diretamente aplicáveis se baseados em versões diferentes. A utilização de `console.log()` dentro do nó `Code` é uma técnica de depuração fundamental para inspecionar o conteúdo das variáveis em tempo de execução.

### 3.3. A Importância do Modo de Execução do Nó 'Code'

Após várias tentativas, descobriu-se que o modo de execução do nó `Code` era crucial. O erro `Code doesn't return a single object [item 0]` com a sugestão `If you need to output multiple items, please use the 'Run Once for All Items' mode instead` foi a pista definitiva [5].

*   **`Run Once for Each Item`**: Executa o código para cada item de entrada individualmente. Espera-se que o código retorne um único item ou um array de itens para cada execução.
*   **`Run Once for All Items`**: Executa o código uma única vez, recebendo todos os itens de entrada como um array. É ideal para processar múltiplos itens de uma vez e retornar um array de novos itens.

Para o cenário de separar múltiplos arquivos de um único item de entrada, o modo **`Run Once for All Items`** provou ser o correto, permitindo que o código acessasse todos os dados binários de uma vez e gerasse múltiplos itens de saída. No entanto, mesmo com o modo correto, a forma de acessar os dados binários ainda era um desafio, levando à necessidade de buscar os dados binários do nó `Merge` em etapas posteriores.



## 4. Arquitetura da Solução Final

Após uma extensa fase de depuração, a arquitetura final foi consolidada em um fluxo de trabalho robusto e eficiente, que aborda todos os requisitos iniciais. A solução combina o uso de nós padrão do n8n com chamadas diretas à API do SharePoint, garantindo flexibilidade e controle total sobre o processo.

O fluxo de trabalho final segue a seguinte sequência lógica:

`Webhook` -> `Edit Fields` -> `HTTP Request (Lista Arquivos)` -> `Merge` -> `Code (Lógica Principal)` -> `Loop Over Items` -> `HTTP Request (Upload)`

### 4.1. Análise Detalhada dos Nós

#### **1. Webhook**
*   **Propósito:** Ponto de entrada da automação. Recebe um único item contendo os metadados no corpo (`json.body`) e os múltiplos arquivos anexados na propriedade `binary`.

#### **2. Edit Fields**
*   **Propósito:** Realizar transformações iniciais nos dados. Neste caso, foi utilizado para padronizar o nome do funcionário para maiúsculas (`employeeName.toUpperCase()`), garantindo consistência na criação dos nomes das pastas.

#### **3. HTTP Request (Listar Arquivos)**
*   **Propósito:** Passo crucial para a lógica de prevenção de sobrescrita. Este nó faz uma chamada `GET` à API do SharePoint para listar todos os arquivos existentes na pasta de destino.
*   **Endpoint:** `https://[TENANT].sharepoint.com/sites/[SITE]/_api/web/GetFolderByServerRelativeUrl('/sites/[SITE]/[CAMINHO_PASTA]')/Files`
*   **Resultado:** Retorna um array de objetos, onde cada objeto contém os metadados de um arquivo existente na pasta, incluindo seu nome (`Name`).

#### **4. Merge**
*   **Propósito:** Recombinar os dados. Após a chamada HTTP, os dados originais (com os arquivos binários) são perdidos. O nó `Merge` é configurado no modo `Combine` com `Combine By: Position` para juntar o item original do `Edit Fields` (que ainda contém os binários) com o resultado da chamada HTTP. O resultado é um único item que contém tanto os arquivos a serem enviados quanto a lista de arquivos já existentes no destino.

#### **5. Code (Lógica Principal)**
*   **Propósito:** Este é o cérebro da automação. Configurado para rodar no modo `Run Once For All Items`, este nó executa duas tarefas críticas:
    1.  **Calcular o Índice Inicial (`startIndex`):** Ele analisa a lista de arquivos retornada pelo `HTTP Request` e determina qual o próximo sufixo numérico a ser usado. A lógica é robusta para encontrar o maior número existente nos nomes de arquivo que correspondem ao padrão base (ex: `123 - ADMISSIONAIS 10.pdf` -> encontra o número 10).
    2.  **Preparar Itens para o Loop:** Ele itera sobre os arquivos binários recebidos do webhook e, para cada um, cria um novo item de saída. Cada novo item contém os dados JSON originais, o arquivo binário correspondente, e uma nova propriedade, `fileIndex`, que representa o nome sequencial final do arquivo (ex: `startIndex + 0`, `startIndex + 1`, etc.).

#### **6. Loop Over Items**
*   **Propósito:** Itera sobre a lista de itens gerada pelo nó `Code`. Para cada arquivo a ser enviado, ele executa o passo seguinte uma vez.

#### **7. HTTP Request (Upload)**
*   **Propósito:** Realiza o upload do arquivo para o SharePoint. Este nó é executado para cada item do loop.
*   **Método:** `POST`
*   **Endpoint (URL):** A URL é construída dinamicamente e contém a lógica final de nomeação do arquivo. Utiliza a API `Files/add` do SharePoint.
    *   `.../Files/add(url='[NOME_ARQUIVO]',overwrite=true)`
*   **Lógica de Nomeação na URL:** A parte mais crítica é a construção do nome do arquivo, que utiliza o `fileIndex` calculado anteriormente:
    *   `{{ $json.body.employeeCAD + " - " + $json.body.documentType + ($json.fileIndex === 0 ? "" : " " + $json.fileIndex) + "." + $binary.file.fileExtension }}`
    *   Esta expressão garante que o primeiro arquivo (índice 0) não tenha sufixo numérico, e os subsequentes tenham (ex: `... 1.pdf`, `... 2.pdf`).
*   **Corpo (Body):** O corpo da requisição contém os dados binários do arquivo a ser enviado.

> **Melhor Prática de Arquitetura:** A separação de responsabilidades entre os nós é fundamental para a manutenibilidade. A decisão de usar um nó `HTTP Request` para listar os arquivos e um nó `Code` para processar essa lista, em vez de tentar fazer tudo em um único nó, torna o fluxo mais legível e fácil de depurar. A utilização do nó `Merge` para recombinar fluxos de dados é um padrão essencial em automações n8n complexas.



## 5. Lições Aprendidas e Melhores Práticas

A jornada de desenvolvimento desta automação forneceu insights valiosos e reforçou diversas melhores práticas aplicáveis a projetos de automação em geral, especialmente na plataforma n8n.

### 5.1. A Natureza da Estrutura de Dados do n8n

A lição mais fundamental foi a compreensão profunda de como o n8n gerencia os dados. A separação entre dados `json` e `binary` não é apenas uma convenção, mas uma característica arquitetural que dita como os nós interagem. Nós que não são projetados especificamente para manipular arquivos (como o `HTTP Request` ou o `IF`) frequentemente descartam a propriedade `binary`. A solução para isso é o uso estratégico do nó `Merge` para recombinar os dados de diferentes ramos do fluxo, garantindo que os dados binários estejam disponíveis nas etapas subsequentes. [6]

### 5.2. A Imprevisibilidade entre Versões

O desenvolvimento foi significativamente impactado pelas diferenças entre a versão do n8n utilizada pelo usuário e as versões mais recentes documentadas. Funções como o helper `$http` no nó `Code` ou a existência de nós como `Move Binary Data` não eram garantidas. Isso ressalta a importância de:

*   **Desenvolver para o Ambiente Alvo:** Sempre que possível, desenvolver e testar no ambiente de produção ou em um ambiente de homologação idêntico.
*   **Depuração Iterativa:** Não assumir que uma solução documentada funcionará de imediato. A depuração passo a passo, inspecionando a saída de cada nó, é a única maneira de garantir o progresso.

### 5.3. O Poder e os Perigos do Nó `Code`

O nó `Code` é a ferramenta mais poderosa do n8n, mas também a mais propensa a erros. A experiência demonstrou que:

*   **O Modo de Execução é Crucial:** A escolha entre `Run Once for Each Item` e `Run Once for All Items` altera fundamentalmente a forma como o código deve ser escrito e como ele acessa os dados de entrada (`$input` vs. `$items`).
*   **Acesso a Dados de Nós Anteriores:** A sintaxe `$node["NomeDoNo"].item` é a forma correta de acessar dados de um nó específico, sendo essencial quando um nó intermediário (como o `HTTP Request`) altera o fluxo de dados.
*   **Alternativas Visuais:** Antes de recorrer ao nó `Code`, deve-se sempre explorar se a funcionalidade desejada pode ser alcançada com nós padrão (`Edit Fields`, `Set`, etc.). Isso torna o fluxo mais legível para outros desenvolvedores e menos propenso a erros de sintaxe.

### 5.4. Interação com APIs Externas (SharePoint)

Ao integrar com APIs como a do SharePoint, a documentação oficial da API é mais importante do que a documentação do nó do n8n. A resolução do erro `400 Bad Request` não veio de uma configuração no n8n, mas da compreensão de como a API do SharePoint espera que os caminhos de pasta e nomes de arquivo sejam formatados na URL.

*   **`By ID` vs. `By Path`:** A descoberta de que o nó `SharePoint` do n8n estava usando `By ID` implicitamente, enquanto a necessidade era fornecer um caminho de pasta (`By Path`), foi um ponto de virada. Quando o nó padrão não oferece a flexibilidade necessária, usar o nó `HTTP Request` para fazer chamadas diretas à API é a solução mais robusta.
*   **Codificação de URL (`encodeURIComponent`):** Embora não tenha sido a solução final neste caso, a prática de codificar componentes dinâmicos de uma URL é uma defesa essencial contra erros de `Bad Request` causados por caracteres especiais em nomes de pastas ou arquivos.

> **Dica de Mercado:** Para integrações complexas, construa e teste as chamadas de API em uma ferramenta como o Postman ou o Insomnia primeiro. Uma vez que a chamada funcione lá, replicá-la no nó `HTTP Request` do n8n é uma tarefa muito mais simples e com menos variáveis.



## 6. Conclusão

A construção desta automação no n8n para upload de arquivos no SharePoint, com renomeação inteligente e prevenção de sobrescrita, foi um excelente estudo de caso sobre os desafios e as recompensas do desenvolvimento de automações complexas. A jornada, marcada por inúmeros erros e depurações, ressaltou a importância de uma compreensão profunda da plataforma, a necessidade de adaptabilidade às especificidades da versão em uso e a inestimável colaboração com o usuário para validar cada passo. A solução final é um testemunho da robustez do n8n quando suas capacidades são exploradas de forma estratégica, combinando a simplicidade dos nós visuais com o poder da programação JavaScript para resolver problemas complexos de integração. Este documento serve como um registro detalhado das lições aprendidas, fornecendo um roteiro para futuras implementações e reforçando a importância da paciência e da metodologia iterativa no desenvolvimento de automações.

## 7. Referências

[1] Dados de entrada do Webhook (JSON e Binário).
[2] Erro: `Cannot read properties of undefined (reading 'binary')`.
[3] Erro: `Cannot read properties of undefined (reading 'json')`.
[4] Erros: `$items.find is not a function` e `$items is not iterable`.
[5] Erro: `Code doesn't return a single object [item 0]` e sugestão de `Run Once for All Items`.
[6] Problemas de perda de dados binários após nós intermediários.


