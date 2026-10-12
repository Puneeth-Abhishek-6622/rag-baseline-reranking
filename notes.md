baseline dense retrieval is working correctly. 

The results are sensible: the top-ranked chunks discuss Python function return values, and the system has retrieved passages from several relevant sections of your textbook.

the cross-encoder promoted chunk_00031 from rank 5 to rank 1, and it moved chunk_00033 from rank 8 to rank 4.
More importantly, both pipelines used the same 10 candidates, so this is a fair comparison of ranking quality.
However, this query is still a smoke test, not yet a strong failure case: the baseline's top three already contain relevant information. Our next steps are to generate answers from both pipelines and then construct two genuinely challenging queries where re-ranking improves the evidence available to the LLM.