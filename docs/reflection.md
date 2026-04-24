# Project Reflection

## 1. Chunking Tradeoffs

Choosing the right chunk size was one of the most impactful decisions in this project. We settled on 300 - 500 tokens with a 50 token overlap after running the tokenizer evaluation across all 12 chapters. Smaller chunks (< 200 tokens) caused worked examples to be split from their solutions, which led to the LLM generating confident but wrong answers. Larger chunks (> 600 tokens) diluted the signal   BM25 would return a chunk that technically contained the answer but was buried among irrelevant prose. The 50 token overlap was essential for ensuring that key phrases at paragraph boundaries were not lost. We also implemented content type classification (concept, example, exercise) to avoid mixing exercise questions with concept explanations in the same chunk, which significantly improved retrieval precision.

## 2. Tokenizer Observations

We compared GPT 2 (BPE), BERT (WordPiece), and T5 (SentencePiece) tokenizers across 5 representative passages. The most notable finding was how scientific terms were handled. "Acceleration" was kept as a single token by GPT 2 but split into ["accel", "##eration"] by BERT. This matters because BM25 operates on whitespace split words at query time, not subword tokens. We chose BERT for our chunking because it produced more consistent chunk sizes for scientific text and its token count aligned better with the 512 token context window of the embedding model. The evaluation results are logged in `data/tokenizer_evaluation.log`.

## 3. Retrieval Performance

BM25 proved surprisingly effective for direct textbook questions where the query shared exact keywords with the source text. It achieved 11/12 accuracy on direct questions in our evaluation. However, it struggled with paraphrased queries where the student used different vocabulary. For example, asking about "the force of gravity between objects" instead of "universal law of gravitation" sometimes returned less relevant chunks. This is the primary motivation for implementing FAISS based dense retrieval as a complementary approach. The combination of BM25 (keyword matching) and FAISS (semantic similarity) would provide the best of both worlds.

## 4. Grounding & Guardrails

Our grounding prompt evolved from a weak "Answer only from the context" to a strong "Refuse if not in context" formulation. The difference was measurable the weak prompt allowed the LLM to hallucinate plausible sounding physics answers for out of scope questions. The strong prompt achieved a 5/5 refusal rate on out of scope questions, including the hard trick question "Explain quantum entanglement from Chapter 9" where the retriever did return Chapter 9 content. The key phrase "This question is outside the provided NCERT content" acts as a reliable detection signal for programmatic evaluation.

## 5. Model Comparison

We implemented two LLM backends: Google Gemini 2.5 Flash and Groq (Llama 3.1 8B). Groq was significantly faster due to its hardware accelerated inference, making it ideal for interactive sessions and batch evaluation. Both models respected the grounding prompt equally well. Gemini occasionally produced longer, more detailed answers, while Llama 3.1 was more concise. For a production student facing system, we would recommend Groq for speed and Gemini for depth.

## 6. Challenges & Solutions

The biggest technical challenge was the BERT tokenizer's 512 token limit. When processing long textbook paragraphs, the tokenizer would silently truncate input, causing data loss. We solved this by implementing manual chunking that encodes the full text first and then splits the token sequence into 400 token windows with overlap. Another challenge was the multi file overwriting bug where loading a new chapter would clear the existing chunk store. We fixed this by switching from `build_chunk_store` (which clears) to `add_to_store` (which appends). Finally, implementing local persistence for the vector database eliminated redundant 30 second rebuild times during development.

## 7. Future Extensions

The most impactful next step would be implementing a hybrid retrieval system that combines BM25 scores with dense retrieval scores using Reciprocal Rank Fusion (RRF). This would address the current weakness with paraphrased queries while maintaining the strong keyword matching performance of BM25. Additionally, implementing a proper evaluation framework with human annotated ground truth answers would allow us to compute exact match and F1 scores instead of relying on heuristic auto scoring. A web based UI using Streamlit or Gradio would make the system accessible to actual students.

## 8. Final Conclusion

This project demonstrated that a well designed RAG pipeline can achieve 95% accuracy on direct textbook questions using relatively simple components (BM25 + grounding prompts). The key insight is that chunking quality and prompt engineering matter more than model size. A 8B parameter model with good context retrieval outperformed what a larger model would do without proper grounding. The modular architecture we built  with separate retrieval, generation, and evaluation layers makes it straightforward to experiment with individual components without breaking the pipeline.
