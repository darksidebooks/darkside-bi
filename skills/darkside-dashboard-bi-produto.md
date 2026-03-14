---
name: darkside-sellout
description: Consulta dados de sell-out (vendas B2B e D2C) da DarkSide Books direto do Metabase. Use esta skill SEMPRE que o usuário pedir sell-out, vendas por loja, performance de livraria, ranking de lojas, dados de rede varejista, vendas por canal físico, sell-through por ponto de venda, ou qualquer variação de "quanto vendeu em tal loja/rede". Ative também quando mencionar nomes de redes como Leitura, Saraiva, Amazon, Mercado Livre, Fnac, ou quando pedir comparativo entre lojas. Ative para dados D2C (site, app) quando o usuário mencionar "loja própria", "site darkside", "d2c", "varejo próprio". Esta skill sabe exatamente onde os dados estão no Metabase e como consultá-los com eficiência.
---

# DarkSide Sell-out — Guia de Consulta

## Banco de dados
- **database_id: 3** (Darkdata Presentation)
- Sempre usar este ID em todas as queries

---

## CANAL B2B — tabela `erp_exit_orders`

### Filtros obrigatórios para sell-out B2B limpo
```sql
WHERE tipo_operacao = 'S'           -- saídas (não devoluções)
  AND TRIM(tipo_venda) = 'ATACADO'  -- canal B2B (TRIM necessário — campo tem espaço)
```

> **Atenção:** `tipo_venda` tem espaço trailing ('VAREJO ') — sempre usar `TRIM()`.

### Campos principais
| Campo | Descrição |
|---|---|
| `isbn` | ISBN do produto |
| `descricao` | Nome do título |
| `data_nf` | Data da nota fiscal (usar para filtro de período) |
| `volume` | Quantidade vendida |
| `faturamento` | Receita realizada (após desconto) |
| `preco` | Preço unitário praticado |
| `preco_capa` | Preço de capa |
| `desconto` | Desconto aplicado |
| `cnpj` | CNPJ da loja compradora |
| `loja` | Nome da loja |
| `grupo_loja` / `grupo` | Grupo econômico |
| `estado` | UF |
| `cidade` | Cidade |
| `nota` | Número da NF (usar para COUNT de notas distintas) |
| `tipo_pedido` | Tipo: DRK_B2B_VENDA, DRK_B2B_INTG_AMAZON, DRK_B2B_CONSIGNADO, etc. |

### Tipos de operação B2B (tipo_pedido)
- `DRK_B2B_VENDA` — venda normal atacado
- `DRK_B2B_INTG_AMAZON` — integração Amazon
- `DRK_B2B_CONSIGNADO` — consignado
- `DRK_B2B_PREVENDA` — pré-venda
- `DRK_B2B_VTEX` — canal VTEX B2B
- `DRK_B2B_IMPORTACAO` — importação
- `NORMAL` — pedido normal sem classificação

### Query padrão B2B por título e período
```sql
SELECT
  TO_CHAR(data_nf, 'YYYY-MM')              AS mes,
  SUM(volume)                               AS unidades,
  ROUND(SUM(faturamento)::numeric, 2)       AS receita,
  ROUND(AVG(preco)::numeric, 2)             AS preco_medio,
  COUNT(DISTINCT nota)                      AS notas_fiscais
FROM erp_exit_orders
WHERE isbn = '{ISBN}'
  AND EXTRACT(YEAR FROM data_nf) = {ANO}
  AND tipo_operacao = 'S'
  AND TRIM(tipo_venda) = 'ATACADO'
GROUP BY 1
ORDER BY 1
```

### Query totalizadora B2B (auditoria)
```sql
SELECT
  SUM(volume)                               AS total_unidades,
  ROUND(SUM(faturamento)::numeric, 2)       AS total_receita,
  ROUND(AVG(preco)::numeric, 2)             AS preco_medio,
  COUNT(DISTINCT nota)                      AS total_notas,
  MIN(data_nf)                              AS primeira_nf,
  MAX(data_nf)                              AS ultima_nf
FROM erp_exit_orders
WHERE isbn = '{ISBN}'
  AND EXTRACT(YEAR FROM data_nf) = {ANO}
  AND tipo_operacao = 'S'
  AND TRIM(tipo_venda) = 'ATACADO'
```

### Query B2B por loja/grupo (sell-out por canal)
```sql
SELECT
  COALESCE(grupo_loja, grupo, 'Sem grupo')  AS grupo_economico,
  loja,
  cnpj,
  estado,
  SUM(volume)                               AS unidades,
  ROUND(SUM(faturamento)::numeric, 2)       AS receita
FROM erp_exit_orders
WHERE isbn = '{ISBN}'
  AND EXTRACT(YEAR FROM data_nf) = {ANO}
  AND tipo_operacao = 'S'
  AND TRIM(tipo_venda) = 'ATACADO'
GROUP BY 1, 2, 3, 4
ORDER BY 5 DESC
```

---

## CANAL D2C — tabela `order_items_d2c`

### Campos principais
| Campo | Descrição |
|---|---|
| `refid` | Referência do SKU (ex: `82042_1_0_002`) — **campo mais confiável para filtrar produto** |
| `item_id` | ID do SKU na VTEX |
| `ean` | EAN do produto |
| `order_id` | ID do pedido |
| `creation_date` | Data de criação do pedido |
| `status` | Status do pedido |
| `qty` | Quantidade |
| `unitprice` | Preço unitário em centavos |
| `totalprice` | Preço total em centavos (preço de tabela × qty) |
| `selling_price` | Preço praticado na vitrine em centavos — **usar este para receita** |
| `total_descontos` | Descontos aplicados em centavos (valor negativo) |
| `sales_channel` | Canal: 1=Site Desktop, 5=App Mobile, 2=Site Mobile |
| `sales_channel_name` | Nome do canal |
| `origin` | Origem: Marketplace, etc. |
| `utm_source`, `utm_medium`, `utm_campaign` | Dados de marketing |
| `coupon` | Cupom aplicado |

> **Atenção:** Todos os valores monetários estão em **centavos**. Dividir por 100.

### Filtro de status D2C — excluir cancelamentos
```sql
AND status NOT IN ('canceled', 'canceling', 'cancel', 'cancellation-requested', 'request-cancel')
```

### Como encontrar o refid de um produto
```sql
SELECT DISTINCT p.sku_id, p.product_name, p.ean, p.ref_id
FROM products p
WHERE LOWER(p.product_name) LIKE '%nome do produto%'
```

### Campo de receita D2C — qual usar
- **`selling_price × qty`** → receita líquida realizada (preço da vitrine após desconto de promoção). **Usar este para auditoria e relatórios.**
- **`totalprice`** → preço de tabela antes de promoções. Não usar como receita.
- **`total_descontos`** → desconto da vitrine (negativo). `totalprice + total_descontos = selling_price × qty`

### Query padrão D2C por título e período
```sql
SELECT
  TO_CHAR(creation_date, 'YYYY-MM')        AS mes,
  status,
  COUNT(DISTINCT order_id)                 AS pedidos,
  SUM(qty)                                 AS unidades,
  ROUND(SUM(selling_price * qty) / 100.0, 2)  AS receita_liquida,
  ROUND(SUM(totalprice) / 100.0, 2)           AS receita_bruta,
  ROUND(AVG(unitprice) / 100.0, 2)            AS preco_medio_unit
FROM order_items_d2c
WHERE EXTRACT(YEAR FROM creation_date) = {ANO}
  AND refid LIKE '{REFID_PREFIX}%'
  AND status NOT IN ('canceled','canceling','cancel','cancellation-requested','request-cancel')
GROUP BY 1, 2
ORDER BY 1, 2
```

### Query totalizadora D2C (auditoria)
```sql
SELECT
  status,
  SUM(qty)                                          AS unidades,
  ROUND(SUM(totalprice)      / 100.0, 2)            AS receita_bruta,
  ROUND(SUM(total_descontos) / 100.0, 2)            AS descontos,
  ROUND(SUM(selling_price * qty) / 100.0, 2)        AS receita_liquida
FROM order_items_d2c
WHERE EXTRACT(YEAR FROM creation_date) = {ANO}
  AND refid LIKE '{REFID_PREFIX}%'
  AND status NOT IN ('canceled','canceling','cancel','cancellation-requested','request-cancel')
GROUP BY 1
ORDER BY 2 DESC
```

---

## SELL-OUT POR LOJA (B2B) — tabela `vendas_detalhadas_book_info`

Usada para sell-out de uma **loja específica** (ex: relatório de reposição).

### Filtro por CNPJ de loja
```sql
WHERE REPLACE(REPLACE(REPLACE(cnpj_loja,'.',''),'-',''),'/','') = '{CNPJ_LIMPO}'
  AND data_venda >= CURRENT_DATE - INTERVAL '90 days'
```

> Sempre limpar o CNPJ removendo pontos, traços e barras antes de comparar.

### Query padrão sell-out por loja (90 dias)
```sql
SELECT
  sku                           AS isbn,
  MAX(descricao)                AS descricao,
  MAX(categoria)                AS categoria,
  SUM(volume)                   AS itens_90d,
  ROUND(AVG(preco)::numeric, 2) AS preco_medio
FROM vendas_detalhadas_book_info
WHERE REPLACE(REPLACE(REPLACE(cnpj_loja,'.',''),'-',''),'/','') = '{CNPJ_LIMPO}'
  AND data_venda >= CURRENT_DATE - INTERVAL '90 days'
GROUP BY sku
ORDER BY itens_90d DESC
```

### Encontrar CNPJ por nome de loja
```sql
SELECT DISTINCT loja, cnpj_loja, cidade_loja, uf_loja
FROM vendas_detalhadas_book_info
WHERE loja ILIKE '%nome da loja%'
LIMIT 10
```

---

## RESULTADO VALIDADO — Auditorias confirmadas (referência)

### Pedra Papel Tesoura (ISBN 9786555983579) — 2025

**B2B** (`erp_exit_orders`, `tipo_venda = 'ATACADO'`):
- Unidades: **25.142** ✅
- Receita: **R$ 806.196,72** (diferença de R$4 vs declarado R$806.192,72 — arredondamento)

**D2C** (`order_items_d2c`, `refid LIKE '82042%'`, campo `selling_price × qty`):
- Unidades: **1.061** ✅
- Receita: **R$ 38.766,71** ✅

---

## NOTAS IMPORTANTES

1. **Nunca usar `item_id` para JOIN com produtos** — o campo não é confiável para cruzamento. Preferir `refid` ou `ean`.
2. **Valores D2C sempre em centavos** — dividir por 100 em toda query.
3. **`tipo_venda` tem espaço trailing** — sempre usar `TRIM(tipo_venda)`.
4. **Para sell-out de loja física**, usar `vendas_detalhadas_book_info` (dados do Metabase B2B via sell-out de parceiros).
5. **Para saídas do ERP (faturamento próprio)**, usar `erp_exit_orders`.
6. **D2C canais**: `sales_channel = '1'` (Site Desktop), `'5'` (App Mobile), `'2'` (Site Mobile). Canal `VAREJO` no ERP = D2C.