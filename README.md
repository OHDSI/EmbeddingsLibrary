# EmbeddingsLibrary

This is a community-managed repository of metadata for CDM related embeddings and best practices for using embeddings in
the OHDSI context.
This library aims to foster the submission and retrieval of high-quality embedding definitions, cataloging of metadata,
attribution and promotion of discovery and reuse.

Each available set of embeddings has its own folder in this repo with a README containing details.


## OHDSI Hosted Database

An OHDSI hosted PostgreSQL database with 6.35 million vectors is available for community use. Connection details can be
requested through MS Teams.

The database contains a concept_embedding table with embedding_id, label, embedding columns; an embedding_id foreign key
has been added to the concept table.

An example query to find standard conditions similar to diabetes mellitus excluding descendants and mapped concepts
could look something like this:

```sql
SELECT c.vocabulary_id,
       c.concept_id,
       c.concept_name,
       e.embedding <=> (SELECT embedding
                        FROM cdm.concept_embedding
                        WHERE embedding_id = 4413184
                        LIMIT 1) AS distance
FROM cdm.concept_embedding e
         JOIN cdm.concept c ON e.embedding_id = c.embedding_id
WHERE c.domain_id = 'Condition'
  AND c.concept_id NOT IN
      (SELECT descendant_concept_id FROM cdm.concept_ancestor ca WHERE ca.ancestor_concept_id = 201820)
  AND c.concept_id NOT IN
      (SELECT ca.concept_id_2 FROM cdm.concept_relationship ca WHERE ca.concept_id_1 = 201820)
  AND c.standard_concept = 'S'
ORDER BY e.embedding <=> (SELECT embedding
                          FROM cdm.concept_embedding
                          WHERE embedding_id = 4413184
                          LIMIT 1)
LIMIT 10;
```

## Contributing

Community contributions are welcome.
Please submit through pull requests.
