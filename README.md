# SDIP + gemini-3.8-flash — BIRD Mini-Dev Results (PostgreSQL)

## System Details

- **System:** Spatial Data Intelligent Platform (SDIP, 空间数据智能平台)
- **Organization:** Beijing Freedo Technology Co., Ltd. (北京飞渡科技股份有限公司, Freedo)
- **Model:** gemini-3.8-flash, temperature 0, thinking disabled
- **Method:** Ontology-grounded text-to-SQL. The question is answered through the platform's semantic layer (one object type per table, bound attributes, declared join keys). A read-only SQL gateway then validates and normalises the generated SQL deterministically before it runs.

## Results

- **Split:** Mini-Dev (500 tasks, PostgreSQL)
- **Execution Accuracy (EX): 69.00%**, from the official `evaluation_ex.py` against the current Hugging Face revision of the gold set.
- **Oracle Knowledge:** Yes. BIRD's evidence hint is included in the input.

| Difficulty | EX | Tasks |
|---|---|---|
| Simple | 83.11% | 148 |
| Moderate | 66.80% | 250 |
| Challenging | 53.92% | 102 |
| **Total** | **69.00%** | 500 |

### Which gold set

There are two versions of the PostgreSQL gold set.

- **Hugging Face, current revision.** `birdsql/bird_mini_dev` @ `f65faf4ae3b638c1fa6df1d3370c8d92c8366301` (2026-01-18), `data/mini_dev_pg-00000-of-00001.json`. Scored against it, the submission gets **69.00**, recorded in `results/eval_ex_hf_current.txt`.
- **The older `minidev.zip` package.** Scored against it, the same predictions get **68.80** (83.11 / 66.40 / 53.92), recorded in `results/eval_ex_minidev_zip.txt`.

The two versions differ in three ways:

1. The zip package lists questions 137 and 138 twice and omits 119 and 120. From position 487 onward its order differs from the Hugging Face version.
2. The gold SQL for question 1322 differs.
3. The gold SQL for question 879 differs (a `CAST` was added).

We answered 119 and 120 separately (1 of the 2 correct) and ordered the prediction file by the Hugging Face positions.

## Method

**Per question.** The question is sent to the platform's agent in the form `BIRD Mini-Dev <db_id>: <question> Hint: <evidence>`. The agent's first step is deterministic: it calls the semantic-layer answer tool with the original wording. Inside that tool:

1. **Concept selection (1 LLM call).** The model sees the database's concept list (object types) and picks up to 4 candidate concepts. The platform then adds directly joinable tables from the declared relations, up to 8 tables in total.
2. **Concept description (no LLM).** Each candidate concept comes with:
   - its definition;
   - its attributes, each with data type, name and BIRD's column description;
   - the complete value list for low-cardinality columns (40 distinct values or fewer), read live from the database;
   - its declared join keys.
3. **SQL writing (1 to 3 LLM calls).** The model writes PostgreSQL that refers to concepts and attributes. The query goes through the SQL gateway. A new round happens only on a gateway rejection (the reasons go back to the model), when a number from the question is missing from the SQL, or when a filtered query returns 0 rows. There are at most 3 rounds and a 600 s time budget.

So each question takes 2 to 4 LLM calls. There is one candidate per question: no voting, no self-consistency, no fine-tuning. A few-shot retriever over BIRD train exists in the code but was **switched off** for these runs.

**SQL gateway (deterministic, no LLM).**

- *Statement checks.* It accepts exactly one SELECT. It rejects SELECT INTO, FOR UPDATE, LATERAL, table functions, NATURAL/USING joins and schema-qualified functions.
- *Schema checks.*
  - Tables must be known concepts.
  - Columns must be bound attributes; ambiguous or wrongly prefixed columns are rejected, with the valid attributes listed.
  - A column equality between two concept tables must use a declared join key.
  - Functions must be on an allow-list.
- *Rewrites.*
  - `SELECT *` is expanded to the bound columns.
  - Literals in `=` and `IN` are normalised to the stored spelling of known values.
  - `ORDER BY` without an explicit null order gets `NULLS LAST`.
  - A top-level division column that is not inside `ROUND` is cast to `double precision`.
  - A row limit is enforced: `LIMIT 5001`, and a truncated result counts as wrong.
- *Execution.* It checks the schema fingerprint, then runs the query on a read-only connection with a 90 s statement timeout. If the result is empty or all-zero and the query has an unverified literal, the platform looks that literal up case-insensitively in the database. If exactly one value matches, it substitutes it and runs the query again, once.

**Ontology.** The ontology is generated from the database schema: primary keys from PostgreSQL, foreign keys from PostgreSQL plus the original SQLite `foreign_key_list`, and column descriptions from BIRD's `database_description` CSVs. No LLM was used to build it. A handful of foreign keys that the databases do not declare were added by hand (see disclosure 3).

**How much of the score comes from the semantic layer.** We have not measured this on Mini-Dev. We ran an ablation on 200 BIRD dev questions: same model, same evidence, same scoring, with the semantic layer removed so that the model writes SQL from the raw `CREATE TABLE` statements and sample rows.

| | EX on 200 dev questions | Execution errors |
|---|---|---|
| Through the semantic layer | 71.0% | 0 |
| Model writes SQL directly | 69.0% | 11 |

The difference is not significant (p = 0.48). Our reading is that BIRD's evidence hints already provide most of the business knowledge a semantic layer would add. Most of the score should be attributed to the model and to the gateway's checks, not to the ontology.

**Submission SQL.** The platform runs on a view layer with snake_case names. Each predicted SQL was rewritten back to the official table and column names. Every rewritten query was re-executed on the official database to confirm it returns the same rows as the original.

## What we used during development (please read)

This is a **validated** number, not a held-out one.

1. **Mini-Dev overlaps with BIRD dev, which we tuned on.** All 500 Mini-Dev questions come from BIRD dev. We split BIRD dev into a frozen development part of 995 questions, which we tuned on and whose misses we analysed, and a holdout.
   - 331 of the 498 distinct Mini-Dev questions fall in the development part.
   - The other 167 fall in the holdout. They were never run before our first Mini-Dev run (v1).
2. **Some rules came from looking at specific dev misses, and several of those questions are in Mini-Dev.**
   - The literal-normalisation rule came from analysing dev question 765, which is in Mini-Dev.
   - Several schema-specific hints in the SQL-writing prompt were written while looking at dev misses.
3. **Some foreign keys were added by reading dev gold SQL.** A few foreign keys that the databases do not declare (in `debit_card_specializing` and `card_games`) were added by following joins in the dev gold SQL. Mini-Dev question 462 is one of the questions cited.
4. **The ratio-to-float rule came from Mini-Dev misses.** That rule was added after we read the misses of our first Mini-Dev run. That run scored 63.80, and part of the gap was ratio columns returned as `numeric`, which the official script compares unequal to `float`. The test set was run once more in full after the change.

| Run | Date | Gold | Official EX |
|---|---|---|---|
| v1 | 2026-10-16 | minidev.zip | 63.80 |
| v2 (this submission) | 2026-10-18 | minidev.zip | 68.80 |
| v2 (this submission) | 2026-10-18 | Hugging Face current | **69.00** |

## Files

- `results/predict_mini_dev_PostgreSQL.json` — predictions in BIRD format, keyed by the Hugging Face positions 0–499.
- `results/eval_ex_hf_current.txt` — official `evaluation_ex.py` output against the Hugging Face gold.
- `results/eval_ex_minidev_zip.txt` — official output against the `minidev.zip` gold (that file's own ordering).

## Reproducing the score

The scoring command:

```bash
python3 -u evaluation_ex.py --db_root_path ../postgresql/ \
  --predicted_sql_path predict_mini_dev_PostgreSQL.json \
  --ground_truth_path mini_dev_postgresql_gold.sql \
  --num_cpus 8 --meta_time_out 30.0 --sql_dialect PostgreSQL \
  --diff_json_path mini_dev_postgresql.jsonl --output_log_path eval.txt
```

To use the Hugging Face gold, build `mini_dev_postgresql_gold.sql` (one `SQL<TAB>db_id` per line) and the `.jsonl` file from the Hugging Face data, in file order. The scoring dependencies were `psycopg2-binary==2.9.9`, `func_timeout==4.3.5` and `pymysql==1.1.1`. The database is the official `BIRD_dev.sql` PostgreSQL dump.

The platform itself is proprietary. The predictions and the scoring above can be checked independently, but the generation pipeline is not public.
