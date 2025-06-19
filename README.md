# Powertip – Guidelines de Desenvolvimento Power BI Desktop

> **Objetivo**
>
> Este documento reúne boas práticas para desenvolvimento no **Power BI Desktop**, abrangendo etapas de ETL, modelagem, criação de medidas DAX, design de relatórios, controle de versão e documentação. O objetivo é fornecer um guia estruturado para analistas (nível Jr/Pleno) seguirem padrões que melhorem a performance, manutenibilidade e consistência dos projetos de Power BI. A aplicação dessas guidelines ajuda a construir relatórios confiáveis, mais fáceis de evoluir e com visual profissional, alinhados às recomendações oficiais da Microsoft e da comunidade. A seguir, as orientações estão organizadas por etapa do desenvolvimento.

---

## ETL (Power Query)

No processo de ETL (Extract, Transform, Load) dentro do Power BI (usando o Power Query), é importante manter consultas bem estruturadas, claras e eficientes. As práticas a seguir visam facilitar a compreensão das transformações de dados e otimizar o desempenho da carga.

### Consultas de Referência

- **Reutilize lógica com consultas referenciadas**: Sempre que precisar usar a mesma lógica de transformação em várias consultas, evite duplicar passos manualmente. Em vez disso, crie uma consulta base (com Carga Desabilitada) e faça com que outras consultas referenciem essa consulta base. Assim, você mantém a lógica central em um só lugar, facilitando atualizações futuras.

- **Cuidado com desempenho**: Embora úteis, consultas referenciadas podem impactar o tempo de atualização. Cada consulta que referencia outra pode fazer a fonte de dados ser acessada várias vezes (não há “cache” automático do resultado da primeira)

Isso significa que no refresh, a consulta base pode ser executada múltiplas vezes – por exemplo, uma vez para cada consulta referenciada – tornando a atualização mais lenta e sobrecarregando a fonte de dados,

A imagem abaixo ilustra esse cenário, em que a Query1 (não carregada) é referenciada por três outras consultas, causando reexecuções separadas de Query1:

[ADICIONAR IMAGEM]

> Dica de performance: Se notar lentidão devido a referências múltiplas, considere bufferizar a consulta base ou reavaliar sua estratégia de ETL. Porém, note que funções como Table.Buffer não impedem reexecuções em consultas separadas
. Uma alternativa recomendada pela Microsoft é usar Dataflows (Fluxos de Dados) ou etapas de preparação upstream. Mover a lógica compartilhada para um dataflow permite armazenar os dados transformados de forma persistente no serviço do Power BI, acelerando o refresh e evitando hits repetidos à fonte.
. Em resumo: use consultas referenciadas para evitar duplicação de passos, mas esteja atento ao impacto e opte por soluções escaláveis (como dataflows) em casos de grande volume ou múltiplas dependências
> [Consultas referenciadas](https://learn.microsoft.com/pt-br/power-bi/guidance/power-query-referenced-queries)

### Descrição de Consultas

- **Documente as consultas no próprio Power BI**: Aproveite o campo de Descrição das consultas (em Propriedades da consulta, no editor do Power Query) para anotar o que cada consulta faz, sua origem ou qualquer detalhe relevante. Essa descrição não aparece para os usuários finais, mas serve como documentação interna do projeto – útil para você no futuro ou outros desenvolvedores que forem dar manutenção. Por exemplo, você pode descrever uma consulta assim: “Importa vendas brutas do ERP X, filtra apenas últimos 2 anos, agrega por mês e região.” Tais descrições tornam mais fácil entender o ETL sem precisar inspecionar cada etapa.

- **Nomem consultas claramente**: No painel Campos do Power BI (e nas exibições de Relatório e Modelo), os nomes das tabelas/consultas são o que aparece. Portanto, renomeie as consultas para nomes de negócio intuitivos. Em vez de ficar com “Consulta1”, “Consulta2” ou nomes técnicos da fonte (vw_sls_qtr_amt), use algo como VendasTrimestrais ou Faturamento_Últimos2Anos. Isso facilita identificar os dados no modelo e nos visualizadores (slicers, legendas, etc). Considere um padrão de nomenclatura consistente (ex: iniciar nomes de tabelas de fatos com f_ e de dimensões com d_, conforme detalhado adiante na seção de Modelagem).

### Nomenclatura de Etapas

- **Renomeie as etapas aplicadas:** O Power Query gera nomes automáticos para cada passo (por ex.: Tipo Alterado, Linhas Filtradas, Índice Adicionado), muitas vezes incluindo espaços e caracteres especiais, o que resulta em código M com #"Nome da Etapa" entre aspas. É recomendável renomear cada etapa para um nome sem espaços e que indique claramente sua ação. Por exemplo, renomeie Tipo Alterado para TipoAlterado_Produtos ou Linhas Filtradas para FiltrarVendasÚltimos2Anos. Assim, além de evitar espaços (eliminando a necessidade de #"" no código), você documenta a função da etapa. Essa prática deixa o M mais legível e fácil de manter – “código mais limpo, fácil de ler e mais conveniente de escrever”

- **Dica – aproveite a IA:** Ferramentas de Inteligência Artificial podem ajudar na documentação das etapas. Por exemplo, você pode copiar o código M da sua consulta e pedir para um assistente de IA (como o ChatGPT) sugerir nomes melhores ou explicar cada etapa. Isso pode dar insights de nomenclaturas mais claras. Além disso, a Microsoft vem incorporando recursos de IA no Power Query (como o Column from Examples e, no futuro, possivelmente o Copilot), que podem facilitar a criação de transformações com nomes amigáveis. Use essas ajudas para aprimorar a qualidade das etapas, mas sempre revise para manter consistência com seu padrão.

> Resumo: A nomeação adequada de consultas e etapas é um investimento em manutenibilidade. Pequenos cuidados, como evitar espaços e adotar descrições significativas, tornam o trabalho futuro muito mais fácil – afinal, “o código se escreve uma vez, mas lê-se muitas”.

### Carregue Somente Colunas Necessárias

Remova colunas que não serão usadas em visuais, relacionamentos ou cálculos. Menos colunas = modelo menor, refresh mais rápido e lista de campos limpa.

```powerquery
Table.SelectColumns(Source, {"Pedido", "Cliente", "Valor"})
```

> **Referências:** [Optimize column load](https://learn.microsoft.com/power-query/performance)

### Renomeie Colunas com Nomes de Negócio

Converta siglas técnicas (`cust_id`) em termos claros (`ClienteID`) e mantenha idioma consistente. Isso melhora legibilidade, uso de Q\&A e entendimento pelos usuários.

> **Referências:** [Rename columns](https://learn.microsoft.com/power-query)

### Nomenclatura de Tabelas

- **Use prefixos para tipo de tabela:** Uma prática comum é prefixar o nome das tabelas de acordo com seu papel no modelo. Por exemplo, usar f_ (de fact) no início de tabelas de fato (transacionais, geralmente valores numéricos acumuláveis) e d_ (de dimension) em tabelas dimensionais (categorização, informações descritivas). Ex.: f_Vendas, d_Produto, d_Calendario. Isso ajuda a identificar rapidamente na lista de campos quais são fatos e quais são dimensões. Além disso, agrupa visualmente tabelas semelhantes (já que o Power BI lista em ordem alfabética, todas que começam com f_ ficarão juntas). Se preferir, pode escrever por extenso (FatoVendas, DimProduto), mas os prefixos curtos são suficientes e bastante utilizados em projetos Power BI.

- **Nomes descritivos e sem espaços:** Assim como em colunas, evite espaços ou caracteres especiais nos nomes de tabela. Use _ (underline) se precisar separar palavras, ou CamelCase/PascalCase. Por exemplo, em vez de “Tabela Vendas 2023”, use f_Vendas2023 ou f_Vendas_2023. Isso evita ter que referenciar como 'Tabela Vendas 2023' (com aspas) nas fórmulas DAX e facilita scripts automatizados. Mantenha nomes curtos porém significativos – o suficiente para distinguir a fonte ou função. Por exemplo, se houver duas tabelas de vendas de sistemas distintos, poderíamos usar f_VendasERP e f_VendasCRM.

- **Exemplo de padronização:** Considere um modelo simples de vendas: poderíamos ter as tabelas d_Calendario, d_Cliente, d_Produto e f_Vendas. Sem espaços e com prefixos, fica claro. Outro exemplo, para dados financeiros: d_ContaContábil, d_CentroCusto e f_Lançamentos. Escolha singular ou plural conforme preferência, mas mantenha consistente (muitos preferem dimensões no singular – ex.: d_Cliente – já que se referem a entidade, e fatos no plural – f_Vendas – pois agregam múltiplos eventos; porém não é regra fixa). O crucial é que o nome da tabela transmita seu conteúdo ou papel no modelo sem ambiguidade.

> Com essas práticas de ETL, suas consultas serão mais organizadas, documentadas e otimizadas. Isso estabelece uma base sólida para a próxima etapa: a Modelagem de Dados.

---

## Modelagem de Dados

A modelagem é o coração do projeto Power BI. É onde estruturamos as tabelas importadas, definimos relações entre elas e preparamos o terreno para cálculos DAX eficientes. Seguir um esquema estrela, configurar corretamente os relacionamentos e evitar armadilhas como excesso de colunas calculadas no modelo garante que o desempenho e a usabilidade do relatório sejam os melhores possíveis. Vamos às guidelines de modelagem:

### Esquema Estrela

Exemplo de modelo de dados com esquema em estrela (tabela fato “Sales” conectada a quatro dimensões: Date, Product, State, Region). As setas indicam relações de 1 (um) para * (muitos), das dimensões filtrando a fato

> **Referências:** [Designing star schemas](https://learn.microsoft.com/power-bi/guidance/star-schema)

### Relacionamentos

- **Direção do filtro (Cross-filter direction):** Por padrão, use relacionamentos unidirecionais (Single) filtrando do lado “um” para o lado “muitos”. Isso corresponde ao esquema estrela clássico e evita cálculos ambíguos. Em casos especiais (como dois fatos compartilhando dimensões, ou relações muitos-para-muitos controladas), pode-se avaliar usar bidirecional, mas com cautela – bidirecionais podem introduzir dependências complexas e risco de filtros circulares. Em resumo: mantenha Single direction sempre que possível, especialmente entre dimensões e fatos principais, para um modelo robusto e fácil de entender.

- **Evite relacionamentos muitos-para-muitos diretos:** Se encontrar necessidade de relacionar duas tabelas em uma cardinalidade : (muitos-para-muitos) – algo que o Power BI até suporta nativamente – reavalie o design. Na maioria dos casos, um relacionamento : indica ausência de uma dimensão intermediária que deveria quebrar essa relação. Tente inserir uma tabela de dimensão intermediária (p. ex., uma lista única de valores que existem nas duas tabelas) para normalizar o modelo e voltar a relações 1:* de cada lado. Relacionamentos muitos-para-muitos podem causar duplicações inesperadas ou dificuldade para o engine filtrar corretamente, além de prejudicar a performance. Use-os apenas se for claramente necessário e compreendido (por exemplo, uma tabela de fatos granular querendo relacionar com outra tabela de fatos também granular por uma coluna comum – o ideal seria criar uma dimensão comum).

- **Relações ativas vs. inativas:** O Power BI permite marcar apenas um relacionamento ativo entre duas tabelas por vez (caso haja mais de um caminho possível). Se você tem, por exemplo, uma tabela de Vendas ligada a d_Calendario tanto pela data de venda quanto pela data de entrega, terá que escolher um relacionamento principal (ativo) – digamos DataVenda ativo – e o outro ficará inativo. Você pode então usar funções DAX como USERELATIONSHIP para ativar o relacionamento alternativo em medidas específicas (ex.: calcular entregas usando DataEntrega). A guideline aqui é: mantenha o modelo simples com um relacionamento ativo por par de tabelas; relacionamentos inativos são úteis, mas não exagere (muitos relacionamentos inativos podem indicar necessidade de separar tabelas ou criar cálculos diferentes). Uma prática comum é duplicar a dimensão de data (duas tabelas de calendário distintas) para evitar relacionamentos inativos – por ex., d_CalendarioVenda e d_CalendarioEntrega, cada uma ativa com Vendas. Isso elimina a necessidade de USERELATIONSHIP e simplifica o uso de funções de inteligência de tempo, ao custo de redundância de dimensões. Avalie conforme o caso.

### Evitar Colunas Calculadas em DAX

- **Prefira colunas calculadas no Power Query (M) ou fonte de dados:** Uma coluna calculada DAX é criada dentro do modelo Power BI, após a carga dos dados. Embora às vezes conveniente, essa prática geralmente não é a ideal em termos de desempenho e otimização do modelo. Colunas calculadas via DAX são calculadas linha a linha após todo o conjunto de dados ser importado, e tendem a não se compressar tão eficientemente quanto colunas criadas na fonte ou no Power Query
Além disso, elas aumentam o tamanho do modelo e consomem mais memória, pois o VertiPaq (mecanismo do Power BI) não reotimiza completamente a compressão quando você adiciona uma coluna via DAX posteriormente
Em termos simples, adicionar colunas pelo Power Query (nas consultas, antes do load) ou já trazer pronto do banco de dados resulta em um modelo mais enxuto e em tempos de atualização menores do que criar essas colunas depois, via DAX no Desktop

- **Impacto no processamento e memória:** Quando você adiciona N colunas calculadas no modelo, o processo de refresh precisa recalcular todas elas sempre que os dados são atualizados. Isso pode prolongar significativamente o tempo de atualização, pois essas colunas DAX são avaliadas somente após todas as consultas Power Query terminarem de carregar. Além disso, cada coluna calculada ocupa espaço no modelo e, por não estar otimizada na compressão global, frequentemente ocupa mais espaço do que ocuparia se viesse da fonte. Estudos práticos mostram cenários de colunas DAX chegando a consumir até 10x mais memória do que se fossem materializadas no Power Query. Ou seja, colunas calculadas podem “inchar” o modelo e prejudicar performance de consultas, já que mais dados precisam ser varridos. Por isso, a diretriz geral é: evite colunas calculadas DAX sempre que houver alternativa (M ou SQL).

- **Exceções – quando usar colunas calculadas:** Há casos em que uma coluna calculada é justificável ou mesmo necessária. Por exemplo, se o cálculo envolve lógica complexa de medidas ou depende do contexto do modelo que não existe na etapa de ETL (Power Query), então só é possível via DAX. Um caso clássico é quando se precisa de hierarquias pai-filho (organogramas, etc.) usando funções DAX como PATH – aí faz sentido ter colunas DAX, já que dependem de iterações no próprio modelo. Outro caso: criar uma coluna de classificação dinâmica baseada em medidas (rankings que mudam conforme filtro) – isso não dá para fazer no Power Query. Nestas situações, avalie o trade-off e documente a necessidade. Mas ainda assim, questione: essa lógica poderia ser empurrada para a fonte de dados ou resolvida com uma dimensão auxiliar? Muitas vezes a resposta é sim. Portanto, trate colunas calculadas DAX como último recurso. A preferência deve ser: 1) cálculo na consulta SQL ou vista, 2) cálculo no Power Query (coluna personalizada), 3) cálculo DAX (somente se for inevitável). Essa recomendação está alinhada às melhores práticas da Microsoft, que afirmam ser menos eficiente adicionar colunas no modelo via DAX do que via M

### Tabela Calendário via DAX

Crie uma tabela de datas com `CALENDAR` + `ADDCOLUMNS`, marque‑a como *Date table* e relacione‑a às fatos.
Evite usar CALENDARAUTO sempre que possível.

```dax
d_Calendario =
VAR _datamin = MIN( fVendas[data_venda] )
VAR _datamax = MAX( fVendas[data_venda] )
VAR _start = DATE( YEAR( _datamin ), 1, 1 )
VAR _end = DATE( YEAR( _datamax ), 12, 31 )
VAR _calendar = CALENDAR( _start, _end )
VAR _add = ADDCOLUMNS( _calendar , 
    "Ano", DATE(YEAR([Date]), 1 , 1 ),
    "Ano Número", YEAR([Date]) , 
    "Mês Texto", FORMAT([Date],"mmm-yy"), 
    "Class-Mes", YEAR([Date])*100 + MONTH([Date]),
    "Mês", EOMONTH([Date], -1) + 1,
    "Semestre", IF(MONTH([Date]) <= 6, DATE(YEAR([Date]), 1, 1), DATE(YEAR([Date]), 7, 1)),
    "Trimestre", DATE(YEAR([Date]), ((QUARTER([Date]) - 1) * 3) + 1, 1),
    "Dia da Semana" , WEEKDAY([Date]),
    "Dia da Semana Texto" , 
        SWITCH( 
            WEEKDAY([Date]),
            1, "Domingo",
            2, "Segunda",
            3, "Terça",
            4, "Quarta",
            5, "Quinta",
            6, "Sexta",
            7, "Sábado",
            BLANK()
        ),
    "Dia Número", DAY([Date])  
    )
RETURN 
_add

```

#### Tabela Calendário Especial

Relacionamento 1 pra muitos bidirecional com a d_Calendario
Permite fitrar períodos específicos.

```dax
d_Calendario_Especial = 
VAR _incio = MIN(d_Calendario[Date])
VAR _fim = MAX(d_Calendario[Date] )
VAR _datetable = CALENDAR ( _incio, _fim )
VAR _today =  TODAY()
VAR _month = MONTH(_today)
VAR _day = day(_today)
VAR _year = YEAR(_today)
VAR _lastyear1 = YEAR(_today)-1
VAR _lastyear2 = YEAR(_today)-2
VAR _thismonthstart = DATE(_year,_month,1)
VAR _thismonthend = EOMONTH(DATE(_year, _month, 1),0)
VAR _thisyearstart = DATE(_year,1,1)
VAR _thisyearend = DATE(_year,12,31)
VAR _lastyearstart1= DATE(_lastyear1,1,1)
VAR _lastyearend1= DATE(_lastyear1,12,31)
VAR _lastyearstart2= DATE(_lastyear2,1,1)
VAR _lastyearend2= DATE(_lastyear2,12,31)
VAR _lastmonthstart = EDATE(_thismonthstart,-1)
VAR _lastmonthend = _thismonthstart-1 
VAR _thisquarterstart = DATE(YEAR(_today),SWITCH(true,_month>9,10,_month>6,7,_month>3,4,1),1)
VAR _last12month = EDATE(_thismonthstart,-12)
 
RETURN
UNION(
ADDCOLUMNS(FILTER(_datetable,[Date] >= _today ),"Period","Hoje", "Sigla", "HJ" ,"Order",1),
ADDCOLUMNS(FILTER(_datetable,[Date] >=  _today -1  ),"Period","Ontem", "Sigla", "LD1" ,"Order",2),
ADDCOLUMNS(FILTER(_datetable,[Date] >=  _today -7 &&  [Date] <=  _today ),"Period","Últimos 7 Dias", "Sigla", "LD7" ,"Order",3),
ADDCOLUMNS(FILTER(_datetable,[Date] >=  _today -15 &&  [Date] <=  _today ),"Period","Últimos 15 Dias", "Sigla", "LD15" ,"Order",4),
ADDCOLUMNS(FILTER(_datetable,[Date] >=  _today -30 &&  [Date] <=  _today ),"Period","Últimos 30 Dias", "Sigla", "LD30" ,"Order",5),
ADDCOLUMNS(FILTER(_datetable,[Date] >=  _today -60 &&  [Date] <=  _today ),"Period","Últimos 60 Dias", "Sigla", "LD60" ,"Order",6),
ADDCOLUMNS(FILTER(_datetable,[Date] >=  _today -90 &&  [Date] <=  _today ),"Period","Últimos 90 Dias", "Sigla", "LD90" ,"Order",7),
ADDCOLUMNS(FILTER(_datetable,[Date] >=  _today -180 &&  [Date] <=  _today ),"Period","Últimos 180 Dias", "Sigla", "LD180" ,"Order",8),
ADDCOLUMNS(FILTER(_datetable,[Date] >=  _today -360 &&  [Date] <=  _today ),"Period","Últimos 360 Dias", "Sigla", "LD360" ,"Order",9),
ADDCOLUMNS(FILTER(_datetable,[Date]>= _lastmonthstart && [Date] <= _lastmonthend ),"Period","Mês Anterior", "Sigla", "LM" ,"Order",1),
ADDCOLUMNS(FILTER(_datetable,[Date]>= _lastmonthstart && [Date] <= _lastmonthend ),"Period","Mês Anterior", "Sigla", "LM" ,"Order",1),
ADDCOLUMNS(FILTER(_datetable,[Date]>= _thismonthstart && [Date] <= _thismonthend ),"Period","Mês Atual", "Sigla", "CM", "Order",2),
ADDCOLUMNS(FILTER(_datetable,[Date]>= _thisyearstart && [Date] <= _thismonthend ) ,"Period","Ano até Agora", "Sigla", "YTD", "Order",3),
ADDCOLUMNS(FILTER(_datetable,[Date]>= _thisyearstart && [Date]<= _thisyearend ),"Period","Ano até Final", "Sigla","FY", "Order",4),
ADDCOLUMNS(FILTER(_datetable,[Date]>= _last12month && [Date] <= _thismonthend ) ,"Period","Últimos 12 meses", "Sigla","LM12", "Order",5),
ADDCOLUMNS(FILTER(_datetable,[Date]>= _lastyearstart1 && [Date] <= _lastyearend1),"Period","Ano Anterior 1", "Sigla","PY1", "Order",6),
ADDCOLUMNS(FILTER(_datetable,[Date]>= _lastyearstart2 && [Date] <= _lastyearend2),"Period","Ano Anterior 2", "Sigla", "PY2", "Order",7),
ADDCOLUMNS(_datetable, "Period", "Todos", "Sigla","TDS" , "Order", 10)
)
```

---

## Cálculos, Análises e Fórmulas DAX

Nesta seção abordamos boas práticas ao criar medidas, métricas e outras fórmulas DAX no Power BI. Escrever DAX de forma consistente e eficiente é crucial para obter resultados corretos e facilitar a manutenção por você ou outros desenvolvedores. Também trataremos de recursos avançados como grupos de cálculo e parâmetros de campo, que ajudam a organizar e simplificar análises complexas.

### Convenção de Nomes

- **Nomeie medidas de forma descritiva:** Cada medida DAX que você criar deve ter um nome claro que indique o quê ela representa e, se pertinente, como ela está sendo calculada. Por exemplo, evite nomes genéricos como “Total” ou “Valor2”. Prefira Total Vendas, Qtd Clientes Ativos, % Crescimento YOY. Inclua unidades ou contexto se necessário, e seja consistente: se usar “Total [Algo]” para somas, mantenha esse padrão (Total Vendas, Total Custos, Total Lucro). Se for uma média, especifique (ex: Média Idade Cliente). Para medidas de % ou índices, deixar claro (ex: Margem % em vez de só “Margem”). Lembre-se que esses nomes aparecem nos gráficos e tooltips, então pense também na perspectiva do usuário final – o nome deve fazer sentido no negócio.

- **Evite repetir nome da tabela no nome da medida:** No Power BI, ao usar a medida em um visual, geralmente o nome da medida já é autoexplicativo sem precisar mencionar a tabela. Por exemplo, se você tem uma medida de soma de vendas, “Total Vendas” basta – não há necessidade de “Vendas[Total Vendas]” ou “Total Vendas Fato”. Da mesma forma, não precisa colocar o nome da tabela de dimensão na medida (ex.: não é preciso “Contagem Clientes (d_Cliente)”). Uma exceção é se você tiver medidas muito similares em contextos diferentes e precisar diferenciá-las. Mas, de maneira geral, mantenha os nomes das medidas curtos e sem redundância.

- **Diferencie medidas de cálculo versus colunas físicas:** Não use exatamente o mesmo nome de uma coluna existente para nomear uma medida, pois isso pode causar confusão. Por exemplo, se na tabela f_Vendas existe a coluna Quantidade, não crie uma medida também chamada “Quantidade” para somar essa coluna – chame de Total Quantidade Vendida ou algo do tipo. Assim, você e outros sabem quando estão usando a medida agregada versus a coluna granular. Alguns projetos adotam prefixos ou sufixos para medidas (por ex., prefixar todas as medidas com “m_” ou colocar “(M)” no final). Isso não é obrigatório e o Power BI já distingue medidas com um ícone sigma (∑), mas se a equipe preferir, pode-se adotar. Apenas tenha certeza de explicar esse padrão para todos envolvidos.

### Criar tabelas de medidas e pastas

- **Centralize medidas em tabelas dedicadas:** À medida que o número de medidas cresce, pode ficar confuso encontrá-las se elas estiverem misturadas em várias tabelas (por padrão, uma medida é armazenada na tabela onde foi criada). Uma boa prática é criar uma tabela de medidas – uma tabela “fake” sem dados, usada apenas para guardar medidas. Por exemplo, você pode usar o recurso Entrar Dados para criar uma tabela chamada “_Medidas” ou “Measures” com uma única linha e coluna fictícia, e depois ocultar essa coluna. Em seguida, mover (redefinir) todas as medidas existentes para essa tabela. Isso faz com que no painel de Campos você tenha uma seção só com medidas, facilitando localizar e gerenciar. Você pode até criar múltiplas tabelas de medidas se quiser categorizar (ex: “Medidas Financeiras”, “Medidas RH”, etc.), mas cuidado para não exagerar – muitas tabelas de medidas podem complicar novamente. Uma ou poucas agrupações lógicas geralmente bastam.

- **Utilize pastas de exibição:** Outra funcionalidade útil são as pastas de medidas. No editor de modelo, você pode atribuir às medidas (e também colunas, hierarquias) um “Nome da Pasta de Exibição”. Assim, dentro da tabela (seja uma tabela de medidas ou qualquer outra), as medidas podem ser organizadas hierarquicamente. Por exemplo, você pode colocar Total Vendas, Total Custos, Lucro dentro da pasta “Financeiro”, e medidas de clientes em outra pasta “Clientes”. Essas pastas aparecerão no painel de Campos dentro da tabela, apenas como forma de organização visual (não afeta nada na lógica). As pastas são especialmente úteis se você preferiu não usar uma tabela de medidas separada – assim, dentro da própria tabela de fato, você pode segregar medidas em subgrupos. Para criar pastas, selecione uma medida no modelo, na janela de propriedades defina “Pasta de Exibição” (digitando algo como “Financeiro\Receita” para subpastas).

### Escrita DAX (Variáveis, Comentários)

- Use variáveis (VAR) para cálculos complexos: Ao escrever medidas DAX não tenha receio de usar variáveis. Elas ajudam a quebrar a lógica em partes nomeadas, melhorando tanto a performance quanto a legibilidade

- Além disso, variáveis permitem isolar lógica para depurar: você pode testar retornar a variável sozinha temporariamente para verificar seu valor, o que facilita debug

```dax
Lucro % =
    VAR _Vendas = [Total Vendas]
    VAR _Custos = [Total Custos]
    VAR _return = DIVIDE(_Vendas - _Custos, _Vendas)
    RETURN _return
```

- **Comente suas fórmulas:** O DAX permite comentários de linha usando // ou comentários de múltiplas linhas entre /* ... */. Faça uso disso em medidas complexas ou de lógica menos óbvia. Por exemplo:

```dax
// Calcula clientes ativos (aqueles sem data de cancelamento até hoje)
Clientes Ativos =
CALCULATE(
    DISTINCTCOUNT(Clientes[ClienteID]);
    FILTER(Clientes; Clientes[DataCancelamento] = BLANK() || Clientes[DataCancelamento] > TODAY())
)
```

> Acima, o comentário explica em português simples o objetivo. Isso ajuda outro desenvolvedor (ou você mesmo, meses depois) a entender o porquê daquele cálculo. Você pode comentar trechos inteiros temporariamente para testar também. Só lembre que, diferentemente do M do Power Query, os comentários em DAX feitos no editor de medidas não são preservados visualmente na interface após salvar a medida (o editor “limpa” a formatação).

- Formate o código DAX: Medidas mais longas devem ser formatadas com quebras de linha e indentação para facilitar leitura. No exemplo de variável acima, note o uso de quebras e recuo. Isso não afeta o resultado, mas faz diferença para quem lê. Você pode usar ferramentas como DAX Formatter (do SQLBI, disponível online) que automaticamente arrumam a formatação da fórmula DAX. Um código bem formatado reduz erros e melhora a manutenibilidade. Uma dica: adote um estilo padrão (por exemplo, funções DAX sempre em maiúsculas, como CALCULATE, nomes de colunas sempre prefixados com tabela, etc.) e siga consistentemente. O próprio Power BI às vezes ajusta a capitalização automaticamente. O importante é coerência.

- Melhores práticas DAX gerais: Prefira medidas a colunas calculadas (já discutido), use funções especializadas quando possível (por ex, DIVIDE no lugar de operador / para tratar divisão por zero, COALESCE para substituir BLANKs, etc.), evite iteradores desnecessários (sempre que puder usar um cálculo simples ao invés de um FILTER + SUMX, faça-o). E teste suas medidas em cenários de filtro variados para garantir robustez (ex.: o Grand Total do relatório mostra valor coerente? A medida lida bem com ausência de dados?). Seguir as convenções e dicas acima vai te poupar tempo de depuração e tornar o modelo DAX muito mais confiável.

> **Referências:** [DAX best practices](https://learn.microsoft.com/dax/best-practices)

### Grupos de Cálculo

- **Reduza medidas redundante:** Os Grupos de Cálculo são um recurso avançado do modelo tabular (suportado no Power BI a partir de 2020, inicialmente via Tabular Editor e agora também nativo no Desktop). Em essência, eles permitem definir um conjunto de cálculos (chamados itens de cálculo) que podem ser aplicados sobre quaisquer medidas existentes. Com isso, você evita criar dezenas de medidas estáticas. Um exemplo clássico é Time Intelligence: ao invés de ter medidas separadas para Total Vendas Mês Atual, Total Vendas Mês Anterior, Total Vendas Acumulado Ano, Total Vendas YOY%, etc., você cria uma única medida base Total Vendas e um Calculation Group chamado “Inteligência de Tempo” com itens como “Mês Atual”, “Mês Anterior”, “Acumulado Ano”, “YOY%”. Esses itens têm fórmulas DAX genéricas usando a função SELECTEDMEASURE() (que referencia a medida em uso). Daí, no relatório, você coloca o Calculation Group como uma slicer ou coluna em matriz, e com isso consegue ver Total Vendas sob diferentes cálculos de tempo sem precisar de N medidas para cada variação. Em resumo, grupos de cálculo economizam medidas ao parametrizar o tipo de cálculo a aplicar.

- **Quando usar:** Avalie usar Calculation Groups quando notar um “padrão” repetitivo de medidas no seu modelo. Time intelligence é o caso mais evidente (vários KPIs querendo versões MTD, YTD, YOY). Outro caso: calculações financeiras como diferentes cenários (Realizado, Orçado, Projetado) – você pode fazer um calc group “Cenário” ao invés de ter medidas separadas para cada métrica em cada cenário. Mais um: formatações dinâmicas – calc groups permitem até definir formatação numérica dinâmica por item (ex: mostrar % para um item YOY% e número normal para um item Valor). Isso facilita manutenções, pois se a lógica de YOY mudar, você ajusta em um lugar só em vez de editar N medidas.

> Referência: [Grupos de Cálculo](https://learn.microsoft.com/en-us/power-bi/transform-model/calculation-groups)

### Parâmetros de Campos

Os Parâmetros de Campo são um recurso relativamente novo que permite ao usuário final trocar dinamicamente dimensões ou medidas em um visual através de seleções em um slicer. Em vez de usar múltiplas páginas ou bookmarks para oferecer diferentes cortes de análise, você pode usar um parâmetro de campo e um slicer para que o usuário escolha, por exemplo, qual métrica mostrar em um gráfico ou qual categoria usar no eixo X. Isso traz flexibilidade aos relatórios, oferecendo uma experiência mais personalizada sem precisar criar dezenas de visuais escondidos.

Para criar, vá em Modelagem > Novo parâmetro > Campos. 

Use o nome da tabela criada `p_<Nome>` para permitir ao usuário alternar medidas ou dimensões via slicer, economizando páginas e visuais.

> **Referências:** [Field parameters](https://learn.microsoft.com/power-bi/create-reports/field-parameters)

---

## Design e Visualização

### Tema Personalizado JSON

Adote um tema de cores personalizado: O Power BI permite aplicar temas aos relatórios, que configuram paleta de cores, fontes, e estilos visuais de forma consistente. Em vez de manualmente ajustar cada gráfico para usar as cores da sua empresa, por exemplo, você pode criar ou importar um arquivo JSON de tema contendo essas definições. Isso assegura que todos os visuais compartilhem o mesmo esquema de cores e formatação padrão, dando unidade estética ao relatório. Você pode começar customizando um tema pelo menu do Power BI (Exibir > Temas > Personalizar Tema) – ajuste cores principais, cor de fundo, fontes etc – e depois exportar para JSON para refinamentos adicionais. No JSON, é possível controlar praticamente todos os elementos (cores de cada categoria, estilos de linhas, formato de números, propriedades padrão de visuais específicos, etc.).

> **Referências:** [Report themes](https://learn.microsoft.com/power-bi/create-reports/desktop-report-themes)

### Layout Consistente via Figma

Faça o layout no Figma.

- **Respeite espaçamentos e alinhamentos:** Um dos aspectos mais notados em design de relatórios é o espaçamento. Margens consistentes nas bordas da página, espaço uniforme entre visuais e entre títulos e gráficos, tudo isso transmite ordem. Adote uma grid (ex.: espaçamento base de 8px ou 10px) e alinhe os elementos a essa grid – o Figma auxilia muito nisso com recursos de auto-layout e grades. Depois, no Power BI Desktop, você pode replicar essas posições usando o painel de formatação geral de cada visual (definindo tamanho e posição X,Y numéricos) ou arrastando com snap-to-grid ligado. Evite “empilhar” muitos visuais sem alinhamento – use as guias do Power BI para garantir que tamanhos de gráficos similares sejam iguais, etc.

- **Consistência de estilo:** Se um visual tem cantos arredondados e sombra, todos deveriam ter (a não ser por alguma diferença proposital). Se os títulos são fonte Arial 14pt negrito, mantenha igual em todos. Pequenos detalhes de consistência fazem o produto final parecer profissional e polido. Aqui novamente, usar um tema personalizado já configura muitos desses padrões automaticamente. Mas caso ajuste manualmente, lembre de aplicar a todos. Ferramentas de design permitem criar um “design system” – por exemplo, no Figma você pode ter componentes representando um card de KPI, um gráfico, etc. Tente seguir um sistema semelhante no PBI: todos cards de KPI com mesmo background e formato, todos gráficos da mesma categoria com cores correspondentes, etc.

- **Não coloque texto estático ou ícones no fundo (layout) se puder evitar:** Uma dica importante: embora seja tentador colocar no design de background (seja feito no Figma ou outra ferramenta) elementos como rótulos, ícones decorativos e textos, tenha cuidado. Evite textos explicativos ou títulos já “queimados” no fundo estático, porque se precisar mudar ou traduzir depois, é muito mais difícil (tem que editar a imagem de fundo). É melhor usar as caixas de texto nativas do Power BI para títulos, legendas, etc., pois assim você edita rapidamente no Desktop e elas também suportam dinâmicas (expressões de título). Ícones ou imagens, prefira inseri-los também via Power BI (como imagens ou formas) – só use no fundo se for meramente decorativo e não precisar alterar. Além disso, textos no fundo não respondem a tema (não mudam cor com tema claro/escuro, por ex.) e não aparecem em modo de Alto Contraste (acessibilidade). Portanto, para acessibilidade e flexibilidade, mantenha o layout base apenas com cores/shapes gerais, e insira escritos e ícones via o próprio PBI.

### Interação entre Visuais

No Power BI, quando clicamos em um elemento de um visual (por exemplo, uma barra em um gráfico), os outros visuais por padrão realçam (highlight) os dados relacionados em vez de filtrar completamente. Esse comportamento padrão de cross-highlight muitas vezes confunde os usuários, pois os gráficos ficam semitransparentes mostrando partes destacadas, em vez de mostrar somente o que foi selecionado. Uma boa prática é alterar as interações para cross-filter (filtro), de forma que a seleção em um visual filtre os demais completamente

Configure interações para **Filtrar** em vez de Realçar nas opções de relatório como padrão, tornando o comportamento mais intuitivo.

> **Referências:** [Visual interactions]([https://learn.microsoft.com/power-bi/create-reports/service-interactions](https://learn.microsoft.com/pt-br/power-bi/create-reports/power-bi-reports-filters-and-highlighting)

### Padrões Visuais e Dashboards

- **Evite telas poluídas:** Em design de dashboard, menos é mais. Procure não sobrecarregar a página com muitos gráficos, textos e elementos decorativos. Concentre-se nas informações chave – aquilo que realmente suporta as decisões ou insights esperados. Uma tela poluída cansa o usuário e dificulta encontrar o que é importante. Prefira ter 5 visualizações bem selecionadas e organizadas do que 15 espremidas. Se você tem muita informação para mostrar, considere quebrar em várias páginas temáticas ou usar recursos como drill-through (ir para detalhes ao clicar) e tooltips de página (mostrando detalhes sob demanda ao passar o mouse). Isso mantém a visão principal limpa, com detalhamento acessível apenas quando desejado. Utilize também espaço em branco estrategicamente – margens e respiros entre visualizações ajudam a segmentar mentalmente as seções.

- **Escolha adequada de tipos de visual:** Siga as melhores práticas de visualização de dados: use gráficos de barras/colunas para comparar categorias, use linhas para séries temporais, use pizza/donut apenas para partes de um todo simples (e não muitas categorias), use tabelas somente se for realmente necessário apresentar detalhes textuais ou numéricos que não cabem em gráfico. Evite gráficos 3D ou muito exóticos que mais confundem do que explicam. Lembre-se que o objetivo principal é comunicar dados de forma clara, não impressionar com design arrojado. Claro, você pode – e deve – fazer bonito, mas sem sacrificar a clareza. Por exemplo, um KPI card com um número grande e um indicador de tendência pode comunicar um dado principal melhor do que um gráfico saturado de info. Sempre pergunte: “qual a história que quero contar com esse visual? ele está claro a alguém de fora?”. Se não, simplifique.
Consistência de cores e legendas: Defina um código de cores e mantenha-o. Se “Receita” é azul em um gráfico e “Despesa” é vermelha, siga isso nos outros gráficos também. O Power BI por padrão tenta manter cores iguais para categorias iguais (se você usar um campo consistentemente), mas em medidas calculadas ou diferentes visualizações às vezes é preciso ajustar manualmente. Faça esse esforço. Isso ajuda o usuário a criar associação (ex: azul sempre receita). Da mesma forma, use rótulos e títulos coerentes: se em uma página você chama de “Clientes Ativos”, não chame em outra de “Clientes em Atividade” – padronize terminologia. Atenha-se à linguagem do negócio e, se possível, use termos que os usuários usam no dia a dia (pode ser que internamente usem “clientes ativos” mesmo, então vá por aí).

- **Títulos e explicações:** Cada visual deve ter um título que indique claramente o que ele representa (eixo X vs Y, filtro aplicado, etc.). Um bom título responde “o quê” e possivelmente “onde/quando”. Ex: “Vendas Mensais por Região (Últimos 12 meses)”. Se o título for muito longo, use o complemento no subtítulo ou tooltip informativo. Você pode usar tooltips de explicação (através de botões de informação – um ícone de “i” que ao passar o mouse mostra uma descrição). Isso é útil para métricas complexas ou filtros implícitos. Por exemplo, um pequeno “i” ao lado do título “Churn de Clientes” que explique “% de clientes cancelados em relação ao total de clientes ativos no início do mês”. Essas dicas tornam o dashboard mais autoexplicativo.

- **Acessibilidade e tamanho de texto:** Certifique-se de que os textos estão legíveis – fonte não muito pequena, contraste bom (texto escuro em fundo claro ou vice-versa). O Power BI tem configurações de tema de alto contraste para usuários que precisem, mas você já deve de saída evitar combinações de cores que pessoas com daltonismo não distinguem (por ex, vermelho vs verde puros – se isso for necessário, adicione talvez um ícone ou texto indicando). Pequenos cuidados assim ampliam seu público.

---

## Versionamento e Publicação

Controlar versões e histórico do desenvolvimento do relatório é fundamental, especialmente em ambientes corporativos ou de colaboração entre vários desenvolvedores. Além disso, a etapa de publicação no serviço Power BI deve ser feita com critério, garantindo que a versão certa chegue aos usuários e que se mantenha um backup do trabalho. Aqui separamos dicas de versionamento para projetos simples (individuais) e para projetos complexos (com várias pessoas e integração com ferramentas de version control como Git).

### Versionamento, Histórico e Publicação

#### Para projetos simples (OneDrive + arquivo .pbix)

- **Salve versões incrementalmente:** Se você é o único desenvolvedor do relatório ou sua equipe é pequena, uma abordagem prática de versionamento é utilizar o próprio arquivo .PBIX com nomenclatura de versão ou data. Por exemplo, ao finalizar um conjunto de alterações importantes, salvar como RelatorioVendas_v1.pbix, depois v1.1, v2.pbix e assim por diante, ou anexando data no nome (ex: RelatorioVendas_2023-06-01.pbix). Isso permite reter cópias antigas para referência ou rollback se algo der errado.

- **Utilize OneDrive ou SharePoint:** Colocar o arquivo .pbix em uma pasta sincronizada com OneDrive (ou SharePoint/Teams) traz vantagens: ele mantém um histórico de versões automático do arquivo na nuvem. Por exemplo, o OneDrive for Business guarda múltiplas versões de um mesmo arquivo conforme você o salva. Assim, se precisar recuperar o estado de dois dias atrás, é possível através do histórico de versões no OneDrive (sem precisar ter renomeado manualmente vários arquivos). Além disso, trabalhar a partir de OneDrive facilita colaboração simples – outro colega pode abrir (preferencialmente em modo somente leitura se for simultâneo) ou você pelo menos tem backup em nuvem caso sua máquina apresente problemas.

- **Publicação manual no serviço:** Num cenário simples, provavelmente você irá publicar o relatório no serviço do Power BI (PowerBI.com) manualmente via o botão Publicar. Lembre-se de sempre publicar a versão correta (é comum às vezes confundir e publicar um PBIX errado se houver vários). Uma dica: mantenha o Workspace de teste separado do de produção. Por exemplo, publique primeiro em um workspace só seu ou da equipe para validação; depois use a função Republicar no workspace oficial ou promova via pipeline (se disponível). Em todo caso, guarde sempre uma cópia local do PBIX correspondente ao que está publicado – o serviço não é um repositório de desenvolvimento, pois embora dê para baixar o PBIX publicado (a não ser casos de dataset com aumento incremental), melhor ter sua fonte de verdade local.

- **Controle de alterações:** Mantenha um changelog simples – pode ser em um bloco de notas ou seção na documentação – anotando o que mudou em cada versão (ex: “v1.2 – adicionada página de resumo financeiro; ajustada medida de margem para excluir impostos; etc.”). Isso ajuda na hora de comunicar usuários sobre atualizações e também para você lembrar o histórico. Em projetos simples isso pode ser informal, mas não negligencie: ao revisitar o relatório meses depois, você agradecerá ter notas do que foi feito.

#### Para projetos complexos (Power BI Projects (PBIP) + Git)

- **Use o formato Power BI Project (PBIP):** Para equipes maiores ou desenvolvimento complexo, a Microsoft introduziu o formato .PBIP (Power BI Desktop Project). Diferente do PBIX monolítico, o PBIP quando salvo cria uma pasta com múltiplos arquivos textuais (JSON, XML) representando o relatório e o modelo separadamente. Por exemplo, uma pasta PBIP contém subpastas para Report (com layout visual, definições de visual) e Semantic Model (tabelas, medidas, roles etc.). Esses arquivos podem ser lidos e comparados em controle de versão. Ativar PBIP requer ligar a opção de preview nas configurações do Desktop (em Opções > Recursos em Pré-visualização > Power BI Project). Após ativado, você pode Salvar Como projeto PBIP. A principal vantagem é que isso torna o projeto “amigável ao Git”, isto é, pronto para uso em sistemas de controle de versão como GitHub ou Azure DevOps. Cada alteração que você faz, ao salvar, reflete em diferenças nos arquivos de texto, que podem ser acompanhadas via diffs, commits, branches, etc., assim como se fosse um código fonte de software. Isso traz um nível profissional de versionamento ao Power BI, com histórico detalhado e colaboração paralela mais segura.

---

## Documentação

### Estrutura Ideal

Uma boa documentação de projeto de BI deve cobrir, pelo menos, os seguintes pontos

- Informações Gerais do Projeto

 Nome do projeto/relatório, responsável(s) pelo desenvolvimento, data de criação e últimas atualizações, versão atual, departamento ou cliente solicitante. Inclua também o propósito geral em uma frase. (Ex.: Relatório Vendas Mensais – Responsável: Ana Souza – Versão 1.2 (Atualizado em Mar/2025)).

- Objetivo do Projeto

Descreva o que o dashboard busca resolver ou responder. Quais perguntas de negócio motivaram sua criação? Quais decisões ele subsidia? Deixe claro o porquê do relatório. (Ex.: Este relatório tem por objetivo acompanhar as vendas mensais por região e produto, comparando com metas e ano anterior, para identificar tendências de crescimento ou queda e direcionar ações comerciais.)

- Escopo e Público-Alvo: (opcional, mas útil)

Delimite se necessário o que está ou não incluído e para quem se destina o relatório. Por exemplo: Escopo: vendas nacionais de produtos físicos (não inclui serviços); Público: gerentes regionais de vendas e diretoria comercial.
Fontes de Dados: Liste de onde vêm os dados utilizados. Seja específico: nomes de bancos de dados/tabelas, arquivos Excel/CSV, fontes web, etc. Inclua conexões e credenciais se pertinente (talvez em anexo seguro). Exemplo: Dados de vendas do ERP (SQL Server, DB=ERP_ACME, tabela SalesOrders), Planilha Excel “Metas_Regionais.xlsx” (SharePoint), etc. Isso ajuda na manutenção e na confiabilidade, além de apontar onde atualizar caso a fonte mude.

- Arquitetura

Escrever

- Localização (Workspace)

Escrever
  
- Atualização dos Dados (Automação)

Informe com que frequência os dados são atualizados e como. Ex.: “Atualização agendada diariamente às 7h via gateway”, ou “Dados de planilha precisam ser atualizados manualmente todo início de mês”. Se houver dependências (ex: “deve rodar fluxo de dados antes, às 6h”), coloque. Assim, quem assumir sabe o ciclo. Eventualmente inclua tempo médio de atualização, ou dicas se falhar (ex: “precisa limpar cache X se erro Y ocorrer”).

- Transformações de Dados (ETL)

Resuma as principais limpezas e transformações feitas no Power Query. Por exemplo: “Unimos dados de vendas de duas fontes diferentes; removemos outliers de preços (preço > 1 milhão tratados como erro e excluídos); calculamos coluna de ‘Status Cliente’ ativo/inativo a partir da última compra; etc.” Não precisa transcrever cada passo M, mas os pontos notáveis e lógicas de negócio implementadas no ETL devem constar. Isso permite compreender como os dados brutos foram moldados para o modelo.

- Modelagem de Dados

Descreva a estrutura do modelo: quais são as tabelas fato e dimensões, e como se relacionam (um diagrama ER simples pode ser muito útil aqui, ou uma lista de relações). Destaque particularidades: “Tabela Calendário customizada incluindo feriados nacionais”, “Dimensão Produto consolidada de múltiplas tabelas”, “Relacionamento muitos-para-muitos utilizado entre X e Y por motivo Z”, etc. Se existirem medidas calculadas que atuam como colunas (ex: uma tabela criada via DAX), mencione. O leitor deve entender o “esqueleto” do modelo sem precisar abri-lo.

- Métricas e Cálculos DAX:

Liste as principais medidas DAX com suas definições de negócio. Por exemplo: Margem Bruta (%) – definição: “(Receita – Custo) / Receita, considerando apenas produtos físicos, excluindo impostos”. Você pode inclusive apresentar a fórmula DAX simplificada. O importante é explicar o conceito e eventuais filtros implícitos. Faça isso para KPIs e medidas complexas especialmente. Assim, usuários e futuros devs sabem exatamente o que cada indicador significa e como é obtido.

- Detalhamento das Páginas/Visuais do Relatório

Faça um resumo página a página do dashboard, explicando o que cada página mostra e destacando visuais específicos se necessário. Por exemplo: “Página 1 – Visão Geral: apresenta cartões resumindo Vendas, Custos, Lucro do mês atual vs. anterior; gráfico de barras com Vendas por Região; tendência anual de Vendas em linha...”. Para cada visual complexo, explique: “Gráfico X: compara a % de churn por segmento de cliente, calculada conforme métrica Y”. Isso ajuda principalmente quando há visuais menos triviais ou análises cruzadas que não são óbvias a olho nu.


- Segurança e Compartilhamento

Documente configurações de RLS (Row-Level Security) se houver – quais roles existem e seu filtro (ex: Role GerenteRegional filtra DimRegião[Gerente] = UserPrincipalName()). Liste quem tem acesso ao relatório (workspaces, apps, ou compartilhamentos diretos). Isso também pode incluir quem pode editar versus só visualizar. Registrar essa “matriz de acesso” é importante para governança.


- Melhorias Futuras (Backlog):

Opcionalmente, a documentação pode listar ideias ou demandas de melhoria para versões futuras, para que fique registrado. Ex: “Incluir análise de churn de clientes – previsto para próxima versão”, ou “Otimizar desempenho substituindo agregações por DirectQuery em tabela X”. Isso serve como memória do que ainda poderia ser feito.

--

Seguindo todas essas guidelines – desde a ingestão dos dados até a documentação final – você estará aplicando as melhores práticas de desenvolvimento no Power BI Desktop. O resultado esperado são relatórios performáticos, fáceis de manter e aprimorar, visualmente consistentes e bem recebidos pelos usuários, além de um processo de desenvolvimento mais organizado e profissional. Boas análises e bom desenvolvimento!

> Última revisão: 19‑Jun‑2025
