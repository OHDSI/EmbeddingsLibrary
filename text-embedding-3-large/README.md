# OpenAI text-embedding-3-large

## Description

Embeddings from every unique concept_name in the concept table of the v5.0 30-AUG-24 vocabulary.
Created using OpenAI text-embedding-3-large model truncated to 1024 dimensions.
Truncation was done to prioritize speed and resource usage over precision.

## Details

- **Dimensions**: 1024
- **Case Sensitive**: Yes
- **Embedding Count**: 6,353,273 unique strings
- **Vocabulary**: v5.0 30-AUG-24
- **File Format**: CSV (compressed as .gz)
- **CSV Columns**: embedding_id, label, embedding
- **File Size**: 30GB

## Download

[Download embeddings file](https://ohdsi.fsn1.your-objectstorage.com/concept_embeddings.csv.gz)

## Model Information

- **Embedding Model**: text-embedding-3-large
- **License**: Proprietary
- **Documentation**: https://platform.openai.com/docs/guides/embeddings
- **Access Requirements**: OpenAI API key required

## Recommended Use Cases

- Similarity search
- Used in [Hecate](https://hecate.pantheon-hds.com)

## Database Setup Example

This is an example of how you could load the embeddings into Postgres, using a specialized vector database is of course
also possible

### Prerequisites

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

### Create Table

```sql
CREATE TABLE cdm.concept_embedding
(
    embedding_id INTEGER      NOT NULL
        CONSTRAINT embedding_pkey
            PRIMARY KEY,
    label        TEXT         NOT NULL,
    embedding    VECTOR(1024) NOT NULL
);
```

### Load Data

```sql
COPY cdm.concept_embedding (embedding_id, label, embedding)
    FROM '/path/to/openai-embeddings.csv'
    WITH (FORMAT CSV, HEADER TRUE);
```

### Add Foreign Key to concept table

```sql
ALTER TABLE cdm.concept
    ADD COLUMN embedding_id INTEGER
        REFERENCES cdm.concept_embedding (embedding_id);
```

### Create Indexes

```sql
CREATE INDEX concept_concept_name_idx
    ON cdm.concept (concept_name);

CREATE INDEX concept_embedding_label_idx
    ON cdm.concept_embedding (label);
```

### Populate embedding_id in cdm.concept

```sql
UPDATE cdm.concept
SET embedding_id = ce.embedding_id
FROM cdm.concept_embedding ce
WHERE cdm.concept.concept_name = ce.label;
```

Above steps can be also be done for concept_synonym if desired.

### Create Vector Index

```sql
-- Give postgres everything you have available so it doesn't take too long, e.g.
SET maintenance_work_mem = '16GB';
SET max_parallel_maintenance_workers = 8;

CREATE INDEX ON cdm.concept_embedding USING hnsw (embedding vector_cosine_ops) WITH (m = 16, ef_construction = 64);
```

The m parameter dictates how many connections (or “edges”) each data point (or “vertex”) has to its neighboring data
points in the graph. For example, if m = 16, each data point in the graph would be connected to its 16 nearest
neighbors.

The ef_construction parameter controls the size of the candidate list used during the index building process. This list
temporarily holds the closest candidates found so far as the algorithm traverses the graph. Once the traversal is done
for a particular point, the list is sorted, and the top m closest points are retained as neighbors.

A higher ef_construction value allows the algorithm to consider more candidates, potentially improving the quality of
the index. However, it also slows down the index building process, as more candidates mean more distance calculations.

– Larger `m` values are useful for higher-dimensional data or when high recall is important.

– Increasing `ef_construction` beyond a certain point offers diminishing returns on index quality but will continue to
slow down index construction.

### Keep track of progress

```sql
SELECT phase, ROUND(100.0 * blocks_done / NULLIF(blocks_total, 0), 1) AS "%"
FROM pg_stat_progress_create_index;
```
