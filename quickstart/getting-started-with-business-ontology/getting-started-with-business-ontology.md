author: Tianxia Jia, Chanin Nantasenamat
id: getting-started-with-business-ontology
categories: snowflake-site:taxonomy/solution-center/certification/quickstart, snowflake-site:taxonomy/product/ai, snowflake-site:taxonomy/product/platform
language: en
summary: Build a governed Business Ontology on top of well-built semantic views, connect it to a Cortex Agent, and measure what it adds with a fair A/B test.
environments: web
status: Published
feedback link: https://github.com/Snowflake-Labs/sfquickstarts/issues
fork repo link: https://github.com/sfc-gh-cnantasenamat/getting-started-with-business-ontology

# Getting Started with Business Ontology

## Overview

Ask a simple question like "How many suppliers do we have?" and you might expect a simple answer. In most companies, though, there are several correct-looking numbers. The vendor master holds 15 vendor records, but one of them is a probationary vendor that procurement policy says doesn't count, so the official answer is 14. Both numbers are computed correctly. They just answer slightly different questions.

People resolve this kind of ambiguity from experience: they know which number Finance reports, what "Tier A supplier" means at your company, and that you never cut a material that goes into something you sell. An AI agent doesn't have that experience. Given two plausible metrics, it may pick the wrong one, hedge with "15 or 14", or give different answers to different teams. Good semantic views help a lot, but each view describes one team's data. None of them is the place to decide which of two competing definitions is the official one.

Business Ontology, currently in Private Preview, is a governed layer of business meaning that sits above your semantic views. It doesn't replace them: the semantic views still compute every number. The ontology records what each business word means, which metric is canonical and which is only a look-alike, which policies apply, and how concepts connect across domains. A domain owner approves each definition, and the agent looks it up before it queries data.

With the ontology in place, the agent searches for "supplier count" first, finds the approved definition that says probationary vendors are excluded, queries the semantic view metric that implements it, and answers 14. It also names the definition it used, so anyone can check the answer. When the definition changes, a steward updates it once in the ontology, and every agent that consults it follows, with no edits to semantic views or prompts.

In this guide, you build exactly that on a synthetic SAP supply-chain dataset: three domain semantic views plus an ontology covering Procurement, Finance & Controlling, and Sales & Distribution. You build it in 10 steps. Each step runs one section of a companion script, shows the `$business-ontology` Cortex Code (CoCo) prompt that does the same thing, and ends with a check so you know the step worked. At the end, you compare two otherwise identical agents, one with the ontology and one without, to see what it adds.

A quick word on those prompts. Cortex Code (CoCo) is Snowflake's AI coding assistant, available as a desktop app or a command-line tool. It can load *skills*, which are packaged instructions that teach it how to do a specific job well. `business-ontology` is the skill for building and managing Business Ontology, and it comes bundled with Cortex Code, so there's nothing to install. Starting a message with `$business-ontology` tells Cortex Code to use that skill for the request. You then describe what you want in plain language, for example "Create three domains...", and the skill turns it into the right Business Ontology SQL calls. It shows you each draft for review before anything is written. Where a step includes a line marked **Underneath**, it shows the SQL that the skill runs for you.

### What You'll Learn

- What belongs in a semantic view and what belongs in Business Ontology
- How to model canonical metrics, their look-alikes, company vocabulary and policies
- How to connect ontology nodes to the semantic view metrics that implement them
- How to configure a Cortex Agent to search the ontology before it queries data
- How to measure what the ontology adds with a fair A/B test

Those are the skills. Here's the concrete result they produce.

### What You'll Build

By the end of this guide, your account holds the objects below: three domains of governed nodes, the relationships and representations that connect them to each other and to your data, and the semantic views and agents that put them to work. The counts are the same ones the **Check** lines confirm along the way.

| Object | Count | Notes |
|---|---|---|
| Domains | 3 | Procurement · Finance & Controlling · Sales & Distribution |
| Entities | 13 | Supplier, Material, Assembly, Bill of Materials, Purchase Order, Shipment, Carrier, Inspection Lot, AP Line Item, Cost Posting, Sales Order, Customer, Pricing Condition |
| Metrics | 16 | Canonical metrics, their look-alikes (e.g. *Vendor Record Count*, *Carrier Reported On-Time Rate*, *Gross Bookings*), and cross-domain metrics (*Product Gross Margin*, *Customer Profit*, *Revenue at Risk*) |
| Terms | 8 | Code fields and company vocabulary: Vendor Account Group, Material Group, Posting Key, GL Account, Cost Object, Condition Type, Sales Organization, **Tier A Supplier** |
| Policies | 3 | Probationary Vendor Exclusion, Credit Memo Sign Rule, BOM Input Protection |
| Relationships | 51 | MEASURES 13 · DERIVES 13 · CLASSIFIES 8 · RELATED_TO 6 · APPLIES_TO 4 · HAS_PART 3 · HAS_VARIANT 3 · IS_A 1 |
| Representations | 43 | IMPLEMENTS 12 · REPRESENTS 14 · RELATED_SEMANTIC_VIEW 8 · DESCRIBES 7 · REFERENCE_DATA 2 |
| Semantic views | 3 | `PROCUREMENT_ENRICHED_SV`, `FINANCE_ENRICHED_SV`, `SALES_ENRICHED_SV` |
| Agents | 7 | The A/B pair is `ENRICHED_AGENT` (SVs only) and `BON_AGENT` (same SVs + ontology search). The script also creates `BARE_AGENT` and two Finance/Sales team pairs used by the benchmark. |

All objects live in `BUSINESS_ONTOLOGY.SAP`.

> **Policies:** the demo stores policies as `itemKind: POLICY_CONCEPT`. The public docs currently document only `TERM`, `METRIC` and `ENTITY`, and the other kinds are not guaranteed to be stable in preview. If you need to stay on documented kinds, model a policy as a `TERM` named "... Policy" and link it with `APPLIES_TO`.

With the end result in mind, check that you have everything needed to build it.

### What You'll Need

Here's what you need before you start. Everything in the table is required except Python, which you only need for the optional full benchmark in Step 10.

| Requirement | Details |
|---|---|
| Account | Business Ontology enabled (Private Preview); search indexing enabled in your region |
| Snowsight | **Catalog > Business Ontology (Preview)**, and a SQL worksheet to run the scripts |
| Role | One role for the whole guide, used as your primary role. It needs `CREATE ONTOLOGY DOMAIN` on the account to create the three domains (`ACCOUNTADMIN` has it by default), plus the usual privileges to create a database, warehouse, semantic views and agents. The role that creates a domain owns it, so it can then draft, approve and link nodes, and search them. The demo used ACCOUNTADMIN. |
| Data | The [companion repository](https://github.com/sfc-gh-cnantasenamat/getting-started-with-business-ontology), which contains the synthetic SAP data, the semantic views, the ontology, the agents and the benchmark |
| CoCo | Cortex Code Desktop or CLI, for the prompts and the search check. The `$business-ontology` skill is bundled, so no install is needed. |
| Python (optional) | Python 3.10+ and `snowflake-connector-python`, only for the full benchmark in Step 10 |
| Time | About an hour of hands-on work, plus a wait of up to about 6 hours for search indexing (Step 8) |

**Snowsight:** **Catalog > Business Ontology (Preview) > Generate Ontology** creates a domain and lets you add terms interactively. Use CoCo for bulk work, importing from semantic views, and agent setup.

Before you build anything, it helps to be clear on what Business Ontology is and how it differs from the semantic views you may already have, so that's where we start.

## What Is Business Ontology

Business Ontology is the governed meaning layer in Horizon Catalog. It models shared business meaning as a connected graph of **terms, metrics and entities**, organised into **domains**, linked by **typed relationships**, and tied to the Snowflake objects that implement them through **representations**. Only approved items are visible to agents.

**Semantic Views tell Snowflake *how* to compute a metric. Business Ontology tells everyone *what* that metric means, *which* one is canonical, *who* approved it, and *whether* teams agree.**

The table below puts the two layers side by side, using the supplier count example from the introduction.

| Layer | Owns | Example from the demo |
|---|---|---|
| Semantic View | Tables, joins, dimensions, metric SQL; executes the number | `PROCUREMENT_ENRICHED_SV` has both `VENDOR_RECORD_COUNT` (15) and `ACTIVE_SUPPLIER_COUNT` (14) |
| Business Ontology | Which metric is canonical, business vocabulary, policies, cross-domain links | "Supplier Count" means active suppliers; *Probationary Vendor Exclusion* applies to it; it is implemented by `ACTIVE_SUPPLIER_COUNT` |

The picture below shows the same question asked twice. On the left, the agent has only the semantic views: it sees two defensible metrics, `VENDOR_RECORD_COUNT` and `ACTIVE_SUPPLIER_COUNT`, and either picks one or hedges. On the right, the ontology tells it which metric is official and which policy applies, so it answers 14 and cites the governed term.

![Semantic views alone compared with semantic views plus Business Ontology](assets/comparison.png)

The SV still computes the number. The ontology makes the agent pick the right one and say why.

That split between computing a number and deciding what it means is worth spelling out, because the rest of the guide builds on it.

### How Semantic Views and the Ontology Divide the Work

Most of the decisions in this guide follow from one division of labour:

- **The semantic view is the computation authority.** The SQL that produces a number lives in the view. A formula recorded in the ontology is documentation and a cross-check, not the calculation.
- **The ontology is the meaning authority.** When two correct metrics compete for the same business word, the ontology is where you say which one is official, and why.

That's also why a good ontology definition *decides* rather than just describes. "Number of suppliers" describes. "The official supplier count used in all reporting; probationary vendors are excluded; the vendor record count is an operational variant, not the official count" decides, and that sentence is what turns an agent's "it depends" into an answer.

With that rule in mind, here's how it plays out when you decide where a piece of business knowledge belongs.

### What Goes Where

As a rule of thumb, anything about *how* to compute a number belongs in the semantic view, and anything about *which* definition is official, or what a word means across teams, belongs in the ontology. The table gives concrete examples of each.

| Put it in the **Semantic View** | Put it in **Business Ontology** |
|---|---|
| Table joins, dimensions, facts | Which of several metrics is the canonical one |
| Metric SQL, including filters and sign logic (`ACTIVE_SUPPLIER_COUNT`, `COGS_RAW_MATERIALS` with credit memos subtracted) | Business vocabulary the data doesn't name ("Tier A supplier", "gross bookings is not revenue") |
| Synonyms that only help text-to-SQL | Policies that constrain how metrics are used (exclusions, sign rules, what must not be cut) |
| Verified queries | Cross-domain chains (Sales → BOM → Procurement cost) and who approved each definition |

**Don't** create one ontology metric for every SV metric. Create ontology nodes where meaning is contested, company-specific or cross-domain. The SV computes; the ontology decides and explains.

With that mental model in place, the next section explains how the 10 build steps are laid out, so you know what to expect from each one.

## How This Guide Works

The diagram below shows the 10 build steps in order. The blue steps (metrics, terms and policies, and the A/B test) are where most of the value comes from, and the amber step is the indexing wait. If you already have semantic views, the dashed bootstrap path can stand in for the early steps.

![End-to-end process](assets/steps.png)

Each of the 10 steps has the same parts, and all but the first are subsections under the step's heading:

- **What to run:** each step opens by telling you the one thing to do, which is usually to run a named section of a companion script in a Snowsight SQL worksheet. Running the scripts reproduces the demo exactly: the same 40 nodes, 51 relationships and 43 representations that the benchmark results depend on.
- **Use CoCo:** a subsection with an alternative to the script. A short sentence explains what the prompt in the box below it asks the `$business-ontology` skill to do, so you can paste the prompt into Cortex Code (CoCo) instead of running the script section. The prompts are shortened for reading. Use them to learn the workflow and to build an ontology on your own data. If you use them here, your ontology will differ from the demo and so may your results.
- **The SQL Behind It:** where it appears, a subsection showing the system function that both the script and the prompt call, with notes on how it behaves. It's for reference; you don't run it separately.
- **Check:** a subsection that says what to look at once the step has run, where to find it, the exact result you should see, and what to do if it doesn't match. When the check runs a query, an **Expected** subsection under it gives the result. Don't move on until it matches.

Pick one way per step. Running both the script and the prompt drafts every node twice.

> **Keep one worksheet session.** Steps 1 to 6 approve drafts with the session-scoped form of `SYSTEM$APPROVE_ALL_GLOSSARY_TERMS`, which approves only what the current session drafted. Run them all in the same worksheet. If you switch sessions, approve by term ID instead (Step 6 shows both forms).

If you already have semantic views of your own, read **Bootstrap from Semantic Views** under **Going Further** after you finish: the import covers Step 2, and Steps 3 to 5 fill the gaps it can't infer.

Now that you know the shape of each step, you set up the demo data and the semantic views that the ontology will sit on top of.

## 0. Demo Setup

Clone or download the [companion repository](https://github.com/sfc-gh-cnantasenamat/getting-started-with-business-ontology), then run the first six scripts in `sql/` in order. Everything lives in the `BUSINESS_ONTOLOGY` database and runs on the `BUSINESS_ONTOLOGY_WH` warehouse, split across two schemas: `RAW` holds the synthetic SAP source tables, and `SAP` holds everything you build on top of them. `03_sap_schema.sql` clones the `RAW` tables into `SAP` and stops with an error if they are missing.

If you'd rather work in a notebook, the repository also has the same SQL as two Snowflake notebooks, split at the indexing wait. `part_1_build_ontology.ipynb` covers this setup and Steps 1 to 7, and `part_2_agents_and_tests.ipynb` covers Steps 8 to 10. Each notebook section has the same name as the matching step here, with a short explanation above every cell, so you can follow either one. Upload them to a Snowflake Workspace and run the cells from top to bottom. For this setup, that's the seven cells under **0. Demo Setup** in Part 1, from `setup` to `check_setup`.

The list below shows the six setup scripts in the order you run them, with a comment after each one saying what it creates. Run each file in full in a Snowsight SQL worksheet before you open the next one.

```
sql/01_setup.sql              # BUSINESS_ONTOLOGY database, RAW schema, BUSINESS_ONTOLOGY_WH warehouse
sql/02_raw_data.sql           # 15 SAP tables + synthetic data in RAW
sql/03_sap_schema.sql         # SAP schema: tables cloned from RAW, T077K and T023T code tables
sql/04_ground_truth.sql       # BENCHMARK_QUESTIONS, BENCHMARK_GROUND_TRUTH (20 questions)
sql/05_bare_svs.sql           # CUSTOMERS_VIEW, BOM_EXPLOSION_VIEW, bare SVs (benchmark arm A only)
sql/06_enriched_svs.sql       # PROCUREMENT / FINANCE / SALES _ENRICHED_SV
```

With the data and semantic views in place, the remaining three scripts build everything else: the ontology, the links to your data, and the agents. Don't run them yet. Each one is a sequence of build steps rather than a single action, so the guide walks through them a section at a time. The list below shows the three scripts and what each one builds, so you can recognise them when the steps refer to them:

```
sql/07_ontology.sql           # 3 domains, 40 nodes, 51 relationships
sql/08_representations.sql    # 43 representations
sql/09_agents.sql             # 7 agents
```

The scripts and the build steps are numbered separately, so the table below shows which script each build step uses, and which notebook cells hold the same SQL. Steps 1 to 6 all work through `07_ontology.sql` in order, and two steps don't run a script at all.

| Build step | What you run | In the notebooks |
|---|---|---|
| 1. Create Domains | `07_ontology.sql`: the `Domains` section | Part 1, `domains` |
| 2. Add Entities | `07_ontology.sql`: the three node sections | Part 1, `nodes` |
| 3. Add Metrics | Nothing new; Step 2 already drafted them | Part 1, explanation only |
| 4. Terms and Policies | Nothing new; Step 2 drafted them, except *Tier A Supplier*, which comes in Step 5 | Part 1, explanation only |
| 5. Define Relationships | `07_ontology.sql`: an approve statement, the `Relationships` section, then the last section | Part 1, `approve_nodes`, `relationships`, `tier_a_supplier` |
| 6. Approve and Verify | A check query only | Part 1, `check_nodes`, `check_formula` |
| 7. Add Representations | All of `08_representations.sql` | Part 1, `representations`, `check_representations` |
| 8. Confirm Search Indexing | Two search commands, no script | Part 2, the commands to run in a terminal |
| 9. Build the Agent | All of `09_agents.sql` | Part 2, `agents`, `check_agent` |
| 10. Run A/B Tests | Questions to the agents in Snowsight, plus an optional Python benchmark | Part 2, `enriched_suppliers`, `enriched_revenue`, `bon_suppliers`, `bon_revenue` |

Before you build anything on top of the setup, it's worth confirming that it worked.

### Check

A semantic view (SV) describes a set of tables in business terms, with metrics and dimensions an agent can query. Scripts `05` and `06` each create three of them, so listing the semantic views in the `SAP` schema should show six. Run the statement below in a SQL worksheet.

```sql
SHOW SEMANTIC VIEWS IN SCHEMA BUSINESS_ONTOLOGY.SAP;
```

#### Expected

The statement returns six rows, one per semantic view. The three enriched views, `PROCUREMENT_ENRICHED_SV`, `FINANCE_ENRICHED_SV` and `SALES_ENRICHED_SV`, are the ones the rest of the guide uses. The three bare views, `PROCUREMENT_BARE_SV`, `FINANCE_BARE_SV` and `SALES_BARE_SV`, are used only by the optional benchmark. If any are missing, rerun the script that creates them, `05_bare_svs.sql` or `06_enriched_svs.sql`.

With the data and semantic views in place, you're ready to build the ontology itself, starting with the domains that hold everything else.

## 1. Create Domains

A domain is a container for one business area, such as Procurement. It groups the nodes that area owns, where a node is any entry in the ontology: an entity, a metric, a term or a policy. The demo uses three domains, one for each part of the business the questions touch.

To create the domains, open `sql/07_ontology.sql` and run the `Domains` section (the three `SYSTEM$CREATE_GLOSSARY_DOMAIN` statements). In the Part 1 notebook, that's the `domains` cell under **1. Create Domains**.

If you'd rather describe what you want in plain language than run SQL, you can build the same domains with Cortex Code.

### Use CoCo

To create the domains through the skill, paste the prompt below into CoCo. It asks the skill to create the three domains and write a description for each one that states its scope and its accountable business owner.

```
$business-ontology Create three domains: Procurement, Finance & Controlling, Sales & Distribution.
Describe each one's scope and its accountable business owner.
```

Whichever way you choose, the same SQL function does the work, and knowing it helps when you read the script or debug an error.

### The SQL Behind It

Both the script and the prompt call this function underneath. One call looks like `SYSTEM$CREATE_GLOSSARY_DOMAIN('Procurement')`, and you don't need to run it separately. Domains are created immediately and don't need approval. The API can't set owners and stewards yet, so record stewardship in the domain description. Domains can't be renamed yet, so choose names carefully.

With the domains created, confirm that all three are there before you add anything to them.

### Check

First, look at the result of each of the three statements: each should return without an error. Then, in Snowsight, open **Catalog > Business Ontology (Preview)** and look at the list of domains. You should see three new domains named exactly Procurement, Finance & Controlling and Sales & Distribution, each with no nodes yet. If a statement fails with an `Insufficient privileges` error, your primary role is missing `CREATE ONTOLOGY DOMAIN` (see **Troubleshooting**). If a domain is missing from the list, rerun its statement.

The domains are empty containers for now. Next, you fill them with entities, the business objects that everything else connects to.

## 2. Add Entities

Entities are the business objects that metrics measure, such as a supplier, a material or a customer. Each one is described in business language, so a reader who doesn't know the tables can still understand it, and names the key column that identifies it.

Back in `07_ontology.sql`, run the three node sections (`Procurement nodes`, `Finance & Controlling nodes`, `Sales & Distribution nodes`), stopping at the `Approve` comment. In the Part 1 notebook, the `nodes` cell under **2. Add Entities** runs the same statements. The script is organised by domain, so this one run drafts the entities, metrics, terms and policies for Steps 2 to 4 together: 39 drafts. A draft is a node that has been written but not yet approved, so agents can't use it yet. Steps 3 and 4 explain the metrics, terms and policies you just drafted and have nothing extra to run.

As in Step 1, you can draft the entities with a prompt instead of the script.

### Use CoCo

To draft the entities through the skill, paste the prompt below into CoCo. It lists each entity by domain, with a business description and, for *Supplier*, the key that identifies it and the vendors it excludes. The skill turns each line into a draft node.

```
$business-ontology Add these entities to Procurement:
- Supplier: an active vendor we purchase from, identified by vendor number (LFA1.LIFNR).
  Vendors in a probationary account group are vendor records but not suppliers.
- Material, Assembly, Bill of Materials, Purchase Order, Shipment, Carrier, Inspection Lot
Add AP Line Item and Cost Posting to Finance & Controlling.
Add Sales Order, Customer and Pricing Condition to Sales & Distribution.
```

Each entity becomes one call to the function below.

### The SQL Behind It

Both the script and the prompt call this function underneath. One call looks like `SYSTEM$DRAFT_GLOSSARY_TERM('{"name":"Supplier","domainName":"Procurement","itemKind":"ENTITY","description":"..."}')`, and you don't need to run it separately. The skill shows each draft before it writes it.

Because this one run drafts 39 nodes at once, check that every statement worked before you move on.

### Check

Look at the result of each statement as it runs. All 39 should succeed, and each one returns the ID of the draft it created. If any statement returns an error, fix the cause and rerun only that statement, because rerunning the whole section drafts the other nodes twice. The drafts don't appear in **Catalog > Business Ontology (Preview)**, in the approved term list or to agents yet. That's expected, and it changes when you approve them in Step 5.

Entities give the ontology its nouns. Next come the metrics that measure them, which is where most of the conflicts the ontology resolves actually live.

## 3. Add Metrics

A metric is a number the business reports, such as a supplier count or revenue. Many questions have more than one plausible answer: "how many suppliers" could mean all vendor records or only the active ones. So for each metric, the ontology records which number is canonical, meaning the official one, and **also adds the defensible-but-wrong look-alike** as its own node that says it is not the canonical one. That gives the agent both options to tell apart, instead of only the one it might stumble on.

There's nothing new to run here, because Step 2 already drafted all 16 metrics. Instead, read them in `07_ontology.sql` to see the pattern, for example *Supplier Count* next to *Vendor Record Count*, and *Revenue* next to *Gross Bookings*.

To see how the same metrics are described in plain language, look at the prompt below, which drafts them with the skill.

### Use CoCo

To draft the metrics through the skill, paste the prompt below into CoCo. It gives each canonical metric with its formula and, next to it, the look-alike metric that says it is not the canonical one, so the skill drafts both as separate nodes.

```
$business-ontology Add these metrics to Procurement:
- Supplier Count: number of active suppliers. Canonical answer to "how many suppliers".
  Excludes probationary vendors. Formula: COUNT(DISTINCT LIFNR) WHERE account group is not probationary.
- Vendor Record Count: all vendor master records, including probationary.
  NOT the supplier count; use only when the question is explicitly about vendor records.
- On-Time Delivery Rate: share of deliveries where actual goods-issue date <= planned date (LIKP).
  Canonical OTD.
- Carrier Reported On-Time Rate: carriers' self-reported OTD (LFA2.OTRAT).
  NOT our OTD; do not use as on-time delivery.
- Procurement Spend, BOM Cost Rollup
```

The prompt covers Procurement only, but every domain follows the same canonical and look-alike pattern, and a few rules apply to all metrics.

### Metric Notes

The other two domains contain these metrics:

- **Finance & Controlling:** COGS Raw Materials, COGS Freight, Cost Center Total, Product Gross Margin, Customer Profit, Break-even Discount.
- **Sales & Distribution:** Revenue (net of discounts) versus Gross Bookings (NOT revenue), Customer Discounts, Revenue at Risk.

Keep these rules in mind when you read or write a metric:

- `formula` is **descriptive**, not executable. The agent computes numbers with the SV metric and uses the formula to check it (see Step 9). Write it so a reviewer can see the rule: the numerator, the denominator, the filter and the grain.
- Only `METRIC` nodes carry `formula`. The formula is returned by `SYSTEM$GET_GLOSSARY_TERM`, not by graph or list reads.
- **Cross-domain metrics are the exception.** No single SV computes *Product Gross Margin* or *Revenue at Risk*. Their definitions must name each component and the bridge between domains (for example, "sold assembly → BOM components → material standard cost"), because the agent combines those components itself.

### Check

There's nothing to check yet, because the metrics are still drafts and can't be read back as approved nodes. Step 6 confirms them: it counts the approved nodes in each domain and reads back the *Supplier Count* formula, which should exclude probationary vendors.

Metrics settle which number is official. Terms and policies add the company vocabulary and the business rules that the data alone doesn't spell out.

## 4. Terms and Policies

Data often holds codes that mean nothing to a newcomer, such as `ZPRB` for a probationary vendor, and companies have their own vocabulary, such as "Tier A supplier". **Terms** decode those codes and capture that vocabulary. **Policies** are business rules the agent must apply, such as leaving probationary vendors out of supplier counts.

There's nothing new to run here either: Step 2 drafted 7 of the 8 terms and all 3 policies. The eighth term, *Tier A Supplier*, is added at the end of Step 5, because the script adds it in its own section after the relationships.

The prompts below show the same terms and policies in plain language.

### Use CoCo

To draft the terms and policies through the skill, paste the two prompts below into CoCo, one at a time. The first drafts the terms. Each line names a term, the code field it decodes, and the code values that business rules depend on.

```
$business-ontology Add these terms:
Procurement:
- Vendor Account Group: KTOKK on LFA1. ZSTR=Strategic, ZSTD=Standard, ZPRB=Probationary
  (flagged EXCLUDE_FROM_ACTIVE_COUNT in reference table T077K).
- Material Group: MATKL on MARA. Full mapping in T023T.
- Tier A Supplier: our internal name for strategic suppliers, meaning vendors in account group ZSTR.
  Synonyms: tier-A, strategic supplier.
Finance & Controlling: Posting Key (31 = invoice, 34 = credit memo), GL Account, Cost Object
Sales & Distribution: Condition Type (PR00 base, K007 customer discount, KF00 freight), Sales Organization
```

Once the terms are drafted, paste the second prompt, which drafts the three policies. Each line names the domain and the policy, followed by the rule the agent must apply.

```
$business-ontology Add these policies:
- Procurement / Probationary Vendor Exclusion: probationary vendors are excluded from supplier
  counts and supplier KPIs until promoted.
- Finance & Controlling / Credit Memo Sign Rule: credit memos (posting key 34) are subtracted
  from every cost and COGS total.
- Procurement / BOM Input Protection: a material used at any level of a sold assembly's BOM is a
  production input and must not be cut to reduce spend.
```

The *Vendor Account Group* term lists only three code values, even though real code fields can have hundreds. The tip below explains why.

### Scaling Code Fields

> **Scale code fields with one term per field, not one per value.** Inline only the values that business rules depend on. Keep the full code list in a reference table joined in the SV, and link the term to it with a `REFERENCE_DATA` representation. New code values arrive through ETL. The term changes only when the *rule* changes.

### Check

There's nothing to check yet, because the terms and policies are still drafts. Step 6 confirms that every node in all three domains, including these, is approved and counted.

Every node is now drafted. Next, you approve the drafts and connect them with typed relationships, so the ontology becomes a connected graph rather than a list.

## 5. Define Relationships

So far the nodes are a list. Relationships connect them into a graph, with an edge, or link, between two nodes. Each edge has a type that says how the nodes relate, for example that *Supplier Count* `MEASURES` *Supplier*. Use the most specific type that fits. An edge runs from a source node to a target node, except `IS_A`, which runs from the instance or subtype to the class. The table lists each type, when to use it, and an example from the demo.

| Type | Use for | Demo example |
|---|---|---|
| `MEASURES` | a metric quantifies an entity | Supplier Count MEASURES Supplier |
| `DERIVES` | the target is computed from the source | Revenue DERIVES Product Gross Margin |
| `CLASSIFIES` | a code or dimension categorises an entity | Vendor Account Group CLASSIFIES Supplier |
| `APPLIES_TO` | a policy constrains a concept | Probationary Vendor Exclusion APPLIES_TO Supplier Count |
| `HAS_PART` | a structural component | Bill of Materials HAS_PART Material |
| `HAS_VARIANT` | the same concept, scoped or narrowed | a look-alike or scoped variant of a canonical metric |
| `IS_A` | a subtype or instance | Tier A Supplier IS_A Supplier |
| `IDENTIFIED_BY`, `SCOPES`, `EQUIVALENT_TO` | keys, context boundaries, true synonyms with their own definition | Not used in the demo |
| `RELATED_TO` | only when nothing else fits | used sparingly |
| `CUSTOM` | only on explicit request, and needs a label | Not used in the demo |

In `07_ontology.sql`, work through the three actions below in this order. Relationship calls refer to nodes by name, and only approved nodes can be found by name, so the first action approves the drafts.

In the Part 1 notebook, these are the three cells under **5. Define Relationships**: `approve_nodes`, `relationships` and `tier_a_supplier`.

1. Approve the 39 drafts from Step 2. The statement below approves every draft that this worksheet session created, and nothing drafted in other sessions, so the nodes exist as approved terms before the relationships refer to them:

   ```sql
   SELECT SYSTEM$APPROVE_ALL_GLOSSARY_TERMS('[]', 'ACTIVE', '{"sessionId":"CURRENT_SESSION"}');
   ```

2. Run the `Relationships` section: 49 `SYSTEM$CREATE_GLOSSARY_RELATIONSHIP` statements.
3. Run the last section, `Added for benchmark Q19`, in three parts: the *Tier A Supplier* draft, then the approve statement above again, then its two relationships. That makes 40 nodes and 51 relationships.

If you're using CoCo instead, the prompt below defines the relationships for you.

### Use CoCo

To define the relationships through the skill, paste the prompt below into CoCo. It names a few relationships explicitly, using domain-qualified names and the relationship types from the table above, then asks the skill to propose the rest for you to review.

```
$business-ontology Define these relationships (use domain-qualified names):
- Procurement.Supplier Count MEASURES Procurement.Supplier
- Procurement.Probationary Vendor Exclusion APPLIES_TO Procurement.Supplier Count
- Procurement.Vendor Account Group CLASSIFIES Procurement.Tier A Supplier
- Procurement.Tier A Supplier IS_A Procurement.Supplier
- Sales & Distribution.Revenue DERIVES Finance & Controlling.Product Gross Margin
- Procurement.BOM Cost Rollup DERIVES Finance & Controlling.Product Gross Margin
Then find more relationships I missed in each domain.
```

Whichever way you created them, it helps to see what the relationships look like as a picture.

### The Resulting Graph

The graph below shows a slice of what you just created. Each box is a node, coloured by kind (entities white, metrics blue, terms purple, policies red), and each arrow is a typed relationship. Notice how *Revenue* in Sales & Distribution and *BOM Cost Rollup* in Procurement both feed *Product Gross Margin* in Finance & Controlling: those two `DERIVES` edges are the cross-domain bridge.

![A slice of the demo graph](assets/relationships.png)

Each arrow in that picture comes from one call to the function below.

### The SQL Behind It

Both the script and the prompt call this function underneath. One call looks like `SYSTEM$CREATE_GLOSSARY_RELATIONSHIP('Procurement.Tier A Supplier','Procurement.Supplier','IS_A')`, and you don't need to run it separately. That call creates the edge straight away. If you want someone to review an edge first, use `SYSTEM$DRAFT_GLOSSARY_RELATIONSHIP(...)` instead and approve it afterwards. Both calls accept domain-qualified names or term IDs.

The graph is useful to people, but the agent doesn't read it directly, which affects how you write your definitions.

### What the Agent Sees

> **Important:** agent search results **do not include relationships**. The agent follows links by reading definitions. Graph edges are for governance, lineage and the catalog. Any link the agent must follow, especially a cross-domain bridge, must **also** be stated in the description text.

With that in mind, confirm that the three actions worked.

### Check

Look at the results of the three actions. Both approve statements should succeed, and all 51 relationship statements (49 in the `Relationships` section and 2 for *Tier A Supplier*) should return without an error. A relationship statement that fails usually names a node that isn't approved yet, so rerun the approve statement and then rerun only the failed relationship. In **Catalog > Business Ontology (Preview)**, the nodes now appear in their domains, and opening *Supplier Count* shows its `MEASURES` link to *Supplier*.

With nodes and relationships in place, take a moment to confirm that everything is approved and counted correctly before you connect it to data.

## 6. Approve and Verify

Agents can only use approved nodes, so a draft that was missed is invisible to them. If you followed Step 5, everything is already approved, and this step confirms it before you go further.

Start by running the queries in the **Check** below, which are the `check_nodes` and `check_formula` cells under **6. Approve and Verify** in the Part 1 notebook. If any counts are short, approve the remaining drafts with one of the two forms below. The first approves only the drafts whose term IDs you list, which works from any session. The second approves every draft from the current session, the same statement you ran in Step 5:

```sql
-- Approve specific drafts, or everything from this session. Never call with no arguments:
-- an unscoped call promotes CREATE drafts account-wide.
SELECT SYSTEM$APPROVE_ALL_GLOSSARY_TERMS('["<termId1>","<termId2>"]');
SELECT SYSTEM$APPROVE_ALL_GLOSSARY_TERMS('[]', 'ACTIVE', '{"sessionId":"CURRENT_SESSION"}');
```

If you'd rather have CoCo find any leftover drafts for you, use the prompt below instead of the SQL above.

### Use CoCo

To find and approve any remaining drafts through the skill, paste the prompt below into CoCo. It lists the drafts this session created, so you can review them before the skill approves them.

```
$business-ontology Show all drafts I created this session and approve them.
```

Either way, the check below tells you whether anything is still missing.

### Check

Run the two queries below. The first counts the approved nodes in each domain, and the second reads back the formula of the *Supplier Count* metric.

```sql
SELECT 'Procurement' AS domain, ARRAY_SIZE(PARSE_JSON(SYSTEM$GET_GLOSSARY_TERM_LIST('Procurement')):terms) AS nodes
UNION ALL SELECT 'Finance & Controlling', ARRAY_SIZE(PARSE_JSON(SYSTEM$GET_GLOSSARY_TERM_LIST('Finance & Controlling')):terms)
UNION ALL SELECT 'Sales & Distribution', ARRAY_SIZE(PARSE_JSON(SYSTEM$GET_GLOSSARY_TERM_LIST('Sales & Distribution')):terms);

-- Formula is returned only by the per-term read (argument must be a constant)
SELECT PARSE_JSON(SYSTEM$GET_GLOSSARY_TERM('Procurement.Supplier Count')):formula::STRING;
```

#### Expected

The first query returns three rows, with 19 nodes for Procurement, 12 for Finance & Controlling and 9 for Sales & Distribution (40 in total). The second returns one row with the *Supplier Count* formula, which should count distinct vendor numbers and exclude vendors in the probationary account group. If a count is short, some drafts are still unapproved, so approve them with one of the forms above and run the check again. If the formula comes back empty, the node is still a draft.

If you also want to look at the relationships, `SYSTEM$GET_GLOSSARY_GRAPH('Procurement')` returns the approved graph (drafts are absent; use `SYSTEM$GET_GLOSSARY_DRAFT_GRAPH()` for those). The graph for one domain also includes edges to neighbouring domains, and large graphs can come back truncated, so its counts won't add up to exactly 51.

The ontology now says what things mean. Next, you tie each node to the Snowflake object that implements it, so the agent knows where to compute each answer.

## 7. Add Representations

The ontology now defines what each node means, but not where its data lives. A representation fills that gap: it records which Snowflake object, such as a table, a column or a semantic view metric, implements a node. This is not just catalog decoration. The skill's agent builder reads approved SV representations to write each SV tool's description, and that description is how the agent chooses which view to query.

There are five representation roles, and each one links a particular kind of node to a particular kind of Snowflake object. The diagram below shows each pairing.

![Representation roles](assets/representations.png)

The table lists the same roles, with how many of each the demo creates.

| Role | From → To | Demo count |
|---|---|---|
| `IMPLEMENTS` | metric → the exact SV metric (field level) | 12 |
| `RELATED_SEMANTIC_VIEW` | cross-domain metric → each SV that supplies a component | 8 |
| `REPRESENTS` | entity → table or view | 14 |
| `DESCRIBES` | term → column | 7 |
| `REFERENCE_DATA` | term → code table (T077K, T023T) | 2 |

Run all of `sql/08_representations.sql`, which contains 43 `SYSTEM$CREATE_GLOSSARY_ASSOCIATION` statements. In the Part 1 notebook, that's the `representations` cell under **7. Add Representations**, followed by the `check_representations` cell for the check below. This is the last section of Part 1.

As with the earlier steps, CoCo can add the representations from a prompt instead.

### Use CoCo

To add the representations through the skill, paste the prompt below into CoCo. It links one metric to its semantic view metric explicitly, then asks the skill to propose a representation for every other node and show them to you for review before it writes them.

```
$business-ontology In Procurement, associate the metric Supplier Count with metric
ACTIVE_SUPPLIER_COUNT (logical table VENDORS) in semantic view
BUSINESS_ONTOLOGY.SAP.PROCUREMENT_ENRICHED_SV as IMPLEMENTS.
Then propose representations for every other node in all three domains and show them for review.
```

Each of the 43 representations comes from one call to the same function.

### The SQL Behind It

The script and the skill both call `SYSTEM$CREATE_GLOSSARY_ASSOCIATION`. The example below, one of the 43 statements, links the *On-Time Delivery Rate* metric to the `OPERATIONAL_OTD_RATE` metric on the `SHIPMENTS` logical table in `PROCUREMENT_ENRICHED_SV`, with the role `IMPLEMENTS`. You don't need to run it separately, because it is already in the script:

```sql
SELECT SYSTEM$CREATE_GLOSSARY_ASSOCIATION('Procurement.On-Time Delivery Rate',
  '{"refType":"SEMANTIC_VIEW","fqn":"BUSINESS_ONTOLOGY.SAP.PROCUREMENT_ENRICHED_SV",
    "subObjectType":"METRIC","entityQualifier":"SHIPMENTS","subObjectName":"OPERATIONAL_OTD_RATE"}',
  'IMPLEMENTS');
```

One caution applies to representations after you create them:

> In preview, representations are **not revalidated** when a semantic view is rebuilt, renamed or dropped, and there is no automatic drift detection between a definition and its implementation. After any SV change, re-check with `$business-ontology Find drift between ontology and semantic views`.

Next, confirm that the representations were created and point at real objects.

### Check

Look at the result of each statement as it runs. All 43 should succeed (19 for Procurement nodes, 14 for Finance & Controlling and 10 for Sales & Distribution), and each result should contain `"error":false`. A result with `"error":true` usually means a node name or an object name is misspelled. Then read back the representations of two nodes, one metric and one entity:

```sql
SELECT 'Supplier Count' AS node, TO_JSON(PARSE_JSON(SYSTEM$GET_GLOSSARY_TERM('Procurement.Supplier Count')):representedAs) AS representations
UNION ALL SELECT 'Customer', TO_JSON(PARSE_JSON(SYSTEM$GET_GLOSSARY_TERM('Sales & Distribution.Customer')):representedAs);
```

#### Expected

The query returns two rows. In the *Supplier Count* row, look for the role `IMPLEMENTS` pointing at the metric `ACTIVE_SUPPLIER_COUNT` in `FINANCE_ENRICHED_SV`. In the *Customer* row, look for the role `REPRESENTS` pointing at `CUSTOMERS_VIEW`. Both should report `"validity":"VALID"`. If a row is empty, its statement didn't run, and if validity isn't `VALID`, the object it points at is missing, so rerun Step 0 and then the statement. Use the per-node read rather than `SYSTEM$GET_GLOSSARY_GRAPH` for this check, because the graph response can be truncated and may not list every representation.

The ontology is complete and bound to your semantic views. Before an agent can use it, though, it has to appear in search, which is what the next step checks.

## 8. Confirm Search Indexing

Agents don't read the ontology directly. They search an index, a searchable copy that Snowflake builds from your approved nodes in the background. **Indexing can take up to about 6 hours**, so a node you approved a moment ago may not be findable yet. Deleted terms can also keep appearing in results for a while after deletion.

To find out whether the index is ready, run the two searches below in a terminal, using the Cortex Code command-line tool (CLI). Each one searches the Business Ontology index, the same index the agent uses, for one of the terms the benchmark depends on:

```bash
cortex search object "supplier count" --types=business-ontology
cortex search object "tier a suppliers" --types=business-ontology
```

### Check

Read the results that each search prints. The first should list `Procurement.Supplier Count`, and the second `Procurement.Tier A Supplier`, each shown with its kind (for example, `METRIC` or `TERM`) and its definition. If a search returns no objects, the index isn't ready yet or can't see your terms. Confirm that the Step 6 counts still match, that the role you search with can read the domain, and that enough time has passed, then search again.

Because the agents depend on this index, the search result decides when you can move on.

> **Stop here until both searches pass.** Creating the agents in Step 9 doesn't need the index, so you can run it while you wait. The notebooks keep it in Part 2 so that everything after the wait starts from one place; either order works. Don't ask the agents questions or run the benchmark until the searches pass; the demo harness checks the same probes with `run_benchmark.py --check-index`.

Once search can find your terms, or while you wait for it, you create the agents that put the ontology to work.

## 9. Build the Agent

A Cortex Agent is an AI assistant in Snowflake that answers questions by calling tools, such as a semantic view it can query. In this step you create agents that also have a tool for searching the ontology, so they look up what a term means before they compute anything.

The diagram below shows what happens when a question comes in. The agent first searches the ontology for each business term and gets back the canonical metric, any policies and the company vocabulary. It then computes the number with the semantic view metric that the definition names, combines results across domains when the metric spans more than one, and answers with the governed term cited.

![How the agent uses the ontology](assets/agent.png)

Run all of `sql/09_agents.sql`, or the `agents` cell under **9. Build the Agent** in the Part 2 notebook. It creates 7 agents with the same model, budget and core instructions. Agents with Business Ontology (BON) search turned on have `BON` in their name, such as `BON_AGENT`. An A/B comparison asks two versions of something the same questions to see which does better. The two agents you compare are:

| Arm | Agent | Tools |
|---|---|---|
| B | `ENRICHED_AGENT` | 3 enriched SVs |
| C | `BON_AGENT` | the same 3 enriched SVs + `business_ontology` search |

The other five are used only by the full benchmark: `BARE_AGENT` (arm A, bare SVs), and the consistency pairs `FINANCE_ENRICHED_AGENT` / `FINANCE_BON_AGENT` and `SALES_ENRICHED_AGENT` / `SALES_BON_AGENT`.

If you're building an agent for your own data, CoCo can write the agent for you.

### Use CoCo

To build the agent through the skill, paste the first prompt below into CoCo. It asks the skill to write the agent spec, with one tool for each enriched semantic view and the ontology search turned on, and to check the three ontology settings before it creates the agent.

```
$business-ontology Create an agent that uses the ontology for Procurement, Finance & Controlling
and Sales & Distribution, with tools for PROCUREMENT_ENRICHED_SV, FINANCE_ENRICHED_SV and
SALES_ENRICHED_SV in BUSINESS_ONTOLOGY.SAP.
```

If you already have an agent, the second prompt below adds the ontology to it instead of creating a new one. The skill adds the search tool and the three settings, and keeps your existing tools and instructions:

```
$business-ontology Add the ontology to agent BUSINESS_ONTOLOGY.SAP.ENRICHED_AGENT
```

To understand what the script or the skill actually creates, it helps to read the agent's specification.

### The Agent Spec

Whichever way you build it, the spec below is the core of the demo's `BON_AGENT`, shortened for reading (the full spec is in `sql/09_agents.sql`). It's for reading, not running. It shows the ontology switch under `experimental`, the instructions that tell the agent to search the ontology before it queries data, and the four tools: one `business_ontology` search tool and one text-to-SQL tool for each enriched semantic view:

```yaml
models:
  orchestration: auto
experimental:
  EnableSnowscopeBusinessOntologySearch: true
orchestration:
  budget: {seconds: 150, tokens: 40000}
instructions:
  orchestration: |
    Before querying data, search business_ontology for each business term in the question,
    one concept per search. Use the governed definition to choose the canonical metric and to
    apply any policy. Compute numbers with the semantic view metric that implements the term;
    use the ontology formula only as a check. For cross-domain metrics, query each domain tool
    and combine the results as the definition describes.
  response: |
    State the answer first. When you used a governed definition, name the term (domain.node)
    and the semantic view metric that implements it.
tools:
  - tool_spec: {type: snowscope_search, name: business_ontology,
                description: "Governed business terms, metrics, entities and policies."}
  - tool_spec: {type: cortex_analyst_text_to_sql, name: query_procurement, description: "..."}
  - tool_spec: {type: cortex_analyst_text_to_sql, name: query_finance,    description: "..."}
  - tool_spec: {type: cortex_analyst_text_to_sql, name: query_sales,      description: "..."}
tool_resources:
  business_ontology: {corpus: businessOntology}
  query_procurement: {semantic_view: BUSINESS_ONTOLOGY.SAP.PROCUREMENT_ENRICHED_SV, ...}
  # query_finance, query_sales likewise
```

Three lines in that spec are what connect the agent to the ontology, and they need extra care. **`CREATE AGENT` does not validate these three settings.** A typo is accepted, and the agent then silently never searches the ontology:

- `experimental.EnableSnowscopeBusinessOntologySearch: true`
- `tool_spec.type: snowscope_search`
- `tool_resources.business_ontology.corpus: businessOntology` (the resource key must equal the tool `name`)

Note the different spellings: the CLI search type is `business-ontology`, while the agent corpus is `businessOntology`.

That's why the check for this step reads the settings back from the agent you created.

### Check

Because `CREATE AGENT` accepts typos in the three settings above, confirm that they were saved correctly by describing `BON_AGENT`.

```sql
DESCRIBE AGENT BUSINESS_ONTOLOGY.SAP.BON_AGENT;
```

#### Expected

The statement returns one row. In its `agent_spec` column, look for three things: `EnableSnowscopeBusinessOntologySearch` set to `true`, a tool named `business_ontology` of type `snowscope_search`, and `"corpus":"businessOntology"` under that tool's resources. If any of them is missing or spelled differently, the agent will never search the ontology, so correct the spec in `09_agents.sql` and rerun it. If you used CoCo, you can ask it to do this check for you: `$business-ontology Verify the ontology settings on agent BON_AGENT`.

Everything is built. The last step shows the difference the ontology makes, by asking two otherwise identical agents the same questions.

## 10. Run A/B Tests

In this step you compare two agents that are identical except for the ontology tool: `ENRICHED_AGENT` (B) and `BON_AGENT` (C). Because everything else is the same, any difference in their answers comes from the ontology. Start with a quick hands-on comparison, then run the full benchmark if you want numbers.

### Try It

In Snowsight, open **AI & ML > Agents**, and ask `ENRICHED_AGENT` and then `BON_AGENT` the same two questions from the benchmark. In the Part 2 notebook, the four cells under **10. Run A/B Tests** ask the same questions for you, one agent and question per cell. For each question, the table gives the answer the ontology makes official and the look-alike answer an agent without it may give.

| Question | Correct answer | Defensible-but-wrong answer |
|---|---|---|
| How many suppliers do we have? | 14 (active suppliers) | 15 (all vendor records, including probationary) |
| What is our revenue? | 711,915 (net of K007 discounts) | 746,000 (gross bookings) |

Once you have both agents' answers, compare them with the table.

#### Check

Compare the two agents' answers to each question against the table. `BON_AGENT` should give the correct answer, 14 suppliers and 711,915 in revenue, and name the governed term and metric it used, for example `Procurement.Supplier Count` implemented by `ACTIVE_SUPPLIER_COUNT`. `ENRICHED_AGENT` may give the right number on some runs, but it can also pick the look-alike or hedge ("15 or 14"), and it has no governed term to cite. Because agents vary from run to run, ask each question two or three times.

Two questions show the difference. To measure it properly, across many questions and repeated runs, the demo includes a benchmark.

### Run the Benchmark

This part is optional and needs the Python requirements in **What You'll Need**. The benchmark is a Python script that asks the agents 20 questions, grouped into four tiers, from controls that the semantic views alone can answer to questions that need cross-domain reasoning or governed definitions, and scores each answer against the correct one, called the ground truth, stored in `sql/04_ground_truth.sql`.

From the repository root, run the commands below. The first installs the Snowflake Python connector the script needs. The second checks that the search index is ready, using the same probes as Step 8, and stops if it isn't. The third runs the full benchmark against arms B and C and prints the scorecard. Replace `<conn>` with the name of your Snowflake CLI connection:

```bash
pip install snowflake-connector-python

# 1. Confirm the index probes pass
SNOWFLAKE_CONNECTION_NAME=<conn> python -u benchmark/run_benchmark.py --check-index

# 2. Full run: 20 questions, arms B and C, 3 runs (5 for tiers 2 and 3), plus consistency pairs
SNOWFLAKE_CONNECTION_NAME=<conn> python -u benchmark/run_benchmark.py --arms B,C --runs 3 --runs-hard 5
```

A full run takes a while, because each question is asked several times. When it finishes, read the scorecard as described below.

#### Check

When the run finishes, look at the scorecard it prints, with accuracy per arm and per tier. Compare arm B (`ENRICHED_AGENT`) with arm C (`BON_AGENT`) tier by tier. The control tier should be tied, because the semantic views alone can answer those questions. Any gains should cluster in a small number of contested questions, the ones with a look-alike answer. That pattern tells you the ontology is working where it should. In the demo's final run, the enriched semantic views alone scored 71/80 (89%) and the same views plus Business Ontology scored 80/80 (100%), with both tied on the control tier. The misses clustered in a few contested questions, so read the result as a gain in consistency and traceability, not as a general accuracy figure. The companion repository's README has the full scorecard.

The benchmark measures this demo. When you build an ontology for your own data, you'll want an evaluation of your own, designed the same way.

### Design Your Own

When you evaluate an ontology on your own data, follow the same rules the demo used:

- **Questions:** write 5 to 10 per domain where meaning is contested (look-alike metrics, company vocabulary, policies, cross-domain chains). Add **controls** where the SV alone should succeed, and at least one question about a concept that doesn't exist, which the agent should decline to answer.
- **Runs:** run each question several times, because agents vary from run to run. The demo used 3 runs, and 5 for the hard tiers.
- **Scoring:** a hedged answer ("15 or 14") counts as wrong. Track accuracy, right on every run, hedging, and whether the answer cites a governed term.
- **Starting point:** adapt `benchmark/run_benchmark.py` and the ground-truth tables.

To get a first set of questions, paste the prompt below into CoCo. It asks the skill to draft 10 evaluation questions from your approved Procurement nodes, in the same mix of question types as the demo, each with its expected answer, its look-alike answer and the node that governs it:

```
$business-ontology Using the approved nodes in Procurement, draft 10 evaluation questions:
4 where two plausible metrics exist, 2 using company vocabulary, 2 that need a policy,
1 control a semantic view alone should answer, and 1 about a term that doesn't exist.
For each, give the expected answer, the wrong-but-defensible answer, and the governing node.
```

That completes the core build: an approved ontology, bound to semantic views, searched by an agent and measured against a baseline. The section that follows is optional.

## Going Further

The first part shows how to keep the ontology current as definitions change. The second shows how to start a new ontology from semantic views you already have.

### Update the Ontology

Once the ontology is live, most changes take a single prompt. The table lists common maintenance tasks and the prompt for each.

| Task | Prompt |
|---|---|
| Edit a definition or formula | `$business-ontology Update Procurement.Supplier Count: also exclude temporary vendors (ZTMP)` |
| Add a metric | `$business-ontology Add a metric Net Price Realization to Sales & Distribution. Formula: ... Description: ...` |
| Find missing edges | `$business-ontology Find more relationships in the Finance & Controlling domain` |
| Rename a node | `$business-ontology Rename Procurement.Inspection Lot to Quality Inspection` |
| Delete a domain | `$business-ontology Delete the <domain> domain` |

Edits to **ACTIVE** nodes go through an **edit card** (`SYSTEM$DRAFT_GLOSSARY_TERM_UPDATE`). The live definition stays in force until the edit is approved, and agents see the change only after the index refreshes. The skill's quality gate shows the old and new values side by side.

Updating assumes the ontology already exists, as yours does now. If you're starting a new ontology on top of semantic views you already have, the skill can draft much of it for you.

### Bootstrap from Semantic Views

To start that way, paste the prompt below into CoCo, with your own database and schema in place of `BUSINESS_ONTOLOGY.SAP`. It asks the skill to read the semantic views in that schema and propose a draft ontology from them for you to review:

```
$business-ontology Bootstrap ontology from our semantic views in BUSINESS_ONTOLOGY.SAP
```

The skill scans the views (`SHOW`/`DESCRIBE`, metrics, comments, verified queries, join graph) and proposes:

- entities derived from the join graph, not one per table;
- metric and term candidates, with duplicates across views merged;
- a domain for each candidate, with its provenance shown;
- `IMPLEMENTS` representations back to the source metrics.

The diagram below shows that flow from left to right. The skill scans your views and proposes candidates, a steward reviews them, and the items the import can't infer are added by hand before everything becomes the approved ontology.

![Bootstrapping from semantic views](assets/bootstrap.png)

Nothing is written until you approve it at review. What the import **can't** infer is exactly where the value lies:

- which of two look-alike metrics is canonical;
- company vocabulary such as "Tier A";
- policies;
- cross-domain bridges.

Add those by hand using Steps 3 to 5.

Semantic views aren't the only starting point. Depending on where your definitions live today, you can start from any of these sources:

| You have | Prompt |
|---|---|
| CSV, spreadsheet or document on a stage | `$business-ontology Import ontology terms from @my_stage/glossary.csv` |
| dbt project | `$business-ontology Extract ontology from our dbt manifest at @my_stage/manifest.json` |
| Table and column comments | `$business-ontology Extract terms from our tables in <db.schema>` |
| Cortex Sense context | `$business-ontology Promote our <context> context to the ontology` |
| Knowledge in people's heads | `$business-ontology I need an ontology for procurement. We count suppliers excluding probationary vendors...` |
| OWL / RDF / Turtle | Use the `business-ontology-owl-dev` skill |

If any step didn't match its expected check, the table below maps common symptoms to their likely causes and fixes.

## Troubleshooting

Each row starts from what you see, then gives the most likely cause and the fix.

| Symptom | Likely cause | Fix |
|---|---|---|
| `Unknown function SYSTEM$CREATE_GLOSSARY_DOMAIN` | Feature not enabled | Contact your account team |
| `Insufficient privileges ... must have CREATE ONTOLOGY DOMAIN` | Your primary role can't create top-level domains | `USE ROLE` a role that has it, or have an admin run `GRANT CREATE ONTOLOGY DOMAIN ON ACCOUNT TO ROLE <role>`. Secondary roles don't count |
| A relationship statement fails | A node it names is still a draft | Approve the drafts (Step 5, action 1), then rerun the statement |
| Node counts in Step 6 are short | Drafts made in another session | Approve by term ID, or ask CoCo to show and approve your drafts |
| Agent never calls `business_ontology` | Flag, type or corpus misspelled (accepted silently) | `DESCRIBE AGENT` and check all three settings |
| Search returns nothing for new terms | Terms still in draft, not yet indexed, or the searching role can't read the domain | Approve them, wait up to about 6 hours, and re-probe with a role that has `READ` (or higher) on the domain |
| Deleted or renamed terms still returned | Stale index | Wait for re-index; don't benchmark until probes are clean |
| Agent still picks the look-alike metric | No look-alike node, or the definition doesn't say "NOT ..." | Add the look-alike as its own metric that says it isn't canonical; add the policy |
| Agent can't follow a cross-domain link | The link exists only as a graph edge | Also state the bridge in the definition text, since search doesn't return edges |
| Agent picks the wrong SV tool | Tool description doesn't name the concepts | Add `IMPLEMENTS` representations and regenerate the spec, or edit the tool descriptions |
| `GET_GLOSSARY_TERM` error inside a query | The argument must be a constant | Pass a literal name or ID |
| Representation points at a rebuilt view | No revalidation in preview | Re-run the drift check after SV changes |
| `Invalid identifier` inside JSON | Unescaped single quote | Write `'` as `''` inside SQL string literals |

For day-to-day work after this guide, the quick reference below collects the calls you used in one place.

## API Quick Reference

The table lists each operation and the system function behind it.

| Operation | Call |
|---|---|
| Create domain | `SYSTEM$CREATE_GLOSSARY_DOMAIN('<name>')` |
| Draft node | `SYSTEM$DRAFT_GLOSSARY_TERM('{"name":..,"domainName":..,"itemKind":"ENTITY\|METRIC\|TERM","description":..,"formula":..,"synonyms":[..]}')` |
| Approve nodes | `SYSTEM$APPROVE_ALL_GLOSSARY_TERMS('["<id>",...]')` or `SYSTEM$APPROVE_GLOSSARY_TERM('<id or name>')` |
| Edit an active node | `SYSTEM$DRAFT_GLOSSARY_TERM_UPDATE(...)`, then approve the edit card |
| Relationship | `SYSTEM$CREATE_GLOSSARY_RELATIONSHIP('<src>','<tgt>','<TYPE>')` or `DRAFT_` + `APPROVE_` |
| Representation | `SYSTEM$CREATE_GLOSSARY_ASSOCIATION('<node>','<assetRefJson>','<ROLE>')` |
| Read | `SYSTEM$GET_GLOSSARY_TERM('<id or domain.name>')`, `SYSTEM$GET_GLOSSARY_TERM_LIST('<domain>')`, `SYSTEM$GET_GLOSSARY_GRAPH('<domain>')`, `SYSTEM$GET_GLOSSARY_DRAFT_GRAPH()` |
| Delete domain | `SYSTEM$DELETE_GLOSSARY_DOMAIN('<domain>')` |

Gotchas:
- The field is `itemKind`, not `termType`.
- Only `METRIC` nodes take `formula`.
- Relationship calls use positional arguments.
- Never call approve-all without arguments.

When you're finished exploring, clean up the demo objects. Agents search every domain the caller can see, so leftover demo terms become noise for other agents in your account.

## Clean Up

Delete the three demo domains first. A domain can't be deleted while it still contains terms, so the skill runs a cascade (representations, then relationships, then terms, then the domain) and asks for confirmation before deleting. Paste the prompt below into CoCo to delete the Procurement domain and everything in it:

```
$business-ontology Delete the Procurement domain and all its contents
```

Repeat for Finance & Controlling and Sales & Distribution. Then run the two statements below in a SQL worksheet. They drop the `BUSINESS_ONTOLOGY` database, with its `RAW` and `SAP` schemas, the semantic views and the agents, and the `BUSINESS_ONTOLOGY_WH` warehouse that the setup scripts created:

```sql
DROP DATABASE IF EXISTS BUSINESS_ONTOLOGY;
DROP WAREHOUSE IF EXISTS BUSINESS_ONTOLOGY_WH;
```

With the demo objects removed, here's a recap of what you built and where to go next.

## Conclusion And Resources

Congratulations! You've built a governed Business Ontology on top of three well-built semantic views, connected it to a Cortex Agent, and learned how to measure what it adds with a fair A/B test. The semantic views compute the numbers; the ontology decides which number is canonical and explains why.

### What You Learned

- What belongs in a semantic view and what belongs in Business Ontology
- How to model canonical metrics alongside their defensible-but-wrong look-alikes
- How terms and policies capture company vocabulary and business rules
- How representations link ontology nodes to the semantic view metrics that implement them
- How to configure a Cortex Agent with `snowscope_search` over the `businessOntology` corpus
- How to evaluate ontology uplift with controls, repeat runs and hedge-aware scoring

To go deeper on any of these, the resources below are a good place to start.

### Related Resources

- [Business Ontology](https://docs.snowflake.com/en/LIMITEDACCESS/snowflake-horizon/business-ontology): overview and concepts
- [Access control](https://docs.snowflake.com/en/LIMITEDACCESS/snowflake-horizon/business-ontology-access-control): domain-level privileges
- [Agent integration](https://docs.snowflake.com/en/LIMITEDACCESS/snowflake-horizon/business-ontology-agents): `snowscope_search` with the `businessOntology` corpus
- [Horizon Context](https://docs.snowflake.com/en/user-guide/snowflake-horizon/horizon-context): how Business Ontology fits with semantic views and Cortex Sense
- [Companion repository](https://github.com/sfc-gh-cnantasenamat/getting-started-with-business-ontology): build order, benchmark harness, web app and final results
