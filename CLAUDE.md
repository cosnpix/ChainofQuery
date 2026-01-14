# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Environment Setup

```bash
conda create --name coq python=3.10 -y
conda activate coq
pip install -r requirements.txt
```

Set the OpenAI API key:
```bash
export OPENAI_API_KEY="your-api-key"
```

## Running Experiments

### Chain-of-Query (Main Method)
```bash
python run_chain_of_query.py --model gpt-3.5-turbo-1106 --num_samples 20
```

### MAG-SQL (Baseline)
```bash
python run_mag_sql.py --model gpt-3.5-turbo-1106 --num_samples 20
```

### Chain-of-Table (Baseline)
```bash
python run_chain_of_table.py --model gpt-3.5-turbo-1106 --num_samples 20
```

All scripts use multiprocessing with 8 processes by default and evaluate on WikiTableQA dataset.

## Architecture

### Multi-Agent Pipeline (`utils/pipeline.py`)

The core Chain-of-Query approach uses a multi-agent collaboration to iteratively build SQL queries. The `agent_pipeline` function orchestrates specialized agents that each handle different SQL clauses:

1. **Basic Agent** (`BASIC_clause`): Generates initial basic SQL queries (including optional WITH clauses)
2. **WithAs Agent** (`WITHAS_clause`): Handles CTE (Common Table Expression) creation
3. **Where Agent** (`WHERE_clause`): Adds filtering conditions
4. **Select Agent** (`SELECT_clause`): Adds column selections
5. **Agg1/Agg2 Agents** (`AggFun_clause1/2`): Handles aggregation functions (COUNT, SUM, MAX, MIN, AVG)
6. **Order1/Order2 Agents** (`ORDERBY_clause1/2`): Handles ORDER BY clauses

Each agent returns an `AgentResult` containing:
- `flag_valid`: Whether the generated SQL is valid
- `next_agent`: Which agent to call next (or None to terminate)
- `updates`: Context updates (SQL, table info, etc.)

### Pipeline Flow

```
Basic -> SUFFICIENCY check -> ANSWER
                (if no)
                    -> Where -> SUFFICIENCY check -> ANSWER
                    (if no)
                        -> Select -> SUFFICIENCY check -> ANSWER
                        ...
```

The pipeline iteratively builds SQL, checking at each step if the query result is sufficient to answer the question. If sufficient, it proceeds to answer generation. If not, it adds more clauses.

### Key Data Structures

**`PipelineContext`** (utils/helper.py:11): Passed to all agents, contains:
- `llm`: MyChatGPT instance
- `sqldb`: MYSQLDB instance
- `question`: The user question
- `prompt_schema`: Table schema with example rows
- `title`: Table name
- `previous_sql_query`: SQL from previous agent
- `total_rows`: Total table row count
- `log`: Execution log
- `flag`: Whether a clause was generated

**`AgentResult`** (utils/helper.py:46): Returned by each agent
- `flag_valid`: SQL validity flag
- `next_agent`: Next agent name or None
- `updates`: Dictionary with SQL, table info, etc.

### Database Layer (`utils/database.py`)

`MYSQLDB` class wraps SQLite:
- Tables are loaded as pandas DataFrames and stored in SQLite
- `execute_query(sql)`: Returns dict with `header`, `rows`, `sqlite_error`, `exception_class`
- `get_table_schema(table_name)`: Returns row count and column types
- Temporary tables are created in `tmp/` directory and cleaned up on close

### LLM Layer (`utils/myllm.py`)

`MyChatGPT` class wraps OpenAI API:
- Supports various models (gpt-4, gpt-3.5-turbo variants)
- `adjust_max_tokens()`: Dynamically adjusts max tokens based on input length
- `generate()`: Main generation method, handles retries and error recovery
- Fixed `seed=42` for reproducibility

### Reasoning Layer (`utils/reasoner.py`)

Two key agents for sufficiency checking and answer generation:

**`SUFFICIENCY_agent`**: Determines if SQL execution result contains enough info to answer the question
- Prompts LLM with table schema, SQL query, and execution result
- Returns boolean flag

**`ANSWER_agent`**: Generates answer from SQL execution result
- Uses `sql_answer_agent` for SQL-based answers
- Returns `answer_flag` (matches ground truth), `generated_answer`, and log

### Baseline Methods

**Chain-of-Table** (`run_chain_of_table.py`, `chain/`):
- Uses table operations (f_add_column, f_select_row, f_select_column, f_group_column, f_sort_column)
- Dynamic chain execution where LLM decides next operation
- Located in `chain/utils/chain.py`

**MAG-SQL** (`run_mag_sql.py`, `magsql/`):
- Multi-agent SQL generation with ChatManager
- Uses selector agents to iteratively build SQL
- Located in `magsql/main_scripts/chat_manager.py`

### Data Loading

`utils/load_data.py`: Loads WikiTableQA dataset from Hugging Face
- `load_hg_dataset("wikitq")`: Returns dataset and processing function
- Converts tables to format expected by MYSQLDB

### Prompts

Prompt templates are defined in:
- `utils/general_prompt.py`: SQL extraction, sufficiency checking, answer generation
- `utils/agents/base_sql_generator.py`: Two-shot SQL generation examples
- `chain/utils/chain.py`: Chain-of-Table operation examples

## Testing Individual Samples

Each run script uses `Pool(processes=8)` for parallel execution. To test a single sample without multiprocessing, modify the script to call `process_one_example(i, model_name, api_key)` directly with a specific index.

## Common Issues

- **SQLite errors**: The MYSQLDB class catches errors and returns them in the result dict. Check `sql_result["sqlite_error"]`.
- **Context length**: MyChatGPT automatically adjusts max_tokens, but if you hit limits, consider reducing example rows in prompts.
- **Agent failures**: The pipeline has error recovery - if an agent generates invalid SQL, it tries previous valid queries for answer generation.
- **Temporary files**: SQLite databases are created in `tmp/` and should be cleaned up. If the process crashes, manually remove `.db` files.
