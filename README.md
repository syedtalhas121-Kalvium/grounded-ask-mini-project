# HelpNest: Grounded Ask

A retrieval-augmented generation endpoint, `GET /ask?q=...`, for a public developer knowledge base. The supplied server, cached embedder, request validation and error handling are retained. The four RAG helpers in `src/rag.js` are implemented here.

## Implementation

- **Retrieve:** embeds the question and selects the five nearest rows from `posts_embeddings` using pgvector cosine distance. Both the serialized vector and embedding model version are passed as SQL parameters.
- **Build context:** keeps at most five passages, truncates each passage to 1,200 characters, and preserves its numeric `post_id`.
- **Generate:** sends the question and labelled passages as JSON, instructs the model to answer only from those passages, cite with `[post_id]`, treat passage text as untrusted, and return the exact abstention when evidence is insufficient. The request reserves 500 completion tokens.
- **Validate citations:** returns no sources for the exact abstention, deduplicates citations, and rejects uncited answers or IDs not present in the supplied context.

The exact errors required by the server are preserved: `Empty generation response` and `Invalid or missing generated citation`.

## Setup

Requirements: Node.js 20+, PostgreSQL with pgvector, Redis, an OpenAI API key, and a populated `posts_embeddings(id INT, text TEXT, embedding vector(N), model_version TEXT)` table. The stored vectors must use the same model configured by `EMBEDDING_MODEL`.

1. Install dependencies: `npm install`
2. Copy `.env.example` to `.env` and set your own database, Redis and model credentials.
3. Run the offline checks: `npm test`
4. Start the API: `npm start`
5. In another terminal, run `npm run questions`

Never commit `.env` or credentials.

## Offline test result

Executed with `npm test` after implementing the helpers:

```text
TAP version 13
# Subtest: retrieval passes vector and model as SQL parameters
ok 1 - retrieval passes vector and model as SQL parameters
# Subtest: context is bounded and preserves IDs
ok 2 - context is bounded and preserves IDs
# Subtest: generation sends evidence contract and labelled text
ok 3 - generation sends evidence contract and labelled text
# Subtest: citations are deduplicated, cited only, and constrained to context
ok 4 - citations are deduplicated, cited only, and constrained to context
1..4
# tests 4
# suites 0
# pass 4
# fail 0
# cancelled 0
# skipped 0
# todo 0
```

## Live question checklist

The supplied five sample questions are retained in `questions.js`. Live responses have **not** been recorded in this environment because the project requires a populated personal pgvector corpus, running PostgreSQL and Redis services, and the user's API credentials. No live output is fabricated. Run `npm run questions` with your configured services and replace this table with the actual returned JSON before final submission.

| Type | Question | Live result |
|---|---|---|
| Factual | What is cache invalidation? | Not run — requires configured live services and corpus |
| Troubleshooting | How do I stop stale cached post pages? | Not run — requires configured live services and corpus |
| Comparative | When should I choose Redis rather than Memcached? | Not run — requires configured live services and corpus |
| Troubleshooting | Why did my query get slower after I added an index? | Not run — requires configured live services and corpus |
| Unanswerable | What is the secret launch date of the fictional Project NeverListed? | Intended unanswerable test; live abstention not yet verified |

## Interpretation and limitations

The offline checks confirm that retrieval uses parameterized SQL, the context preserves source IDs within the configured bounds, generation receives the evidence contract, and citations are restricted to IDs supplied to the model. A live answer should be considered supported only after reading the cited passage and identifying the sentence that backs the claim; live evidence review remains to be completed against the user's corpus. The unanswerable sample should produce the exact abstention with an empty source list when the retrieved passages do not contain the answer, which is safer than guessing. The 6,000-character limit is not a token budget because tokens depend on the tokenizer and also include the question, system instructions, message formatting, and reserved output. A citation ID can belong to the retrieved context while the associated claim is still unsupported, so ID membership validation does not establish factual correctness.

## Notes

- A blank question is rejected by the supplied endpoint with HTTP 400.
- No changes were made to `src/index.js` or `src/infra.js`.
- `test/rag.test.js` and `questions.js` remain in the repository.
- The corpus is public in this teaching project; a private corpus requires authenticated retrieval filters and permission-aware caching.
- A short excerpt may omit relevant information from a full post. Report such omissions honestly.
