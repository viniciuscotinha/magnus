# 🏭 Simple Pro — Guia do Analista de Produção

> **Para quem é este documento:** analista de produção / BI que precisa entender o sistema **Simple Pro** (Dynamics 365) para modelar dados, cruzar informações e construir relatórios.
> **O que ele responde:** o que cada tabela é e para que serve, como as tabelas se ligam, quais campos importam para análise, quais **métricas de produção** dá para calcular e **onde cada dado nasce**.
> **Fontes:** diagrama ER `DiagramaSimplePro.drawio`, código dos plugins C# (`CSSolutions\MagnusSolucoes`), integração Bling (`bling_api`, `CrmBling`, `ETLBlingMagnusCabelo`, `SincronizadorBling`), PCFs e Web Resources JS. Este guia complementa e corrige o `SISTEMA_MAGNUS.md`.

---

## 1. O que é o Simple Pro (em 1 minuto)

Sistema de **gestão de estoque e produção de cabelos** (postiços/apliques) construído sobre **Microsoft Dynamics 365 / Dataverse**. Ele cobre o ciclo completo:

**Compra da matéria-prima → Conferência/Separação → Estoque → Produção (serviços por departamento) → Expedição → Venda (Bling)**

- O **cabelo físico** é a unidade central (`dev_e_cabelo`): tem peso (Kg), tamanho (cm), tipo e lote.
- A produção é organizada em **Ordens de Produção — OP** (`dev_p_demanda`), que contêm **serviços/atividades** (`dev_servico`) executados por **departamentos** (`dev_g_departamento`), cada serviço agindo sobre um conjunto de cabelos.
- Tudo o que acontece com um cabelo (pesagem, corte, divisão, mesclagem, movimentação, balanço) gera **histórico** rastreável.

---

## 2. ⚠️ Onde os dados vivem (leia antes de montar qualquer relatório)

Existem **duas bases distintas**, e isso muda de onde você puxa cada coisa:

| Fonte | O que tem | Uso para análise |
|---|---|---|
| **Dataverse (Simple Pro)** — org Dynamics de produção | Todas as entidades `dev_*` de **PRODUÇÃO e ESTOQUE** (OP, serviço, cabelo, pausas, balanço, tempos por departamento) | **É a fonte da produção.** Dá para consumir via Dataverse connector / TDS endpoint / OData. |
| **PostgreSQL `bling`** (host `66.94.96.165`, base `bling`) | Dados **COMERCIAIS** do Bling: pedidos de venda/compra, itens, produtos, contatos, vendedores — carregados pelo **ETL** (`ETLBlingMagnusCabelo`) | **É a fonte dos `.pbix` comerciais** (vendas, compras MP, comissões, estoque valorizado). Tabelas em `snake_case`. |

> **Regra de ouro do cruzamento:** produção (Dataverse) × comercial (Postgres) **não têm chave estrangeira comum** — o elo é a **chave de negócio**: o **número do pedido de venda** liga o pedido Bling à OP (`dev_p_demanda`), e o **número do pedido de compra** liga a compra ao lote (`dev_lote`). Sempre **segmente por empresa** (`dev_g_empresa` / `empresa_id`) — o ambiente é **multi-empresa**.

**Detalhes do ETL** (bom saber para confiabilidade do dado):
- Cargas **incrementais** por janela de data alterada: vendas/produtos ≈ últimos **12 dias**; **compras desde 2022-09-01** (histórico completo).
- Roda em lote/agendado, upsert por PK. Se um relatório "não vê" venda antiga, é porque a janela incremental não recarregou — não é erro de produção.
- Tabelas Postgres principais: `pedido_venda`, `pedido_venda_item`, `pedido_venda_parcela`, `pedido_compra`, `pedido_compra_item`, `produto`, `contato`, `vendedor_contato`, `situacao`, `empresa`.

---

## 3. O coração do modelo (as 5 tabelas que você vai usar sempre)

```
                 dev_g_empresa (empresa)         dev_g_departamento (setor)
                        │                                  │
                        ▼                                  ▼
   dev_p_demanda ──1:N──►  dev_servico  ──1:N──►  dev_servico_x_cabelo  ◄──N:1── dev_e_cabelo
     (OP / Pedido)          (atividade)              (junção serviço↔cabelo)      (cabelo físico)
        │  │                   │  ▲                        (qntd_in / qntd_out)        │
        │  │                   │  └── dev_processo (etapa) │                           │
        │  │                   ▼                           ▼                           ▼
        │  └─► dev_config_dep_x_op   dev_pausa        dev_historico_cabelo   dev_lote / dev_tipocabelo /
        │      dev_log_op_x_depto    (pausa serviço)  dev_transformacaocabelo   dev_e_deposito
        └─► dev_pausas_op (pausa OP c/ motivo)
```

- **`dev_p_demanda`** (OP) → contém vários **`dev_servico`** (etapas de produção).
- Cada **`dev_servico`** liga-se a vários **`dev_e_cabelo`** pela junção **`dev_servico_x_cabelo`** (que guarda **peso que entrou e peso que saiu** de cada cabelo naquele serviço).
- Cada serviço pertence a um **`dev_p_processo`** (a etapa: lavar, tingir, etc.) e roda num **`dev_g_departamento`**.
- Tempo/pausas ficam em `dev_servico` (líquido), `dev_pausa` (pausa de serviço), `dev_pausas_op` (pausa de OP com motivo), `dev_config_dep_x_op` e `dev_log_op_x_departamento` (tempo por departamento).

---

## 4. Dicionário de tabelas relevantes para análise

> Legenda: 🔑 = tabela-chave para produção · ⭐ = campo mais útil para análise. Campos-padrão do Dataverse (`statecode`, `statuscode`, `createdon`, `modifiedon`, proprietário) existem em **todas** as tabelas e não são repetidos — mas note: **`statecode` 0=Ativo / 1=Inativo** é usado como "exclusão lógica" (cabelo mesclado/agrupado/sumido no balanço vira Inativo).

### 4.1 ⚙️ Produção (o núcleo)

#### 🔑 `dev_p_demanda` — Ordem de Produção (OP)
Um pedido/ordem que agrupa os serviços de produção. Pode ser tipo **Pedido** (vinculado a venda Bling) ou **Estoque** (produção para estoque).
| Campo ⭐ | Tipo | Uso na análise |
|---|---|---|
| ⭐ `dev_numero` | texto | Número da OP / pedido. **Chave para cruzar com venda Bling.** |
| ⭐ `dev_tipo` | OptionSet | `775730000` = Pedido; outro = Estoque. |
| ⭐ `dev_empresa` | Lookup → `dev_g_empresa` | Empresa dona da OP (segmentação). |
| ⭐ `dev_departamento` | Lookup → `dev_g_departamento` | Departamento **atual** da OP. |
| ⭐ `dev_inicio` | DateTime | Início **real** da produção (preenchido ao iniciar o 1º serviço). |
| ⭐ `dev_previsaoentrega` / previsão por departamento | DateTime | Prazo previsto (calculado em **horas úteis**). |
| ⭐ finalizado (data + flag) | DateTime/bool | Conclusão da OP → **lead time**. |
| ⭐ atrasado / atrasado por departamento | bool | Sinalização de SLA estourado. |
| ⭐ `dev_pausado` | Lookup → `dev_pausas_op` | Pausa **ativa** da OP (se preenchido, OP está pausada). |
| prioridade / motivo / priorizado por | bool/texto | Análise de fila e priorização. |
| `dev_servico_padrao` | Lookup → `dev_servico` | Serviço "Cabelos sem Serviço" (buffer de cabelos da OP). |

#### 🔑 `dev_servico` — Atividade de Produção (serviço/etapa)
A unidade de trabalho: um serviço executado sobre cabelos, dentro de uma OP, numa etapa (`dev_processo`) e departamento. **É a principal tabela de TEMPO.**
| Campo ⭐ | Tipo | Uso na análise |
|---|---|---|
| ⭐ `dev_demandareferente` | Lookup → `dev_p_demanda` | OP a que pertence. |
| ⭐ `dev_processo` | Lookup → `dev_p_processo` | Etapa de produção. |
| ⭐ `dev_departamento` | Lookup → `dev_g_departamento` | Onde foi executado. |
| ⭐ `dev_inicio` / `dev_fim` | DateTime | Duração bruta do serviço. |
| ⭐ `dev_tempodeservicototal` | texto `hh:mm:ss` | **Tempo líquido** = fim − início − pausas (calculado pelo plugin). |
| ⭐ `dev_tempodepausatotal` | texto `hh:mm:ss` | Tempo total pausado no serviço. |
| ⭐ `dev_situacao` | OptionSet | Situação (7 estados — ver §7). Base de dashboards de status. |
| ⭐ `dev_servicofinalizado` | int/bool | Serviço encerrado (1). |
| ⭐ `dev_epadrao` | bool | `true` = serviço padrão "Cabelos sem Serviço" (**excluir das métricas de trabalho real**). |
| ⭐ entregue / entregue parcialmente | bool | Entrega ao próximo processo. |
| `dev_ultimapausa` | Lookup → `dev_pausa` | Última pausa registrada. |

#### 🔑 `dev_servico_x_cabelo` — Serviço × Cabelo (junção N:N) — **tabela de PESO por serviço**
Liga cada cabelo a cada serviço e guarda o peso na entrada e na saída — base de **rendimento/perda**.
| Campo ⭐ | Tipo | Uso na análise |
|---|---|---|
| ⭐ serviço | Lookup → `dev_servico` | (schema: `dev_servico_x_cabelo_servico_x_servico`) |
| ⭐ cabelo | Lookup → `dev_e_cabelo` | (schema: `dev_servico_x_cabelo_x_cabelo`) |
| ⭐ Quantidade Entrou (`qntd_in`) | Decimal (Kg) | Peso do cabelo **ao entrar** no serviço. |
| ⭐ Quantidade Saiu (`qntd_out`) | Decimal (Kg) | Peso **ao sair**. **Perda = Σ in − Σ out.** |
| ⭐ `dev_coletadoinicio` / `dev_coletadofim` | bool | Se pesou no início / no fim. |
| ⭐ `dev_devolvido` | bool | Cabelo devolvido sem processar (retrabalho/desistência). |
| ⭐ data de entrega / data de finalização | DateTime | Marcos de pesagem. |
| é mesclado / foi mesclado | bool | Marca participação em mesclagem. |

#### `dev_p_processo` — Etapas de Produção (configuração)
Define cada etapa e **suas regras** (se pesa, se muda o cabelo, sequência).
| Campo ⭐ | Tipo | Uso |
|---|---|---|
| ⭐ `dev_departamento` | Lookup → `dev_g_departamento` | Departamento da etapa. |
| ⭐ `dev_proxima_etapa` | Lookup → `dev_p_processo` (auto-ref) | Sequência/roteiro de produção. |
| ⭐ pesa no início / pesa no fim | bool | Determina se há pesagem naquela etapa. |
| ⭐ muda cabelo / muda tamanho / une cabelo | bool | Se a etapa transforma o cabelo (afeta rendimento). |
| de espera / deixa criar serviço | bool | Regras de fluxo. |

#### `dev_config_dep_x_op` — Tempo por Departamento × OP — **tabela de TEMPO por setor**
Um registro por (OP × departamento). Acumula o tempo que a OP passou em cada setor.
| Campo ⭐ | Tipo | Uso |
|---|---|---|
| ⭐ `dev_op` | Lookup → `dev_p_demanda` | OP. |
| ⭐ `dev_departamento` | Lookup → `dev_g_departamento` | Setor. |
| ⭐ Tempo Corrido | número | **Horas úteis reais** que a OP passou no setor. |
| ⭐ Tempo Restante | número | Horas úteis previstas ainda no setor (base do prazo). |

#### `dev_log_op_x_departamento` — Movimentação da OP entre departamentos
Log de cada transferência. O `createdon` de cada linha = **carimbo de entrada** no departamento destino → roteiro cronológico da OP.
| Campo ⭐ | Tipo | Uso |
|---|---|---|
| ⭐ `dev_op` | Lookup → `dev_p_demanda` | OP. |
| ⭐ `dev_departamento` | Lookup → `dev_g_departamento` | Destino. |
| ⭐ `dev_departamento_origem` | Lookup → `dev_g_departamento` | Origem. |
| ⭐ `createdon` | DateTime | Momento da transferência. |

#### Pausas
- **`dev_pausas_op`** — pausa **da OP**, com **motivo** (`dev_motivo_selecao` → `dev_motivos_pausas`), `dev_inicio`/`dev_fim`. Use para **tempo parado por motivo**.
- **`dev_pausa`** — pausa **do serviço** (`dev_servicoid`, início/fim da pausa). Alimenta `dev_tempodepausatotal`.
- **`dev_motivos_pausas`** — tabela de referência dos motivos.

### 4.2 📦 Estoque, cabelo e rastreabilidade

#### 🔑 `dev_e_cabelo` — Cabelo (item central de estoque)
| Campo ⭐ | Tipo | Uso |
|---|---|---|
| ⭐ `dev_quantidade` | Decimal | **Peso em Kg** (atenção: Kg, não gramas). |
| ⭐ `dev_tamanho` | int | Tamanho real em cm. |
| ⭐ `dev_tamanho_considerado` | OptionSet | Tamanho normalizado — **valor real = valor % 1000** (ver §7). |
| ⭐ `dev_tipoid` | Lookup → `dev_tipocabelo` | Tipo/categoria. |
| ⭐ `dev_lotedocabelo` | Lookup → `dev_lote` | Lote de origem. |
| ⭐ `dev_deposito` | Lookup → `dev_e_deposito` | Depósito atual. |
| ⭐ `dev_disponivel` | bool | Disponível para uso. |
| ⭐ `dev_ultimo_processo` | Lookup → `dev_p_processo` | Última etapa por que passou. |
| ⭐ `dev_not_in_balanco` | Lookup → `dev_balanco` | Preenchido = cabelo **não encontrado** num balanço (sumiu). |
| `dev_id_original` / `dev_iddivisao` / `dev_cabelo_id` | int/texto | Identidade e genealogia de divisão. |

#### `dev_historico_cabelo` — Histórico do Cabelo (auditoria antes/depois)
Snapshot a cada operação (coleta de peso, finalização, divisão, agrupamento, balanço). **Base de variação de peso/tamanho/tipo.**
| Campo ⭐ | Tipo | Uso |
|---|---|---|
| ⭐ `dev_cabelo` | Lookup → `dev_e_cabelo` | Cabelo. |
| ⭐ `dev_servico` | Lookup → `dev_servico` | Serviço relacionado (se houver). |
| ⭐ `dev_movimentacao` | Lookup → `dev_movimentacoes_cabelo` | Movimentação relacionada. |
| ⭐ Quantidade / Quantidade Final | Decimal | **Peso antes / depois.** |
| ⭐ Tamanho / Tamanho Final; Tipo / Tipo Final | int/Lookup | Mudança de tamanho e tipo. |
| ⭐ Balanço | bool | Registro originado de balanço. |

#### `dev_transformacaocabelo` — Transformação (genealogia)
Grafo origem→destino das operações que **transformam** cabelos.
| Campo ⭐ | Tipo | Uso |
|---|---|---|
| ⭐ `dev_cabeloorigem` / `dev_cabelodestino` | Lookup → `dev_e_cabelo` | Origem e destino. |
| ⭐ `dev_tipo` | texto | `"Divisão de Cabelos"` / `"Agrupamento de Cabelos"` / `"União de Cabelos"` (mesclagem). |
| ⭐ `dev_servicoid` | Lookup → `dev_servico` | Onde ocorreu. |

#### `dev_movimentacoes_cabelo` — Entrada e Saída (entre depósitos)
| Campo ⭐ | Tipo | Uso |
|---|---|---|
| ⭐ `dev_cabelo` | Lookup → `dev_e_cabelo` | Cabelo movimentado. |
| ⭐ `dev_tipo` | bool | **true = Entrada / false = Saída**. |
| ⭐ `dev_deposito` / `dev_deposito_origem` | Lookup → `dev_e_deposito` | Destino / origem. |
| ⭐ `dev_op` | Lookup → `dev_p_demanda` | OP relacionada (se houver). |
| ⭐ `dev_quantidade` | Decimal | Peso movimentado. |
| concluído / salvo | bool | Estado da movimentação. |

#### `dev_lote` / `dev_lote_mesclado` / `dev_codigo_fornecedor` / `dev_e_deposito` / `dev_tipocabelo`
- **`dev_lote`** — lote de compra: `dev_sigla` (→ fornecedor), `dev_tipodeinsercao` (Nº Pedido/Cód. Fornecedor/Manual), **Total Separado (Kg)**, data de recebimento, nº do pedido de compra (elo com Bling), "é grupo de lote".
- **`dev_lote_mesclado`** — composição percentual de lotes num cabelo mesclado (`dev_lote`, `dev_associado`, **percentual**) → permite ratear peso por lote de origem.
- **`dev_codigo_fornecedor`** — sigla + último número sequencial (gera código do lote).
- **`dev_e_deposito`** — armazém, com flags **padrão estoque / padrão produção / padrão matéria-prima**.
- **`dev_tipocabelo`** — categoria do cabelo; flag "com técnica".

### 4.3 🔍 Conferência de compra (recebimento de MP)

#### `dev_conferencia_compra` — Conferência de Compra
Agrupa as separações de um mesmo **tamanho** dentro de um **lote**.
| Campo ⭐ | Tipo | Uso |
|---|---|---|
| ⭐ `dev_lote` | Lookup → `dev_lote` | Lote conferido. |
| ⭐ Tamanho (cm) | OptionSet | Tamanho nominal. |
| ⭐ Total Separado / Peso Compra (Kg) | Decimal | Peso conferido vs comprado. |
| ⭐ Perda Separação (Kg) e % | Decimal | **Perda no recebimento.** |
| ⭐ data de conferência | DateTime | Quando. |

#### `dev_conferencia` — Separação e Conferência das Medidas
Registro individual de pesagem. Guarda os desdobramentos **NT** (natural) e **TM** (teste de mecha).
| Campo ⭐ | Tipo | Uso |
|---|---|---|
| ⭐ `dev_conferencia_relativa` | Lookup → `dev_conferencia_compra` | Conferência-pai. |
| ⭐ `dev_loteid` | Lookup → `dev_lote` | Lote. |
| ⭐ Peso Separação / Peso Natural / Peso Teste de Mecha (Kg) | Decimal | Desdobramento do peso. |
| ⭐ Total Conferido (Kg); Perda Conferência (Kg e %) | Decimal | **Rendimento da conferência.** |
| ⭐ % NT / % TM / % Separação | texto | Composição. |
| `dev_cabelontreferente` / `dev_cabelotmreferente` | Lookup → `dev_e_cabelo` | Cabelos gerados. |

### 4.4 📊 Balanço (inventário físico)

#### `dev_balanco` — Balanço
| Campo ⭐ | Tipo | Uso |
|---|---|---|
| ⭐ `dev_deposito` | Lookup → `dev_e_deposito` | Depósito inventariado. |
| ⭐ Mês do Balanço | OptionSet | Competência. |
| ⭐ filtros: lote / tipo / tamanho (categoria) | Lookup/OptionSet | Escopo do balanço. |
| ⭐ finalizado / processando | bool | Estado. |

#### `dev_balanco_x_cabelo` — Balanço × Cabelo (contagem)
| Campo ⭐ | Tipo | Uso |
|---|---|---|
| ⭐ `dev_balanco` / `dev_cabelo` / `dev_lote` / `dev_tipo` | Lookup | Chaves. |
| ⭐ `dev_quantidade` | Decimal | **Peso contado** (físico). |
| ⭐ Diferença | Decimal | **Contado − sistêmico** = acurácia de inventário. |
| ⭐ existia cabelo? / registrado | bool | Controle do processamento. |

### 4.5 🏢 Organizacional e comercial
- **`dev_g_departamento`** — setores da fábrica; tem **tempo padrão (horas)**.
- **`dev_g_empresa`** — empresa do grupo **+ credenciais Bling** (token/refresh/client id) — é o **cofre de integração**; use só o **nome** e a flag "é padrão" para análise.
- **`dev_lista_preco`** (preço normal/promo), **`dev_promos`** (campanhas), **`dev_registrodeexecucao`** (log das integrações Bling: nº da venda/compra gatilho e criada, válido, erro).
- **`dev_config_peso`** — parâmetros da balança: **percentual limite de mesclagem** (padrão ~6%) e senha de gestor.
- **`dev_bandeja`** — bandeja de expedição/etiqueta: `dev_foi_impresso`, `dev_data_impressao`, `dev_impresso_por` (rastreabilidade de impressão).

---

## 5. Relacionamentos-chave (para montar o modelo no BI)

> Todas 1:N (lado 1 → lado N). O campo lookup fica na tabela do lado N.

| Lado 1 | Lado N | Campo lookup (no lado N) |
|---|---|---|
| `dev_p_demanda` (OP) | `dev_servico` | `dev_demandareferente` |
| `dev_servico` | `dev_servico_x_cabelo` | `dev_servico_x_cabelo_servico_x_servico` |
| `dev_e_cabelo` | `dev_servico_x_cabelo` | `dev_servico_x_cabelo_x_cabelo` |
| `dev_p_processo` (etapa) | `dev_servico` | `dev_processo` |
| `dev_p_processo` | `dev_p_processo` (próxima etapa) | `dev_proxima_etapa` |
| `dev_g_departamento` | `dev_servico` | `dev_departamento` |
| `dev_g_departamento` | `dev_p_demanda` | `dev_departamento` |
| `dev_p_demanda` | `dev_config_dep_x_op` | `dev_op` |
| `dev_p_demanda` | `dev_log_op_x_departamento` | `dev_op` |
| `dev_p_demanda` | `dev_pausas_op` | `dev_op` |
| `dev_servico` | `dev_pausa` | `dev_servicoid` |
| `dev_motivos_pausas` | `dev_pausas_op` | `dev_motivo_selecao` |
| `dev_g_empresa` | `dev_p_demanda` | `dev_empresa` |
| `dev_lote` | `dev_e_cabelo` | `dev_lotedocabelo` |
| `dev_tipocabelo` | `dev_e_cabelo` | `dev_tipoid` |
| `dev_e_deposito` | `dev_e_cabelo` | `dev_deposito` |
| `dev_e_cabelo` | `dev_historico_cabelo` | `dev_cabelo` |
| `dev_e_cabelo` | `dev_transformacaocabelo` | `dev_cabeloorigem` (+ `dev_cabelodestino`) |
| `dev_e_cabelo` | `dev_movimentacoes_cabelo` | `dev_cabelo` |
| `dev_p_demanda` | `dev_movimentacoes_cabelo` | `dev_op` |
| `dev_lote` | `dev_conferencia_compra` | `dev_lote` |
| `dev_conferencia_compra` | `dev_conferencia` | `dev_conferencia_relativa` |
| `dev_balanco` | `dev_balanco_x_cabelo` | `dev_balanco` |
| `dev_e_cabelo` | `dev_balanco_x_cabelo` | `dev_cabelo` |

> **Total:** 30 entidades, ~50 relacionamentos de negócio 1:N (o "~80" da doc antiga incluía relações de sistema `regardingobjectid`, que não são de negócio).

---

## 6. 🎯 Métricas de produção que dá para calcular (o que interessa)

| # | Métrica / KPI | Onde está o dado |
|---|---|---|
| 1 | **Lead time da OP** (abertura → conclusão) | `dev_p_demanda.dev_inicio` (ou `createdon`) → data de finalização; roteiro em `dev_log_op_x_departamento`. **Horas úteis.** |
| 2 | **Tempo por departamento** (real) | `dev_config_dep_x_op.Tempo Corrido` (horas úteis). |
| 3 | **Aderência ao prazo (SLA)** | comparar saída real do setor (`dev_log_op_x_departamento.createdon` do próximo salto) vs previsão (`Tempo Restante` / previsão por departamento). |
| 4 | **Tempo de serviço (líquido)** | `dev_servico.dev_tempodeservicototal` = fim − início − pausas. |
| 5 | **% de tempo pausado** | `dev_tempodepausatotal / (dev_fim − dev_inicio)`. |
| 6 | **Tempo parado por motivo** | `dev_pausas_op` (`dev_inicio`/`dev_fim` + `dev_motivo_selecao`). |
| 7 | **Throughput** (serviços/OPs concluídos por período/depto) | contar `dev_servico` com `dev_servicofinalizado=1` por `dev_departamento`/`dev_processo` e `dev_fim`. |
| 8 | **Perda / rendimento de peso por serviço** | `dev_servico_x_cabelo`: `Σ qntd_in − Σ qntd_out`; rendimento = `qntd_out / qntd_in`. |
| 9 | **Variação de peso por operação** | `dev_historico_cabelo`: Quantidade vs Quantidade Final (cobre coleta, finalização, divisão, agrupamento, balanço). |
| 10 | **Perda de mesclagem** | Σ pesos de origem (`dev_transformacaocabelo` "União de Cabelos") − peso final do cabelo destino. |
| 11 | **Taxa de retrabalho / devolução** | `dev_servico_x_cabelo.dev_devolvido=true`; entrega parcial em `dev_servico`. |
| 12 | **Taxa de recusa na expedição** | recusados / total em `dev_conferencia_pedido_x_cabelo` (dispara retorno da OP ao estoque = retrabalho). |
| 13 | **Acurácia de inventário** | `dev_balanco_x_cabelo.Diferença` (contado − sistêmico) por depósito/mês. |
| 14 | **Perda no recebimento de MP** | `dev_conferencia` / `dev_conferencia_compra`: Perda Conferência/Separação (Kg e %), NT vs TM. |
| 15 | **Giro / saldo de estoque** | `dev_e_cabelo` (peso disponível por lote/tipo/tamanho/depósito) × movimentações. |
| 16 | **Composição da OP por tamanho** | `dev_servico_x_cabelo` → `dev_e_cabelo.dev_tamanho_considerado` (group by) × `dev_quantidade` (sum). |
| 17 | **Faturamento / margem por OP** (comercial × produção) | Postgres `pedido_venda.total` / `pedido_venda_item` × custo de compra (`pedido_compra_item`, `produto.preco_custo`), unindo por **número do pedido** ↔ `dev_p_demanda.dev_numero`. |
| 18 | **Comissão por vendedor** | Postgres `pedido_venda.vendedor_id`, `pedido_venda_item.comissao_*`. |

---

## 7. ⚠️ Convenções e pegadinhas (evitam erro de cálculo)

1. **Tamanho considerado:** o OptionSet codifica categoria nos milhares e o tamanho real nos 3 últimos dígitos → **tamanho real (cm) = valor % 1000** (ex.: `775730030` → 20 cm).
2. **Peso em Kg:** todos os campos de peso/quantidade estão em **Kg** (a doc antiga citava gramas em alguns pontos — desconsidere).
3. **Horas úteis:** prazos e tempos por departamento são calculados em **horário comercial: 08:00–18:00, seg–sex** (10h/dia), **sem feriados** (só fins de semana). **Não confunda com horas corridas.**
4. **Fuso:** timestamps operacionais gravados em **UTC**; converta para Brasília (−3) nos relatórios.
5. **Serviço padrão:** `dev_servico.dev_epadrao=true` ("Cabelos sem Serviço") é um **buffer**, não trabalho real — **filtre fora** ao medir produtividade.
6. **Exclusão lógica:** cabelos agrupados/mesclados/sumidos no balanço ficam **`statecode=1` (Inativo)** — decida se entram ou não em cada análise (para estoque atual, considere só ativos e `dev_disponivel`).
7. **Multi-empresa:** sempre segmente por `dev_g_empresa`.
8. **Contadores sequenciais** (nº de cabelo, nº de serviço, nº de OP de estoque) vêm de variáveis de ambiente e **não são 100% thread-safe** — raras duplicidades de ID podem ocorrer em alta concorrência.
9. **OptionSets sem rótulo no diagrama:** os valores/labels dos OptionSets **não estão** no `.drawio` — para obter os textos (ex.: rótulos de `dev_situacao`, `dev_tipo`, `Mês do Balanço`) é preciso consultar os **metadados do Dataverse** (via TDS/OData `StringMap` ou a própria UI).

### OptionSet `dev_situacao` (situação do serviço) — 7 estados
Usado nos dashboards de acompanhamento (valores `775730001`–`775730007`):
`NÃO INICIADO` · `EM ANDAMENTO` · `ATRASADO` · `PAUSADO` · `ENTREGA PARCIAL` · `FINALIZADO` · `ENTREGUE`

---

## 8. Onde cada dado NASCE (origem/confiabilidade)

Útil para saber a origem de cada número e sua confiabilidade:

| Dado | Nasce em (tela/componente) | Grava em |
|---|---|---|
| Criação da OP (tipo, empresa, nº pedido) | JS `dev_CriarOP.js` (wizard) | `dev_p_demanda` |
| Composição de cabelos na OP | JS `dev_OnloadDemanda.js` | `dev_servico_x_cabelo` |
| **Peso inicial** por cabelo | JS `dev_PesaInicio.js` + **balança local** | `dev_historico_cabelo`, `dev_servico_x_cabelo.dev_coletadoinicio` |
| **Peso final** por cabelo | JS `dev_finalizandoservicoNovo.js` + balança | `dev_historico_cabelo`, `dev_servico_x_cabelo.dev_coletadofim` |
| **Peso de saída** para OP | JS `dev_dar_saida_para_op.js` (bipagem) + balança | `dev_movimentacoes_cabelo` |
| Início/fim/pausa de serviço + tempos | **PCF PausaServico** (botões Iniciar/Pausar/Encerrar) | `dev_servico`, `dev_pausa` |
| Pausa de OP com **motivo** | JS `dev_pausar_despausar_op.js` | `dev_pausas_op` |
| Transferência OP entre departamentos | JS `dev_tranferir_op_departamento.js` | `dev_log_op_x_departamento`, `dev_config_dep_x_op` |
| Mesclagem (com validação de perda %) | JS `dev_MesclarCabelos.js` (senha de gestor) | `dev_e_cabelo`, `dev_transformacaocabelo`, `dev_lote_mesclado` |
| Contagem de balanço | **PCF BalancoPCF** (bipagem + balança) | `dev_balanco_x_cabelo` |
| Conferência de compra (NT/TM/perda) | fluxo de conferência | `dev_conferencia` / `dev_conferencia_compra` |
| Impressão de etiqueta de bandeja | JS `dev_imprimir_etiqueta.js` | `dev_bandeja` |

> **Hardware:** toda pesagem passa por uma **ponte HTTP de balança local** em `http://127.0.0.1:8080` (`/status` e `/peso`, com detecção de "peso estável"). É o único ponto de hardware do chão de fábrica — se pesos "não chegam", verifique esse serviço local (projeto `balancaPython`).

---

## 9. Integração Bling (contexto comercial)

- **Consulta em tempo real** (plugins `bling_api` / `CrmBling`): a OP tipo Pedido puxa o pedido de venda do Bling; itens marcados como **"Técnica"** indicam a operação técnica a executar na produção.
- **ETL** (`ETLBlingMagnusCabelo`): Bling → **PostgreSQL `bling`** → alimenta os `.pbix` comerciais. **Somente leitura do Bling.**
- **Sincronizador** (`SincronizadorBling`): Bling → Dataverse `contact` (só **contatos/clientes**; a parte de pedidos está como stub não implementado).
- **`dev_g_empresa`** guarda o token OAuth de cada empresa (cada empresa = 1 conexão Bling própria).

**Elos comercial ↔ produção:**
- Pedido de **venda** (`pedido_venda.numero`/`id`) ↔ OP `dev_p_demanda.dev_numero`.
- Pedido de **compra** (`pedido_compra.numero`) ↔ `dev_lote` (via código do fornecedor).

---

### Resumo em uma frase
> Para **produção**, o modelo gira em torno de **OP → serviço → (serviço×cabelo) → cabelo**, com **tempos** em `dev_servico`/`dev_config_dep_x_op`, **pausas** em `dev_pausa`/`dev_pausas_op`, **pesos/perdas** em `dev_servico_x_cabelo`/`dev_historico_cabelo`, e **estoque/balanço** em `dev_e_cabelo`/`dev_balanco_x_cabelo` — tudo no **Dataverse**; a camada **comercial** (vendas/compras/custos) está no **PostgreSQL `bling`**, cruzável por **número de pedido**.
