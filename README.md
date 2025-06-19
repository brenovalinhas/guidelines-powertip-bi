# Powertip – Guidelines de Desenvolvimento Power BI Desktop

> **Objetivo**
> Padronizar a criação de dashboards no **Power BI Desktop** para analistas Jr/Pleno, garantindo consistência, legibilidade e facilidade de manutenção em todos os projetos da Powertip.

---

## Introdução

Estas diretrizes cobrem todo o ciclo de desenvolvimento no Power BI Desktop — da ingestão de dados (ETL) ao controle de versão e documentação. Cada seção traz boas‑práticas, exemplos práticos e, logo abaixo, links de referência oficiais para aprofundamento.

---

## ETL (Power Query)

### Consultas de Referência

* Importe cada fonte **uma única vez** (query *stg\_* com *Enable Load* desativado) e crie **referências** para variações de transformação.
* Exemplo rápido:

  ```powerquery
  // Vendas entregues
  let
      Source        = stg_SAP_Vendas,
      DeliveredOnly = Table.SelectRows(Source, each [Status] = "Entregue")
  in
      DeliveredOnly
  ```

**Referências:** [Power Query – Best practices](https://learn.microsoft.com/power-query)

### Descrição de consultas

* Preencha o campo **Description** da query explicando finalidade, filtro e unidade de medida.
* Nomeie queries com termos de negócio: `Fato Vendas`, `Dim Produto`.
  **Referências:** [Document query properties](https://learn.microsoft.com/power-query/query-properties)

### Nomenclatura de etapas

* Renomeie cada passo M de forma clara, sem espaços: `TipoAlterado_Datas`, `Filtro_AnoAtual`.
* Evite `#"Nome da Etapa"` usando nomes sem espaço.
* Dica: use IA (ChatGPT, Copilot) para sugerir nomes claros no Editor Avançado.
  **Referências:** [Power Query step names](https://learn.microsoft.com/power-query)

### Carregue somente colunas necessárias

* Remova campos irrelevantes antes do load para reduzir memória e acelerar o refresh.

  ```powerquery
  Table.SelectColumns(Source, {"Pedido", "Cliente", "Valor"})
  ```

**Referências:** [Optimize column load](https://learn.microsoft.com/power-query/performance)

### Renomeie colunas com nomes de negócio

* Converta nomes técnicos (`cust_id`) em nomes claros (`ClienteID`).
* Mantenha idioma consistente (PT‑BR ou EN) conforme padrão do projeto.
  **Referências:** [Rename columns](https://learn.microsoft.com/power-query)

### Nomenclatura de tabelas

* **Tabelas importadas**: prefixo `f_` (fatos) ou `d_` (dimensões).
  Ex.: `f_Vendas`, `d_Cliente`
* **Tabelas não importadas** (apenas referência): use nomes descritivos sem espaço, ex.: `ComprasRef`.
  **Referências:** [Data modeling naming conventions](https://learn.microsoft.com/power-bi)

---

## Modelagem de Dados

### Esquema estrela

* Separe **fatos** e **dimensões**; evite snowflake.
  **Referências:** [Designing star schemas in Power BI](https://learn.microsoft.com/power-bi/guidance/star-schema)

### Relacionamentos

* Use cardinalidade \**1:* \*\* das dimensões para a fato; mantenha filtro unidirecional.
* Evite relacionamentos *:* – insira dimensão intermediária quando necessário.
  **Referências:** [Relationships in Power BI](https://learn.microsoft.com/power-bi/transform-model/desktop-relationships-understand)

### Evitar criação de colunas usando DAX

* Crie colunas no Power Query ou fonte; use colunas DAX apenas quando indispensável.
  **Referências:** [Calculated columns vs Power Query](https://learn.microsoft.com/power-bi/guidance/guidance-power-query)

### Tabela Calendário via DAX

* Crie tabela `Calendario = CALENDAR(...)` + `ADDCOLUMNS` para Ano, Mês, etc.; marque como *Date table*.
  **Referências:** (link GitHub)

### Tabela Calendário especial

* Reservado para calendário fiscal ou 4‑4‑5 semanas.
  **Referências:** (link GitHub)

---

## Cálculos, Análises e Fórmulas DAX

### Convenção de nomes

* Medidas: termos de negócio + unidade: `Total Vendas R$`, `Margem %`.
  **Referências:** [Naming measures](https://learn.microsoft.com/dax)

### Criar tabelas de medidas e pastas

* Crie uma tabela vazia **Medidas** e organize pastas: `01 – Financeiro`, `02 – Clientes`.
  **Referências:** [Organize measures](https://learn.microsoft.com/power-bi)

### Escrita DAX (variáveis, comentários, etc.)

* Use `VAR _Nome` para partes repetidas e `//` ou `/* */` para comentários.

  ```dax
  Lucro % =
      VAR _Vendas = [Total Vendas]
      VAR _Custos = [Total Custos]
      RETURN DIVIDE(_Vendas - _Custos, _Vendas)
  ```

**Referências:** [DAX best practices](https://learn.microsoft.com/dax/best-practices)

### Grupos de cálculo

* Utilize *Calculation Groups* para Time Intelligence e cenários, reduzindo medidas duplicadas.
  **Referências:** [Calculation groups](https://learn.microsoft.com/power-bi/transform-model/desktop-enhanced-modeling)

### Parâmetros de campos

* Crie parâmetros (`p_SelecionarMetrica`) para trocar medidas/dimensões via slicer.
  **Referências:** [Field parameters](https://learn.microsoft.com/power-bi/create-reports/field-parameters)

---

## Design e Visualização

### Utilizar tema personalizado JSON

* Aplique paleta, fontes e transparência 100 % em fundos via arquivo `.json`.
  **Referências:** [Report themes](https://learn.microsoft.com/power-bi/create-reports/desktop-report-themes)

### Layout consistente via Figma

* Defina espaçamentos e grid antes de construir; textos e ícones devem ser adicionados no Power BI, não no background.
  **Referências:** [Designing dashboards](https://learn.microsoft.com/power-bi)

### Interação entre os visuais

* Altere padrão para **Filtrar** (não Realçar) em *Arquivo > Opções > Configurações de Relatório*.
  **Referências:** [Visual interactions](https://learn.microsoft.com/power-bi/create-reports/service-interactions)

### Padrões visuais e dashboards

* Evite poluição visual; priorize clareza, consistência de cores e hierarquia de informação.
  **Referências:** [Dashboard design guidelines](https://learn.microsoft.com/power-bi)

---

## Versionamento e Publicação

### Versionamento, histórico e publicação

* Use histórico de OneDrive ou SharePoint para `.pbix`; mantenha nomeação clara (`Projeto_v1.2.pbix`).
  **Referências:** [SharePoint version history](https://support.microsoft.com/office)

#### Para projetos simples

* Controle via OneDrive; backups automáticos de versões.

#### Para projetos complexos

* Converta para **.pbip** e versione com Git (GitHub, Azure DevOps).
  **Referências:** [Power BI Project & Git](https://learn.microsoft.com/power-bi/create-reports/pbip-overview)

---

## Documentação

### Estrutura ideal da documentação

1. Objetivo e escopo do relatório
2. Fontes de dados e transformações principais
3. Modelo estrela (diagrama)
4. Definição das principais medidas/KPIs
5. Páginas e visuais explicados
6. Processo de atualização e segurança (RLS)
7. Log de versões e backlog de melhorias

### Dicas

* Documente durante o desenvolvimento, não no fim.
* Utilize campos **Descrição** no modelo e comentários DAX para auto‑documentação.
* Versione a documentação junto ao projeto (Markdown no mesmo repo Git).
* Ferramentas úteis: DAX Formatter, Documenter (SQLBI), ALM Toolkit.

**Referências:** [Power BI documentation practices](https://learn.microsoft.com/power-bi)

---

> Última revisão: 19‑Jun‑2025
