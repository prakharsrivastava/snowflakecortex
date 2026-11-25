# Snowflake aur Databricks: SOP + Calibration AI Architecture

Yeh guide aapke portals mein dekhe gaye objects ko alag-alag samjhata hai: kaunsa data/table kis kaam ka hai, semantic layer aur document search ek saath kyun use hote hain, UI mein kahan jaana hai, aur SQL se kaise inspect/create karna hai.

> **Scope aur verified state:** Snowflake portal mein `CORTEX_LAB.SOP_AI` schema, chhe tables, do stages, ek semantic view, ek Cortex Search service aur `TEST` agent dikhte hain. Databricks workspace mein SOP/calibration tables nahi mile; `demo.demo` ke paanch sales tables mile. Wahan `Demo Data Insights Agent` naam ka Genie Agent paanch sales tables ke saath create hua. Dono cloud environments abhi ek doosre se automatically connected nahi hain.

## 1. Ek nazar mein

```text
Snowflake
  Calibration records -> Semantic View -> Cortex Analyst -----+
  SOP documents -> chunks -> Cortex Search -------------------+-> TEST Agent -> answer

Databricks (current workspace)
  demo.demo sales tables -> Genie Agent -> natural-language SQL answer
  SOP/calibration dataset: abhi Databricks mein verify nahi hua
```

In dono stacks ke concepts milte-julte hain, lekin unke objects, configuration, permissions aur data indexes alag hain. Snowflake ka semantic view ya Cortex Search service Databricks mein seedhe reuse nahi hota.

## 2. Snowflake architecture — jo portal mein bana hua hai

```text
                            +--> CALIBRATION_RECORDS
                            |          |
Sources / loading (details  |          v
abhi verify nahi) ----------+   CALIBRATION_SEMANTIC_VIEW
                            |          |
                            |          v
                            |   Cortex Analyst: calibration_analyst
                            |
                            +--> SOP source / parsed content / chunks
                                       |
                                       v
                                  SOP_CHUNKS
                                       |
                                       v
                             SOP_SEARCH_SERVICE
                                       |
                                       v
                             Cortex Search: sop_search
                                       |
                    +------------------+-----------------+
                    |                                    |
                    +---------- TEST Agent --------------+
                                       |
                                  User answer
```

Dashed/source se aane wale hissa conceptual hai: schema mein stages aur raw/parsed/metadata tables dikhte hain, lekin exact file-to-table ingestion jobs, stage paths, schedules, ya refresh pipeline ko is inspection mein verify nahi kiya gaya. Isliye is guide ko un jobs ke deployed hone ka proof na samjhein.

### Snowflake object map

| Object | Portal mein dekha gaya role |
|---|---|
| `CORTEX_LAB.SOP_AI.CALIBRATION_RECORDS` | Calibration data; semantic view ka source table |
| `SOP_EXTRACTION_RAW_L2`, `SOP_METADATA_L2`, `SOP_PARSED_DOCUMENTS`, `SOP_DOCUMENTS` | Naam se SOP extraction/metadata/parsed/document layers lagte hain; exact columns aur upstream links ko table details mein verify karein |
| `SOP_CHUNKS` | Cortex Search service ka actual source; selected columns `CHUNK_TEXT`, `FILE_NAME`, `CHUNK_ID`, `CHUNK_INDEX` |
| `CALIBRATION_SEMANTIC_VIEW` | `CALIBRATION_RECORDS` par business model |
| `SOP_SEARCH_SERVICE` | SOP text ke liye Cortex Search index/service |
| Agent `TEST` | Tools: `calibration_analyst`, `sop_search`, aur `code_execution` |

### Semantic view aur Cortex Analyst

Semantic view database mein business meaning define karta hai. Aapke `CALIBRATION_SEMANTIC_VIEW` mein ek logical table `CALIBRATION` hai, jo `CALIBRATION_RECORDS` ko map karta hai. Portal ke SQL definition mein:

- **Primary key:** `CALIBRATION_ID`
- **Fact:** `COST_AMOUNT` ko `CALIBRATION_COST` naam se expose kiya gaya
- **Dimensions:** plant, equipment, status, calibration date
- **Metrics:** total records, overdue records, completed records, total cost, overdue percentage

Fact ko dimension ke saath confuse na karein: fact aam taur par measurable value hai; dimension us value ko group/filter karne ka context hai. Metrics business-approved aggregation/formula hain. Fact sirf cost hona zaroori nahi—source mein meaningful, correctly aggregatable values hon to aur facts/metrics add kiye ja sakte hain.

Cortex Analyst structured tables par natural-language sawaal ko SQL analysis mein badalta hai. Semantic view usse batata hai ki `OVERDUE`, `PLANT`, `CALIBRATION_COST` aur metrics ka matlab kya hai. Analyst ko expose kiye gaye model/privileges ke bahar ke tables ka business logic apne aap nahi pata hota.

### Cortex Search

Portal se verify hua configuration:

- Source: `CORTEX_LAB.SOP_AI.SOP_CHUNKS`
- Search text: `CHUNK_TEXT`
- Attributes: `FILE_NAME`, `CHUNK_ID`; `CHUNK_INDEX` source selection mein dikh raha hai, isliye use return/filter attribute banana hai ya nahi woh service configuration mein confirm karein
- Warehouse: `COMPUTE_WH`
- Target lag: `1 day`
- Embedding model: `snowflake-arctic-embed-m-v1.5`
- Serving aur indexing: `ACTIVE`
- Preview: 3 sample chunks; preview mein training sample SOP PDF dikh raha tha, production-approved SOP nahi

Cortex Search chunk text mein relevant passages nikalta hai. Yeh calibration table par group-by/total nikalne ka replacement nahi. Snowflake service embedding/indexing ko managed tareeke se chalati hai; user query ko service API/SDK se bheja ja sakta hai.

### Agent `TEST` ka role

Agent ke configured tools:

- `calibration_analyst`: structured questions aur SQL analytics
- `sop_search`: SOP passages ki retrieval
- `code_execution`: custom code tool; use ki zaroorat aur allowed scope check karein

Udaharan: “Kitne calibrations overdue hain aur SOP ke hisaab se un par kya action lena hai?” Agent Analyst se count/records aur Search se relevant procedure maang sakta hai. Yeh tools ki configuration se mumkin hai; mixed question par final answer ki accuracy ko alag se test karna chahiye.

### Snowflake UI mein kahan jaana hai?

1. **Data → Databases → `CORTEX_LAB` → `SOP_AI`**: schema, tables, stages, semantic view, agent aur search service dekhein.
2. **Tables → table name → Columns / Data Preview / Details**: columns, types, sample rows aur table metadata verify karein.
3. **`CALIBRATION_SEMANTIC_VIEW` → semantic model**: logical table, facts, dimensions, metrics; **View SQL** se definition inspect karein.
4. **Cortex Search services → `SOP_SEARCH_SERVICE` → View Details**: Search Service, Data Preview, Costs tabs; service status aur source/query columns inspect karein.
5. **Agents → `TEST` → Tools**: Analyst/Search/custom tools aur unke referenced database/schema dekhein; View SQL se agent definition inspect karein.
6. **Projects / Worksheets → SQL Worksheet**: read-only verification ya DDL run karein. Pehle role aur active warehouse check karein.

### Snowflake SQL: inspect, phir create

Read-only inspection ke liye worksheet mein:

```sql
USE DATABASE CORTEX_LAB;
USE SCHEMA SOP_AI;

SHOW TABLES IN SCHEMA CORTEX_LAB.SOP_AI;
DESC TABLE CORTEX_LAB.SOP_AI.CALIBRATION_RECORDS;
DESC TABLE CORTEX_LAB.SOP_AI.SOP_CHUNKS;

SELECT COUNT(*) AS ROW_COUNT
FROM CORTEX_LAB.SOP_AI.SOP_CHUNKS;

SELECT CHUNK_ID, FILE_NAME, CHUNK_INDEX, LEFT(CHUNK_TEXT, 300) AS TEXT_SAMPLE
FROM CORTEX_LAB.SOP_AI.SOP_CHUNKS
LIMIT 5;

SHOW CORTEX SEARCH SERVICES IN SCHEMA CORTEX_LAB.SOP_AI;
```

Semantic view ki portal mein dekhi gayi definition ka saar yeh hai. Yeh maujooda semantic view ko **replace** kar sakta hai, isliye ise bina review/test ke production mein na chalayein:

```sql
CREATE OR REPLACE SEMANTIC VIEW CORTEX_LAB.SOP_AI.CALIBRATION_SEMANTIC_VIEW
  TABLES (
    CALIBRATION AS CORTEX_LAB.SOP_AI.CALIBRATION_RECORDS
      PRIMARY KEY (CALIBRATION_ID)
      COMMENT = 'One row represents one calibration record'
  )
  FACTS (
    CALIBRATION.COST_AMOUNT AS CALIBRATION_COST
      COMMENT = 'Calibration cost in Indian rupees'
  )
  DIMENSIONS (
    CALIBRATION.PLANT_NAME AS PLANT
      WITH SYNONYMS = ('facility', 'site')
      COMMENT = 'Manufacturing plant',
    CALIBRATION.EQUIPMENT_TYPE AS EQUIPMENT_NAME
      COMMENT = 'Name of equipment',
    CALIBRATION.CALIBRATION_STATUS AS STATUS
      COMMENT = 'Stored status: COMPLETED, OVERDUE, or DUE_SOON',
    CALIBRATION.CALIBRATION_DAY AS CALIBRATION_DATE
      COMMENT = 'Last recorded calibration date'
  )
  METRICS (
    CALIBRATION.TOTAL_RECORDS AS COUNT(*)
      COMMENT = 'Total number of calibration records',
    CALIBRATION.OVERDUE_RECORDS AS COUNT_IF(STATUS = 'OVERDUE')
      COMMENT = 'Number of overdue calibration records',
    CALIBRATION.COMPLETED_RECORDS AS COUNT_IF(STATUS = 'COMPLETED')
      COMMENT = 'Number of completed calibration records',
    CALIBRATION.TOTAL_CALIBRATION_COST AS SUM(CALIBRATION_COST)
      COMMENT = 'Total calibration cost in INR',
    CALIBRATION.OVERDUE_PERCENTAGE AS
      100.0 * COUNT_IF(STATUS = 'OVERDUE') / NULLIF(COUNT(*), 0)
      COMMENT = 'Percentage of calibration records marked overdue'
  )
  COMMENT = 'Semantic model for pharmaceutical calibration analytics';
```

Service create karne ka template (portal par dekhe gaye base SELECT/config se). Pehle exact identifiers, role grants, table data, and existing service state validate karein. `CREATE` use kiya hai—existing service ho to duplicate error expected hai; use replace karne ka decision alag se lein.

```sql
CREATE CORTEX SEARCH SERVICE CORTEX_LAB.SOP_AI.SOP_SEARCH_SERVICE
  ON CHUNK_TEXT
  ATTRIBUTES FILE_NAME, CHUNK_ID
  WAREHOUSE = COMPUTE_WH
  TARGET_LAG = '1 day'
  EMBEDDING_MODEL = 'snowflake-arctic-embed-m-v1.5'
AS (
  SELECT CHUNK_TEXT, FILE_NAME, CHUNK_ID, CHUNK_INDEX
  FROM CORTEX_LAB.SOP_AI.SOP_CHUNKS
);
```

Agent ki exact DDL ko schema mein `TEST → View SQL` se copy/inspect karein; agent SQL syntax aur tool reference platform version ke saath badal sakte hain. `CREATE OR REPLACE` ko bina backup/review ke na chalayein.

## 3. Databricks architecture — current workspace

```text
Unity Catalog: demo.demo
  customers -----------+
  orders --------------+
  order_items ---------+--> Genie Agent: Demo Data Insights Agent
  products ------------+       |
  shipping_zones ------+       +--> question -> generated SQL -> SQL Warehouse -> answer
```

Yeh **sales demo** architecture hai, Snowflake calibration/SOP architecture nahi. Portal par `demo.demo` ke paanch tables aur unhi paanch tables se configured `Demo Data Insights Agent` verify hua. `SOP_AI` jaisa Databricks catalog/schema ya SOP chunks wahan inspect nahi hue. Agent ke Resources section mein paanch tables listed the.

### Databricks UI mein kahan jaana hai?

1. **Analytics → Genie Agents**: existing Genie agents dekhein; is workspace mein yahin se new agent flow khola gaya.
2. **New → Select tables manually → All catalogs → `demo` → `demo`**.
3. `customers`, `orders`, `order_items`, `products`, `shipping_zones` select karke **Create manually**.
4. Naya agent portal ne `Demo Data Insights Agent` naam se banaya.
5. Agent ke **Resources** section mein connected tables; **Instructions** aur **Examples** mein business rules aur verified question/SQL pairs add karein.
6. Agent chat par test questions poochhein aur generated SQL/results review karein.
7. **Catalog → `demo` → `demo`** se table columns/sample rows inspect karein. **SQL Editor** mein verification SQL chalayein; available SQL Warehouse select karein.

> **Naam ka note:** is workspace ke UI mein “Genie Agents” aur “What should your Genie agent do?” labels the. Isliye is account mein jo object abhi banaya gaya hai woh Genie Agent hai. Databricks ke docs/UI releases mein “Genie space” ya “Genie Agent” naming/product surface badal sakti hai; exact product/feature availability workspace edition aur rollout par depend karti hai. Classic Genie space ko is newly-created agent ke saath bina version check kiye identical na maanein.

### Databricks SQL: read-only verification

Databricks **SQL Editor** mein SQL Warehouse select karke:

```sql
SHOW TABLES IN demo.demo;
DESCRIBE TABLE demo.demo.customers;
DESCRIBE TABLE demo.demo.orders;
DESCRIBE TABLE demo.demo.order_items;
DESCRIBE TABLE demo.demo.products;
DESCRIBE TABLE demo.demo.shipping_zones;

SELECT COUNT(*) AS row_count FROM demo.demo.orders;
SELECT * FROM demo.demo.orders LIMIT 5;
```

Available fields/relations pehle `DESCRIBE TABLE` aur table details se dekhein. Neeche ke query examples ko exact schema dekhne ke baad hi run karein; column names assume na karein:

```sql
-- Agar orders mein order date aur order ID columns available hon:
-- SELECT DATE_TRUNC('MONTH', <order_date>), COUNT(DISTINCT <order_id>)
-- FROM demo.demo.orders GROUP BY 1 ORDER BY 1;
```

Genie Agent create/configure karne ke liye is workspace UI flow ka use kiya gaya. SQL Editor se Genie UI object banane ki zaroorat nahi hai; SQL ka kaam source data inspect/query karna hai. Agent instructions, example questions, resource selection aur permissions UI mein configure karein.

## 4. Dono cloud ke equivalent concepts — aur farq

| Kaam / concept | Snowflake mein aapka object | Databricks mein milta-julta object | Kya same nahi hai |
|---|---|---|---|
| Governed table/catalog | `CORTEX_LAB.SOP_AI` tables | Unity Catalog `catalog.schema.table` | DDL, storage, grants aur lineage alag |
| Business meaning / structured analytics | `CALIBRATION_SEMANTIC_VIEW` + Cortex Analyst | Genie Agent instructions/examples + Unity Catalog metadata; supported configuration mein metric views/semantic definitions | Semantic view ka exact 1:1 clone nahi; metric view bhi khud conversational agent nahi |
| Natural-language SQL | Cortex Analyst | Genie / Genie Agent | Different runtime, configuration, query behavior |
| Document/vector retrieval | Cortex Search service | Databricks AI Search (formerly Mosaic AI Vector Search) index + embedding pipeline; agent tool se connect kiya ja sakta hai | Search service/index auto-share ya auto-migrate nahi hota |
| Tools ko orchestrate karna | Snowflake Cortex Agent (`TEST`) | Genie Agent / Databricks agent experiences | Tool model, APIs, permissions aur supported capabilities alag |
| Compute for SQL | Snowflake warehouse (`COMPUTE_WH`) | Databricks SQL Warehouse | Billing/scale/suspension settings alag |

**Business-semantic mapping:** Snowflake semantic view mein logical tables, keys, dimensions, facts aur metrics explicitly define hain. Databricks mein Genie Agent ke table resources, table/column comments, instructions, SQL examples aur—jab suitable/available ho—Unity Catalog metric views se similar business context ban sakta hai. Ek simple instruction paragraph metric view ka replacement nahi; metric view bhi Genie se sawaal poochhne ka UI nahi.

**Vector-search mapping:** Cortex Search ek managed search service hai. Databricks mein raw document chunks ko embed karke Databricks AI Search (formerly Mosaic AI Vector Search) index/endpoints par index karna hota hai aur us retrieval capability ko agent flow se wire karna hota hai. Databricks Genie Agent se paanch sales tables connect karne se SOP document search nahi banta.

## 5. Dono architectures ko SOP + Calibration ke liye kaise banayein

### Option A — Snowflake mein end-to-end (current best fit)

1. SOP source files ke liye current stage/loading route identify karein; verify karein ki kaunsa job `SOP_EXTRACTION_RAW_L2`, parsed tables aur `SOP_CHUNKS` update karta hai.
2. Stable document ID, filename, version/effective date, chunk ID/order aur source reference maintain karein. Reprocessing par duplicate/stale chunks ko handle karein.
3. `SOP_CHUNKS` ke `CHUNK_TEXT` aur metadata ko Cortex Search service mein index karein; target lag aur warehouse ko freshness/cost need ke mutabik set karein.
4. `CALIBRATION_RECORDS` columns/status definitions ko data owner se verify karke semantic view metrics aur synonyms document karein.
5. Cortex Analyst ko semantic view expose karein; test SQL aur role permissions validate karein.
6. Agent `TEST` mein Search/Analyst tools ko minimum required privileges ke saath wire karein. Code execution tabhi enable karein jab use-case aur policy clear ho.
7. Agent par document-only, metric-only, mixed, ambiguous aur no-result questions test karein; citations/source filename aur generated SQL inspect karein.

### Option B — Databricks mein SOP + Calibration (data migration ke baad)

1. Source files/tables ko approved Databricks ingestion path se laakar Unity Catalog ke governed catalog/schema mein stage karein. Snowflake data ko yahan abhi present maan kar na chalein.
2. Calibration data ke column comments, primary identifiers, status definitions, trusted metrics aur date/currency semantics document karein. Genie Agent resources mein sirf approved tables add karein; metric views available hon to reusable metrics ko wahan model karein.
3. SOP documents parse/chunk karein, stable IDs aur metadata rakhkar embeddings/index banaayein. Databricks AI Search ke index refresh/sync, permissions aur freshness test karein.
4. Search retrieval ko Databricks Agent / supported tool interface se connect karein. Genie Agent se structured analytics aur vector retrieval ko ek hi natural-language orchestration mein support milta hai ya nahi, workspace feature/version ke mutabik validate karein; agar nahi, dedicated agent workflow banayein.
5. SQL Warehouse, serving endpoint/index, access controls aur budget/cost monitoring set karein.
6. Mixed questions test karein: “overdue count + relevant procedure + source citation.” Metric query aur retrieved SOP dono separately verify karein.

### Option C — Dono platforms saath rakhna

- Ek system ko source of truth declare karein; same table ko dono taraf independently edit na karein.
- Cross-cloud copy/replication pipeline aur refresh ownership explicitly banayein. Aapke current portals mein Snowflake se Databricks tak automatic data sharing configured hone ka evidence nahi mila.
- PII, SOP versioning, deletion propagation, role mapping, audit logs, freshness SLAs, warehouse/index cost aur failure alerts document karein.
- Dono systems apne semantic definitions/indexes alag maintain karenge. “Same concept” ka matlab “same object” ya “same answer guarantees” nahi.
- Agar use-case sirf SOP + calibration hai, to pehle ek primary platform par end-to-end implementation test karein; dual-stack tabhi rakhein jab governance, latency, team ownership ya integration requirement justify kare.

## 6. Rules, quality aur security checklist

- **Business definitions:** “overdue” ka source status, due-date logic, timezone aur denominator metric owner approve kare. Example questions + expected SQL/result maintain karein.
- **Searchable text:** chunk mein enough context rakhein; filename, document ID, version, section/page jaise metadata preserve karein. Chunk overlap/size ko test se tune karein.
- **Source validation:** portal preview ke teen rows poori corpus coverage ka proof nahi. SOP ka training-sample banner actual production approval ka substitute nahi.
- **Permissions:** least-privilege roles/grants, table/agent/service access aur user-level filters validate karein. Admin role se successful test normal user permissions prove nahi karta.
- **Answer verification:** generated SQL, filters, joins, totals aur retrieved citations dekhein. AI output ko operational procedure ke taur par bina approved SOP reference ke follow na karein.
- **Freshness:** Cortex Search target lag `1 day` configured dikh raha tha; Databricks pipeline/index refresh policy abhi set/verified nahi. Required SLA ke against dono ko align karein.
- **Cost:** Snowflake warehouse + Cortex Search indexing/query usage; Databricks SQL Warehouse + vector index/model/serving usage. Auto-suspend, workload and budgets monitor karein; trial/free tier zero-cost guarantee nahi.
- **Secrets:** passwords, tokens, private keys ya credential-bearing URLs notebook/docs mein store na karein. Secret scopes / managed identity / platform-approved auth use karein.

## 7. Aapke current setup ki verification table

| Platform | Verified | Abhi verify nahi hua |
|---|---|---|
| Snowflake | `CORTEX_LAB.SOP_AI`; 6 tables, 2 stages; semantic view ki 1 logical table, 1 fact, 4 dimensions, 5 metrics; Search service ACTIVE/indexing ACTIVE; `SOP_CHUNKS` source; `TEST` ke 3 tools | SOP ingestion job/stage mapping/schedule; har table ka schema/data quality; agent mixed-query accuracy; production status/approval of preview SOP |
| Databricks | `demo.demo` ke 5 sales tables; `Demo Data Insights Agent` un tables ke saath; agent UI mein Resources/Instructions/Examples sections | Snowflake SOP/calibration data ka Databricks copy; Genie Agent ka tested answer quality; vector index/document search; metric view availability/config; production permissions/cost profile |

## 8. Official references

- [Snowflake Cortex Search overview](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-search/cortex-search-overview)
- [Snowflake Cortex Analyst](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-analyst)
- [Snowflake semantic views overview](https://docs.snowflake.com/en/user-guide/views-semantic/overview)
- [Snowflake Cortex Agents](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents)
- [Databricks Genie overview](https://docs.databricks.com/aws/en/genie/)
- [Databricks Unity Catalog metric views](https://docs.databricks.com/aws/en/uc-semantics/metric-views)
- [Databricks AI Search (formerly Vector Search)](https://docs.databricks.com/aws/en/ai-search/ai-search)
- Databricks workspace mein **Help / Documentation** se apne rollout ke Genie Agent, Genie space, metric view aur AI Search pages kholein; feature labels aur availability account ke edition/rollout ke saath badal sakte hain.
