# Requirements Document

## Introduction

Este documento define os requisitos para um dashboard de Open Banking desenvolvido em Go. A aplicação conecta-se a provedores de Open Banking para buscar transações bancárias do usuário, categoriza-as automaticamente usando regras e/ou machine learning, e exibe gráficos interativos filtráveis por categoria com granularidade mensal e semanal.

## Glossary

- **Dashboard**: A aplicação web principal que exibe gráficos e resumos financeiros ao usuário.
- **Open_Banking_Provider**: Serviço externo compatível com o padrão Open Banking (ex: Pluggy, Belvo, ou Open Finance Brasil) que fornece acesso às contas e transações bancárias do usuário mediante consentimento.
- **Transaction**: Registro de movimentação financeira (crédito ou débito) retornado pelo Open_Banking_Provider.
- **Category**: Classificação atribuída a uma Transaction (ex: Alimentação, Transporte, Saúde, Lazer, Salário, Outros).
- **Categorizer**: Componente interno responsável por atribuir uma Category a cada Transaction.
- **Chart**: Representação visual (gráfico de barras, pizza ou linha) dos dados financeiros agregados.
- **Filter**: Parâmetro de consulta que restringe os dados exibidos no Chart por Category, período de tempo ou granularidade.
- **Granularity**: Nível de agregação temporal dos dados: mensal (mês a mês) ou semanal (semana a semana).
- **Consent_Flow**: Fluxo de autorização OAuth2/Open Banking pelo qual o usuário concede permissão à aplicação para acessar seus dados bancários.
- **Access_Token**: Credencial obtida após o Consent_Flow que autoriza o acesso aos dados do usuário no Open_Banking_Provider.
- **User**: Pessoa que utiliza o Dashboard para visualizar e analisar suas transações bancárias.

---

## Requirements

### Requirement 1: Autorização e Consentimento do Usuário

**User Story:** Como usuário, quero autorizar a aplicação a acessar meus dados bancários via Open Banking, para que minhas transações sejam buscadas de forma segura com meu consentimento.

#### Acceptance Criteria

1. WHEN o usuário inicia o Consent_Flow, THE Dashboard SHALL redirecionar o usuário para a página de autorização do Open_Banking_Provider via OAuth2.
2. WHEN o Open_Banking_Provider retorna um código de autorização, THE Dashboard SHALL trocar o código por um Access_Token e armazená-lo de forma criptografada, nunca em texto simples.
3. IF o Consent_Flow falhar ou for cancelado pelo usuário, THEN THE Dashboard SHALL exibir uma mensagem de erro indicando o motivo da falha e disponibilizar a opção de iniciar um novo Consent_Flow.
4. WHEN o Access_Token estiver a 60 segundos ou menos da expiração, THE Dashboard SHALL renovar o token automaticamente usando o Refresh_Token sem exigir nova autorização do usuário.
5. IF a renovação automática do Access_Token falhar por motivo transitório (ex.: falha de rede), THEN THE Dashboard SHALL tentar novamente a renovação automática até 3 vezes antes de solicitar ao usuário que inicie um novo Consent_Flow.
6. THE Dashboard SHALL armazenar o Access_Token de forma criptografada, nunca em texto simples.
7. IF o Refresh_Token estiver expirado ou for rejeitado pelo Open_Banking_Provider, THEN THE Dashboard SHALL invalidar a sessão atual e solicitar ao usuário que inicie um novo Consent_Flow.

---

### Requirement 2: Busca de Transações Bancárias

**User Story:** Como usuário, quero que a aplicação busque minhas transações bancárias do Open Banking, para que eu possa visualizar meu histórico financeiro completo.

#### Acceptance Criteria

1. WHEN o usuário solicita a atualização dos dados, THE Dashboard SHALL buscar as Transactions do Open_Banking_Provider utilizando o Access_Token válido.
2. THE Dashboard SHALL buscar Transactions de pelo menos os últimos 12 meses a partir da data de requisição.
3. WHEN o Open_Banking_Provider retorna Transactions, THE Dashboard SHALL persistir todos os campos retornados pelo provedor localmente para cada Transaction.
4. IF o Open_Banking_Provider retornar erro na requisição, THEN THE Dashboard SHALL registrar o erro em log e exibir uma notificação ao usuário, mantendo o Dashboard operacional com os dados previamente persistidos.
5. WHEN uma Transaction já existente for retornada novamente, THE Dashboard SHALL atualizar os dados locais sem criar duplicatas, usando o identificador único da Transaction fornecido pelo Open_Banking_Provider.
6. THE Dashboard SHALL suportar paginação na busca de Transactions, processando todos os resultados disponíveis no Open_Banking_Provider até o limite de 10.000 Transactions por sincronização.
7. IF o Access_Token utilizado na busca for rejeitado pelo Open_Banking_Provider, THEN THE Dashboard SHALL tentar renovar o token uma vez antes de exibir uma mensagem de erro ao usuário.
8. IF o Open_Banking_Provider possuir mais Transactions disponíveis além das 10.000 sincronizadas, THEN THE Dashboard SHALL exibir uma mensagem indicando que a sincronização está incompleta e perguntar ao usuário se ele gostaria de sincronizar mais Transactions.

---

### Requirement 3: Categorização Automática de Transações

**User Story:** Como usuário, quero que minhas transações sejam categorizadas automaticamente, para que eu não precise classificá-las manualmente.

#### Acceptance Criteria

1. WHEN uma nova Transaction é persistida, THE Categorizer SHALL atribuir uma Category à Transaction com base na descrição da Transaction.
2. THE Categorizer SHALL classificar Transactions nas seguintes categorias predefinidas: Mercado, Restaurante, Transporte, Saúde, Lazer, Moradia, Educação, Salário, Transferência e Outros.
3. IF a descrição de uma Transaction não corresponder a nenhuma das categorias predefinidas, THEN THE Categorizer SHALL atribuir a Category "Outros" à Transaction.
4. THE Categorizer SHALL utilizar correspondência de palavras-chave configurável para associar descrições de Transaction a categorias, aplicando a primeira regra correspondente encontrada na ordem de configuração em caso de múltiplas correspondências.
5. WHEN o usuário corrigir manualmente a Category de uma Transaction, THE Dashboard SHALL persistir a Category corrigida substituindo a Category anterior.
6. WHILE a Category corrigida manualmente estiver persistida, THE Dashboard SHALL utilizar exclusivamente essa Category em todos os cálculos e exibições relacionados à Transaction, sem exceções, incluindo caches ou views intermediárias.
7. WHEN a aplicação é inicializada, THE Categorizer SHALL processar em background todas as Transactions que não possuem Category atribuída e atribuir uma Category a cada uma delas. Antes da atribuição essas Transactions devem mostrar na categoria "Classificando..." e assim que concluida sua atribuição mudar para exibir a categoria atribuida. A tela não deve piscar ou mover o scroll durante essa atualização.
8. IF o usuário corrigir manualmente a Category de uma Transaction cujo resultado de categorização em background ainda não foi persistido, THEN THE Dashboard SHALL descartar o resultado do background para essa Transaction e utilizar exclusivamente a Category corrigida manualmente.

---

### Requirement 4: Exibição de Gráficos por Categoria

**User Story:** Como usuário, quero visualizar gráficos dos meus gastos por categoria, para que eu possa entender para onde meu dinheiro está indo.

#### Acceptance Criteria

1. WHILE um período estiver selecionado, THE Dashboard SHALL exibir um gráfico de pizza mostrando a distribuição percentual dos gastos por Category, onde cada fatia representa uma Category com seu respectivo nome e percentual visíveis.
2. WHILE um período estiver selecionado, THE Dashboard SHALL exibir um gráfico de barras verticais mostrando o valor total gasto por Category para o período, ordenado do maior para o menor valor.
3. A esquerda de cada barra deve-se exibir uma barra pontilhada com o valor do período equivalente anterior para ajudar a entender se está aumentando ou diminuindo.
4. IF nenhum período estiver selecionado, THEN THE Dashboard SHALL aplicar o mês corrente como período padrão para exibição dos Charts.
5. WHEN o usuário selecionar uma Category específica no Filter, THE Dashboard SHALL atualizar o gráfico de pizza e o gráfico de barras para exibir apenas os dados da Category selecionada, mantendo o Filter resetável para exibir todas as Categories.
6. WHEN o usuário resetar o Filter, THE Dashboard SHALL atualizar imediatamente os Charts para exibir todas as Categories.
7. IF nenhuma Transaction existir para o período e Category selecionados, THEN THE Dashboard SHALL exibir uma mensagem indicando ausência de dados para o período e Category selecionados, no lugar de cada Chart afetado.
8. THE Dashboard SHALL exibir o valor monetário em formato brasileiro (R$ 1.234,56) em qualquer valor monetário exibido na tela, incluindo todos os Charts e resumos textuais, independentemente de Charts estarem visíveis.

---

### Requirement 5: Filtro por Granularidade Temporal

**User Story:** Como usuário, quero filtrar os gráficos por mês ou por semana, para que eu possa analisar meus gastos em diferentes granularidades de tempo.

#### Acceptance Criteria

1. THE Dashboard SHALL exibir o controle de seleção de Granularity com a opção "mensal" selecionada por padrão ao carregar a página.
2. WHEN o usuário selecionar a Granularity "mensal", THE Dashboard SHALL agregar as Transactions por mês calendário e exibir um Chart de linha com os totais mensais dos últimos 12 meses calendário anteriores ao mês corrente, inclusive.
3. WHEN o usuário selecionar a Granularity "semanal", THE Dashboard SHALL agregar as Transactions por semana ISO e exibir um Chart de linha com os totais semanais das últimas 52 semanas ISO anteriores à semana corrente, inclusive.
4. WHEN o usuário alterar a Granularity, THE Dashboard SHALL atualizar todos os Charts simultaneamente em no máximo 2 segundos sem recarregar a página.
5. WHEN o usuário selecionar um intervalo de datas personalizado, THE Dashboard SHALL restringir os dados exibidos em todos os Charts às Transactions cujas datas estejam dentro do intervalo selecionado, substituindo totalmente as janelas fixas de 12 meses e 52 semanas, sem restrição de período mínimo ou máximo além do que o usuário selecionar, mantendo a Granularity ativa para agregar os dados dentro desse intervalo.
6. IF o intervalo de datas selecionado não contiver nenhuma Transaction, THEN THE Dashboard SHALL exibir uma mensagem indicando que não há dados disponíveis para o período selecionado e ocultar os Charts.

---

### Requirement 6: Navegação e Interface do Dashboard

**User Story:** Como usuário, quero navegar por uma interface clara e responsiva, para que eu possa acessar as informações financeiras de forma intuitiva em qualquer dispositivo.

#### Acceptance Criteria

1. THE Dashboard SHALL ser acessível via navegador web sem necessidade de instalação de software adicional pelo usuário.
2. THE Dashboard SHALL renderizar sem sobreposição de elementos, sem rolagem horizontal e com texto legível.
3. The Dashboard SHALL ser responsivo de acordo com a tela que está exibindo, ajustando o tamanho dos gráficos e os posicionando em sequencia um abaixo do outro se a tela não for grande o suficiente para exibir lado a lado.
4. WHEN o usuário acessar o Dashboard sem um Access_Token válido, THE Dashboard SHALL redirecionar o usuário para o Consent_Flow automaticamente.
5. WHILE os dados estiverem sendo buscados do Open_Banking_Provider, THE Dashboard SHALL exibir um indicador de carregamento visível na tela que não impeça a navegação.
6. WHEN Dados forem baixados do Open_Banking_Provider, eles devem ser armazenados em um banco de dados server-side em um formato que facilite futuras consultas.
7. THE Dashbaord SHALL exibir na tela os dados a medida que eles forem baixados, sem aguardar o carregamento de todas as transações.
8. THE Dashboard SHALL exibir na tela inicial um resumo com: saldo total, total de créditos e total de débitos, usando o período dos últimos 30 dias como padrão quando nenhum período for selecionado pelo usuário.
9. IF o período selecionado não contiver Transactions, THEN THE Dashboard SHALL exibir o resumo com os valores zerados e uma mensagem indicando ausência de dados para o período.

---

### Requirement 7: Persistência e Cache de Dados

**User Story:** Como usuário, quero que os dados já carregados fiquem disponíveis rapidamente, para que o Dashboard não fique lento a cada acesso.

#### Acceptance Criteria

1. THE Dashboard SHALL persistir Transactions localmente em um banco de dados relacional para reduzir chamadas ao Open_Banking_Provider e melhorar a performance de carregamento.
2. WHEN os dados locais tiverem sido atualizados há menos de 1 hora, THE Dashboard SHALL servir os dados do cache local sem realizar nova chamada ao Open_Banking_Provider.
3. WHEN os dados locais tiverem sido atualizados há 1 hora ou mais, THE Dashboard SHALL buscar Transactions atualizadas do Open_Banking_Provider em segundo plano e exibir uma notificação visível ao usuário quando a atualização for concluída.
4. THE Dashboard SHALL completar a renderização inicial dos Charts em menos de 3 segundos a partir do momento em que a navegação para a tela principal for iniciada, quando os dados estiverem no cache local.
5. IF o banco de dados local estiver indisponível, THEN THE Dashboard SHALL exibir uma mensagem de erro indicando indisponibilidade do armazenamento local, tentar buscar os dados diretamente do Open_Banking_Provider e registrar o problema em log.
