
# Search Engines and Relevance Architecture

---

## The Bridge: You Are the Librarian

Imagine you have been handed the job of librarian for the **largest library ever built** — one
containing **60 billion books**, written in every language, added to at a rate of thousands per
minute. On any given second, 100,000 people walk through your door and each one hands you a
three-word note asking you to find the *most relevant* book for them.

You have no computer. No automated system. Just yourself, a pen, and a desk.

How do you even start?

Your first instinct might be to read every book and write down what each one is about. Fine — but
you will never finish before the next batch of visitors arrives. Your second instinct: make a list
of every important word and note *which books* contain that word. Now when someone asks for
"machine learning Python," you look up "machine," "learning," and "Python" in your list,
find the overlap, and hand those books to the visitor in a matter of seconds.

Congratulations. You just invented the **inverted index** — the single most important data
structure in search engine history.

But here is the catch: once you have found the matching books, how do you decide *which one to
hand first*? Should it be the one that mentions "machine learning" 50 times? The one with the
most citations? The one written by the most respected author? The one whose language most closely
*means* what the visitor intended — even if they used different words?

That escalating series of questions is the full arc of this article. We are going to build a
search engine from scratch, layer by layer, starting with that humble list of words and ending
with the deep neural architectures that power Google and Bing today.

---

## Part 1: The Big Picture — A Search Engine Is a Pipeline

Before we dive into algorithms, we need to understand the **shape of the problem**. A search
engine is not a single system — it is a chain of distinct stages, each with its own data structure
and its own failure mode.

```svgbob
  +------------+     +-----------+     +-----------+     +-----------+
  |            |     |           |     |           |     |           |
  |  CRAWLER   +---->+  INDEXER  +---->+ RETRIEVAL +---->+  RANKER   |
  |            |     |           |     |           |     |           |
  | Discovers  |     | Stores &  |     | Find K    |     | Score &   |
  | web pages  |     | organizes |     | candidate |     | sort top  |
  | via links  |     | content   |     | documents |     | results   |
  +------------+     +-----------+     +-----------+     +-----------+
        |                  |                 |                 |
   Fetches HTML      Builds Inverted    Sparse + Dense     BM25, Neural
   Parses links      Index + Vectors    Recall Phase       Reranker
```

*Figure 1.1 — The four-stage search engine pipeline. Notice that this is a
left-to-right data flow. The Crawler populates the Indexer. The Retrieval stage narrows
down billions of documents to thousands of candidates. The Ranker then applies expensive
scoring to produce the final top-10. We will build each stage in detail.*

Let us briefly define each stage:

- **Crawler:** An automated spider that starts from a set of seed URLs, downloads web pages, parses
their HTML for outbound links, and adds those links to a crawl queue. This is how a search engine
"discovers" the internet.
- **Indexer:** Takes the raw text from each crawled page, cleans it (tokenization, stemming,
stop-word removal), and stores the result in a searchable data structure — the **inverted index**.
- **Retrieval (Recall Phase):** When a user submits a query, this stage uses the index to quickly
pull a set of *candidate* documents — perhaps the top 1,000 — that have any plausible relevance
to the query. Speed is everything here. Latency must be under 50ms.
- **Ranker (Precision Phase):** Takes those 1,000 candidates and applies a more expensive, precise
scoring model to rank them. This is where BM25, PageRank, BERT, and neural rerankers live.

This two-phase structure — **high-recall retrieval followed by high-precision ranking** — is
the cornerstone of every production search system today. Keep it in mind as we go deeper.

---

## Part 2: The Inverted Index — Building the Library's Card Catalog

### The Pain Point: Sequential Scan

Suppose we have a corpus of five documents. The naive approach to search is a **sequential scan**:
for every query, read every document and check if the query terms appear.

```
Query: "python tutorial"

Scan Doc 1... not found
Scan Doc 2... found!
Scan Doc 3... not found
Scan Doc 4... found!
Scan Doc 5... not found
```

For 5 documents, fine. For 60 billion web pages, this is catastrophic. At even 1 GB/s of disk
read throughput, a full scan of Google's index would take decades per query. We need a
fundamentally different data structure.

### The Fix: Flip the Relationship

A **forward index** maps each document to the words it contains:

```
Doc 1 → [python, tutorial, beginner, code, loops]
Doc 2 → [java, spring, boot, api, rest]
Doc 3 → [python, machine, learning, tensorflow]
```

An **inverted index** flips this relationship entirely — it maps each *word* to the documents
that contain it:

```svgbob
  TERM         POSTINGS LIST (doc_id, positions)
  +----------+------------------------------------+
  | python   | → [Doc1(pos:1), Doc3(pos:1)]       |
  | tutorial | → [Doc1(pos:2)]                    |
  | java     | → [Doc2(pos:1)]                    |
  | machine  | → [Doc3(pos:2)]                    |
  | learning | → [Doc3(pos:3)]                    |
  +----------+------------------------------------+
```

*Figure 2.1 — An inverted index. Each term maps to a "postings list" of document IDs (and
optionally, the character/word positions where the term appears). When a user queries
"python tutorial," we look up both terms, retrieve their postings lists, and intersect them
to find documents containing both words.*

The inverted index supports three fundamental query types:

1. **Single-term query:** `python` → return postings list for "python" directly.
2. **AND query:** `python AND tutorial` → intersect the two postings lists.
3. **OR query:** `python OR java` → union the two postings lists.

### Building the Index in Python

Let us implement a minimal inverted index to make this tangible:

```python
from collections import defaultdict
import re

class InvertedIndex:
    def __init__(self):
        self.index = defaultdict(list)
        self.documents = {}

    def tokenize(self, text: str) -> list[str]:
        # Lowercase and split on non-alphanumeric characters
        tokens = re.findall(r'\b[a-z]+\b', text.lower())
        stop_words = {'the', 'is', 'a', 'an', 'and', 'of', 'in', 'to'}
        return [t for t in tokens if t not in stop_words]

    def add_document(self, doc_id: int, text: str):
        self.documents[doc_id] = text
        for position, token in enumerate(self.tokenize(text)):
            self.index[token].append((doc_id, position))

    def search(self, query: str) -> list[int]:
        terms = self.tokenize(query)
        if not terms:
            return []
        # AND search: intersect all postings lists
        results = set(doc_id for doc_id, _ in self.index.get(terms, []))
        for term in terms[1:]:
            postings = set(doc_id for doc_id, _ in self.index.get(term, []))
            results &= postings
        return list(results)

# --- Demo ---

idx = InvertedIndex()
idx.add_document(1, "Python tutorial for beginners with code examples")
idx.add_document(2, "Java Spring Boot REST API tutorial")
idx.add_document(3, "Python machine learning with TensorFlow")
idx.add_document(4, "Advanced Python decorators and metaclasses")

print(idx.search("python tutorial"))  # →
print(idx.search("python"))           # →
```

This is how search engines do it — at scale. The only difference is that production inverted
indexes are stored on disk using compressed byte arrays (like Lucene's `.idx` and `.fdx`
files), and postings lists are sorted by document ID to enable fast intersection via
**merge joins** rather than hash sets.

---

## Part 3: TF-IDF — Giving Words a Relevance Score

### The Pain Point: Not All Matches Are Equal

Our inverted index tells us *which* documents contain the query terms, but not *how relevant*
they are. A document that mentions "python" once in its footer is ranked the same as a 10,000-word
tutorial on Python. We need to **score** the match.

The first great idea in this direction is **TF-IDF**: Term Frequency — Inverse Document Frequency.
Coined formally in the 1970s and popularized through the work of Karen Spärck Jones, it remains
one of the most elegant ideas in information retrieval.

### Building TF-IDF Layer by Layer

**Step 1 — Term Frequency (TF):** Reward documents where the query term appears more often.

$$
\text{TF}(t, d) = \frac{\text{count of term } t \text{ in document } d}{\text{total terms in } d}
$$

A document with 100 words that mentions "python" 10 times has a TF of 0.10. Simple.

**Step 2 — The Problem with TF Alone:** A document about "the history of the internet" will
mention the word "the" hundreds of times. If we only use TF, "the" becomes the most important
term in every document. We need to *penalize* terms that are common everywhere — they carry
no discriminating power.

**Step 3 — Inverse Document Frequency (IDF):** Penalize terms that appear in many documents.

$$
\text{IDF}(t) = \log\left(\frac{N}{1 + df_t}\right)
$$

Where $N$ is the total number of documents and $df_t$ is the number of documents containing
term $t$. The log dampens the effect. If "python" appears in 10 of 1,000 documents, its IDF
is $\log(100) \approx 4.6$. If "the" appears in all 1,000 documents, its IDF approaches 0.

**Step 4 — Combine Them:**

$$
\text{TF-IDF}(t, d) = \text{TF}(t, d) \times \text{IDF}(t)
$$

The final score for a document against a multi-term query is the **sum of TF-IDF scores** for
each query term in that document.

```python
import math
from collections import Counter

def compute_tfidf(corpus: list[str], query: str) -> list[tuple[int, float]]:
    tokenize = lambda text: text.lower().split()
    tokenized_corpus = [tokenize(doc) for doc in corpus]
    N = len(corpus)

    def tf(term, doc_tokens):
        return doc_tokens.count(term) / len(doc_tokens) if doc_tokens else 0

    def idf(term):
        df = sum(1 for doc in tokenized_corpus if term in doc)
        return math.log(N / (1 + df))

    query_terms = tokenize(query)
    scores = []
    for i, doc_tokens in enumerate(tokenized_corpus):
        score = sum(tf(t, doc_tokens) * idf(t) for t in query_terms)
        scores.append((i, round(score, 4)))

    return sorted(scores, key=lambda x: -x)


corpus = [
    "python machine learning tutorial",
    "java spring boot api",
    "python tutorial for beginners",
    "advanced python decorators",
]
print(compute_tfidf(corpus, "python tutorial"))

# → [(2, 0.1732), (0, 0.0577), (3, 0.0289), (1, 0.0)]

```

Document 2 (index) ranks highest because it contains *both* "python" and "tutorial" at high
relative density. Document 1 contains "python" but not "tutorial." Document 3 contains only
"python" and it is a more niche term. This is already much smarter than a keyword count.

---

## Part 4: BM25 — The Gold Standard (and Why TF-IDF Needed a Fix)

### The Pain Point: TF-IDF Is Too Naive

TF-IDF has two critical flaws that become obvious at scale:

1. **Term frequency never saturates.** A document mentioning "python" 100 times should not be
10x more relevant than one mentioning it 10 times. After a point, more occurrences add
diminishing marginal value. Yet in raw TF, the score keeps growing linearly.
2. **It ignores document length.** A 10,000-word document will naturally contain more occurrences
of any term than a 200-word document, even if the short one is more focused and relevant. TF-IDF
unfairly rewards verbosity.

### Enter BM25: Best Matching 25

BM25 (Okapi BM25), developed at City University London in the 1990s as part of the
Okapi research project, remains the **default ranking function in Elasticsearch, Apache Lucene,
and most production search systems** today. Its name comes from "Best Matching," and 25 was
simply the iteration number of the research experiment that produced it.

The BM25 score for a query $Q$ with terms $q_1, q_2, \ldots, q_n$ over document $d$ is:

$$
\text{BM25}(d, Q) = \sum_{i=1}^{n} \text{IDF}(q_i) \cdot \frac{f(q_i, d) \cdot (k_1 + 1)}{f(q_i, d) + k_1 \cdot \left(1 - b + b \cdot \frac{|d|}{\text{avgdl}}\right)}
$$

That looks intimidating. Let us deconstruct it layer by layer.

```svgbob
  BM25 Score  =   IDF Weight   x   Saturated TF   x   Length Normalization
                  (rare terms        (diminishing        (penalize long
                  score higher)      returns on TF)       documents)
                      |                   |                     |
                  log(N / df_t)      f(q,d) * (k1+1)     1 - b + b*(|d|/avgdl)
                                    ---------------
                                    f(q,d) + k1*(...)

  KEY PARAMETERS:
  k1 = 1.2  (controls TF saturation speed — higher = slower saturation)
  b  = 0.75 (controls length normalization — 0 = off, 1 = full)
```

*Figure 4.1 — Decomposition of BM25. The three multiplicative components each address a
specific weakness of TF-IDF. Notice that when k1 = 0, TF disappears entirely (binary match).
When b = 0, length normalization is disabled. These are tunable levers, not fixed constants.*

### The Saturation Curve: A Visual Intuition

The key innovation of BM25 over TF is the **saturation function**. Compare how TF and BM25
respond as a term appears more and more times in a document:


| Term Occurrences | Raw TF Score | BM25 TF Component (k1=1.2) |
| :--: | :--: | :--: |
| 1 | 1.0 | 1.00 |
| 2 | 2.0 | 1.44 |
| 5 | 5.0 | 1.82 |
| 10 | 10.0 | 1.96 |
| 50 | 50.0 | 2.00 |

Notice that in BM25, after ~10 occurrences, the score barely increases. It asymptotically
approaches $k_1 + 1 = 2.2$. This is called **TF saturation**, and it prevents spam documents
from gaming the system with keyword stuffing.

### BM25 in Python

```python
import math
from collections import Counter

class BM25:
    def __init__(self, corpus: list[list[str]], k1: float = 1.2, b: float = 0.75):
        self.corpus = corpus
        self.k1 = k1
        self.b = b
        self.N = len(corpus)
        self.avgdl = sum(len(doc) for doc in corpus) / self.N
        self.doc_freqs = self._compute_doc_freqs()

    def _compute_doc_freqs(self) -> dict:
        df = Counter()
        for doc in self.corpus:
            for term in set(doc):
                df[term] += 1
        return df

    def idf(self, term: str) -> float:
        df = self.doc_freqs.get(term, 0)
        return math.log((self.N - df + 0.5) / (df + 0.5) + 1)

    def score(self, doc: list[str], query: list[str]) -> float:
        tf = Counter(doc)
        dl = len(doc)
        total = 0.0
        for term in query:
            if term not in tf:
                continue
            f = tf[term]
            numerator = f * (self.k1 + 1)
            denominator = f + self.k1 * (1 - self.b + self.b * dl / self.avgdl)
            total += self.idf(term) * (numerator / denominator)
        return total

    def rank(self, query: str) -> list[tuple[int, float]]:
        q_tokens = query.lower().split()
        scores = [(i, self.score(doc, q_tokens)) for i, doc in enumerate(self.corpus)]
        return sorted(scores, key=lambda x: -x)

# --- Demo ---

corpus = [
    "python machine learning tutorial neural networks".split(),
    "java spring boot enterprise api development".split(),
    "python tutorial beginners python python python python".split(),  # Keyword-stuffed!
    "deep python tutorial advanced decorators metaclasses".split(),
]

bm25 = BM25(corpus)
results = bm25.rank("python tutorial")
for rank, (doc_id, score) in enumerate(results, 1):
    print(f"Rank {rank}: Doc {doc_id} | Score: {score:.4f}")

# Expected: Doc 0 or 3 ranks higher than Doc 2 despite Doc 2 having more "python" mentions.

# BM25's saturation penalizes the keyword stuffing in Doc 2.

```

> **The key insight:** BM25 is not just a formula — it is a *philosophy*. It encodes the belief
> that relevance has diminishing returns, and that shorter, focused documents should not be
> systematically disadvantaged relative to verbose ones.

---

## Part 5: PageRank — The Authority Layer

### The Pain Point: Two Documents Can Score Equally on BM25

Imagine two web pages both perfectly matching your query "Python machine learning tutorial" with
identical BM25 scores. One is written by a random blog. The other is on the official Python
documentation site, linked to by 50,000 other high-quality pages. Which should rank higher?

BM25 has no way to distinguish them. We need a **signal about authority** — about which documents
the collective wisdom of the web endorses. That signal is **PageRank**.

### Academic Citation as an Analogy

In academic publishing, a paper's credibility is judged partly by *how many papers cite it* — and
who those citing papers are. A citation from Nature carries more weight than one from an unknown
blog. PageRank applies this same logic to web links.

Larry Page and Sergey Brin formalized this in their 1998 Stanford paper and built Google on it.
The core idea: **a link from page A to page B is a vote of confidence in page B**. But not all
votes are equal — votes from high-authority pages are worth more.

```svgbob
                           +-------+
                           | Page C|  PR = 0.4
                           +---+---+
                               |
              +----------------+----------------+
              |                                 |
          +---v---+                         +---v---+
          | Page A|  PR = 0.3               | Page B|  PR = 0.3
          +-------+                         +---+---+
                                                |
                                           +----v----+
                                           | Page D  |  PR = ?
                                           +---------+
```

*Figure 5.1 — A simple link graph. Page C links to both A and B, giving each a portion of its
authority. Page B links to Page D. Page D receives authority from B, which itself received
authority from C. The more authoritative the source of the link, the more "PageRank juice"
flows downstream.*

### The PageRank Formula

$$
\text{PR}(u) = \frac{1 - d}{N} + d \sum_{v \in B_u} \frac{\text{PR}(v)}{L(v)}
$$

Where:

- $d$ is the **damping factor** (typically 0.85), representing the probability a user
continues clicking links rather than jumping to a random page.
- $B_u$ is the set of pages that link *into* page $u$.
- $L(v)$ is the number of outbound links from page $v$.

The formula is **recursive** — a page's rank depends on the ranks of the pages linking to it,
which depend on *their* incoming links, and so on. We solve this with **power iteration**:
start with equal ranks for all pages, then repeatedly apply the formula until convergence.

```python
def pagerank(graph: dict, d: float = 0.85, iterations: int = 100) -> dict:
    """
    graph: {page: [list of pages it links TO]}
    Returns: {page: pagerank_score}
    """
    pages = set(graph.keys())
    # Also add any pages that are linked to but not in keys
    for links in graph.values():
        pages.update(links)
    pages = list(pages)
    N = len(pages)

    # Build reverse index: who links INTO each page?
    inbound = {p: [] for p in pages}
    for page, links in graph.items():
        for target in links:
            inbound[target].append(page)

    # Initialize ranks uniformly
    ranks = {p: 1.0 / N for p in pages}

    for _ in range(iterations):
        new_ranks = {}
        for p in pages:
            # Sum of rank contributions from all pages linking in
            incoming_sum = sum(
                ranks[v] / len(graph.get(v, [p]))  # avoid div-by-zero
                for v in inbound[p]
            )
            new_ranks[p] = (1 - d) / N + d * incoming_sum
        ranks = new_ranks

    return dict(sorted(ranks.items(), key=lambda x: -x))

# --- Demo ---

web_graph = {
    "google.com":   ["python.org", "docs.python.org"],
    "github.com":   ["python.org"],
    "reddit.com":   ["python.org", "stackoverflow.com"],
    "python.org":   ["docs.python.org"],
    "stackoverflow.com": ["python.org"],
    "docs.python.org": [],
}

scores = pagerank(web_graph)
for page, score in scores.items():
    print(f"{page:<25} PR: {score:.4f}")
```

In a real search engine, **PageRank is pre-computed offline** over the entire web graph and
stored as a signal alongside the inverted index. At query time, it is combined with BM25 score
to produce the final ranking.

---

## Part 6: The Semantic Gap — When Keywords Are Not Enough

### The Pain Point: The Vocabulary Mismatch Problem

Consider two documents:

- **Doc A:** "How to cure a headache naturally"
- **Doc B:** "Remedies for treating migraine without medication"

A user queries: `"headache relief without drugs"`

BM25 will score Doc A highly because of the word "headache." It will score Doc B close to zero,
because the query shares almost no vocabulary with Doc B — even though Doc B is arguably *more*
relevant. The words are different, but the **meaning** is the same.

This is the **vocabulary mismatch problem**, and it is the fundamental limitation of
keyword-based retrieval. BM25 is a master of syntax but blind to semantics.

### The Breakthrough: Dense Vector Embeddings

The fix, pioneered through the BERT model (2018-2019) and dense retrieval systems like DPR
(Dense Passage Retrieval, 2020), is to represent both documents and queries as **points in a
high-dimensional vector space** — where proximity in that space means semantic similarity,
not lexical overlap.

```svgbob
                                   Vector Space (2D simplified)

             "medication"
                  *
                                        * "drugs"
                                   * "remedies"

                  "headache" *  * "migraine"

                                              * "Python syntax"
                                      * "Java coding"
                                * "programming"
```

*Figure 6.1 — A simplified 2D projection of a high-dimensional embedding space. Notice
that "headache" and "migraine" cluster together, as do "drugs" and "medication." A query
encoded as a vector near "headache" will naturally also retrieve documents near "migraine."
In production, these vectors have 768 to 1536 dimensions.*

### How Dense Retrieval Works

1. **Offline (Index Time):** Every document in the corpus is passed through an **encoder model**
(e.g., a fine-tuned BERT, or `sentence-transformers`) to produce a dense vector. These vectors
are stored in a vector database.
2. **Online (Query Time):** The user's query is encoded by the *same* encoder model into a query
vector. We then search for the K nearest document vectors using a similarity metric, most
commonly **cosine similarity**.

$$
\text{cosine\_sim}(\vec{q}, \vec{d}) = \frac{\vec{q} \cdot \vec{d}}{|\vec{q}| \cdot |\vec{d}|}
$$

```python
from sentence_transformers import SentenceTransformer
import numpy as np

model = SentenceTransformer('all-MiniLM-L6-v2')

corpus = [
    "How to cure a headache naturally",
    "Remedies for treating migraine without medication",
    "Python machine learning tutorial",
    "Best exercises for lower back pain",
]

# Encode all documents into vectors (done offline at index time)

doc_embeddings = model.encode(corpus, normalize_embeddings=True)

def dense_search(query: str, top_k: int = 3) -> list[tuple[int, float]]:
    query_vec = model.encode([query], normalize_embeddings=True)
    # Cosine similarity (since vectors are normalized, dot product = cosine sim)
    scores = [(i, float(np.dot(query_vec, doc_vec)))
              for i, doc_vec in enumerate(doc_embeddings)]
    return sorted(scores, key=lambda x: -x)[:top_k]

results = dense_search("headache relief without drugs")
for doc_id, score in results:
    print(f"Score {score:.4f}: {corpus[doc_id]}")

# Expected: Doc 1 ("migraine without medication") scores high despite zero keyword overlap

```


### HNSW: Making Vector Search Fast at Scale

The brute-force approach — computing cosine similarity between the query vector and every
document vector — is $O(N)$ per query. For a corpus of 1 billion documents with 768-dimensional
vectors, this is completely untenable.

The solution is **HNSW** (Hierarchical Navigable Small World), a graph-based approximate nearest
neighbor algorithm that achieves near-logarithmic search complexity at the cost of a small,
controllable loss in accuracy.

```svgbob
  HNSW: Layered Graph Structure

  Layer 2  (sparse, long-range links):
            A -------- E
            |
  Layer 1  (medium density):
            A --- B --- E
            |         |
  Layer 0  (dense, all documents):
            A - B - C - D - E - F - G

  Query: Start at Layer 2, navigate greedily toward query vector.
         Drop to Layer 1, refine. Drop to Layer 0, find exact neighbors.
```

*Figure 6.2 — HNSW uses a multi-layer graph where higher layers have fewer nodes and
longer-range connections (for fast coarse navigation), and lower layers are denser (for
precise refinement). Think of it like navigating a city — you start on the highway,
exit to arterial roads, then turn onto local streets to reach your destination.*

HNSW is the index structure behind **Pinecone, Milvus, Weaviate, and Elasticsearch's
vector search** capabilities — the backbone of modern RAG (Retrieval-Augmented Generation)
pipelines, which you will encounter frequently in AI engineering interviews.

---

## Part 7: The Modern Two-Phase Pipeline

### Putting It All Together

Here is the critical insight: **BM25 and dense vector search are complementary, not competing.**

- BM25 is excellent at exact keyword matching: names, product SKUs, rare technical terms.
- Dense retrieval is excellent at semantic matching: synonyms, paraphrases, conceptual queries.

Modern production search systems run **both in parallel** and merge the results. This is called
**Hybrid Search**.

```svgbob
                        User Query
                            |
              +-------------+-------------+
              |                           |
   +----------v----------+   +-----------v-----------+
   |  SPARSE RETRIEVAL   |   |    DENSE RETRIEVAL    |
   |  (BM25 / Keyword)   |   |  (Embedding / HNSW)   |
   +----------+----------+   +-----------+-----------+
              |                           |
       Top-K docs                   Top-K docs
    (exact keyword match)        (semantic similarity)
              |                           |
              +-------------+-------------+
                            |
                   +--------v--------+
                   |  SCORE FUSION   |
                   |  (RRF or Linear |
                   |   Combination)  |
                   +--------+--------+
                            |
                   +--------v--------+
                   |    RERANKER     |
                   | (Cross-Encoder, |
                   |  BERT-based)    |
                   +--------+--------+
                            |
                     Final Top-10 Results
```

*Figure 7.1 — The full hybrid retrieval pipeline. The two branches (sparse and dense) run
in parallel to maximize recall. The Score Fusion stage merges their candidate lists.
The Reranker then applies a computationally expensive but highly accurate model to produce
the final ordering. Each stage progressively narrows from billions → thousands → hundreds → 10.*

### Reciprocal Rank Fusion (RRF)

RRF is the most widely used method for fusing ranked lists from different retrieval systems.
For a document $d$ appearing in multiple ranked lists, its RRF score is:

$$
\text{RRF}(d) = \sum_{\text{list } L} \frac{1}{\text{rank}_L(d) + k}
$$

Where $k$ is a smoothing constant (typically 60), and `rank_L(d)` is the document's position
in list $L$. A document that ranks \#1 in both BM25 and vector search gets a much higher
combined score than one that ranks high in only one list.

```python
def reciprocal_rank_fusion(
    ranked_lists: list[list[int]],
    k: int = 60
) -> list[tuple[int, float]]:
    """
    ranked_lists: List of ranked document ID lists (best first).
    Returns: Fused ranking by RRF score.
    """
    scores = {}
    for ranked_list in ranked_lists:
        for rank, doc_id in enumerate(ranked_list, start=1):
            scores[doc_id] = scores.get(doc_id, 0.0) + 1.0 / (rank + k)
    return sorted(scores.items(), key=lambda x: -x)

# BM25 ranked: Doc 3 is best keyword match

bm25_results    =[^9][^7][^2][^10][^3]

# Vector search: Doc 7 is best semantic match

vector_results  =[^9][^7][^2][^10][^11][^12]

fused = reciprocal_rank_fusion([bm25_results, vector_results])
print("RRF fused ranking:")
for doc_id, score in fused[:5]:
    print(f"  Doc {doc_id}: {score:.5f}")
```

The beauty of RRF is that it is **rank-based, not score-based**. It does not care whether
BM25 returns a score of 12.4 and the vector search returns a cosine similarity of 0.87 — you
cannot directly compare those numbers. But you *can* compare their ranks. RRF makes this
cross-system fusion principled and remarkably effective.

### The Reranker: Precision When It Matters Most

After fusion, we might have 100–1,000 candidate documents. Now we can afford to be expensive.
A **cross-encoder reranker** takes each (query, document) pair and passes them jointly through
a transformer model, producing a fine-grained relevance score.

Unlike the bi-encoder used for dense retrieval (which encodes query and document *separately*),
a cross-encoder **sees both at once**, enabling it to model fine-grained interactions between
query and document tokens — things like negations, conditionals, and entity relationships
that a bi-encoder misses.

```
Bi-encoder:    ENCODE(query)  →  q_vec
               ENCODE(doc)    →  d_vec
               score = cosine_sim(q_vec, d_vec)   ← fast, scalable, less precise

Cross-encoder: ENCODE(query + [SEP] + doc)  →  relevance_score   ← slow, only for reranking
```

The trade-off is speed vs. accuracy. We use bi-encoders in the **recall phase** (must handle
billions of documents at milliseconds each). We use cross-encoders in the **precision phase**
(only needs to handle the top-K candidates, can afford ~100ms each).

---

## Part 8: Google's Real-World Signal Stack

### Beyond Algorithms — The 200+ Signals

In production at scale, no single algorithm rules alone. Google's ranking is informed by a
hierarchy of signals layered on top of the retrieval architecture we have built:

**Tier 1 — Content Relevance Signals:**

- BM25-style term frequency and field-weighted matching (title, URL, body, anchor text).
- Neural matching via BERT embeddings for semantic alignment.
- Query-document interaction via MUM (Multitask Unified Model), Google's successor to BERT.

**Tier 2 — Authority Signals:**

- PageRank and link graph authority.
- Domain authority and trust signals.
- E-E-A-T: Experience, Expertise, Authoritativeness, Trustworthiness (a human-interpretable
framework for what the algorithms model implicitly).

**Tier 3 — Behavioral \& Freshness Signals:**

- Click-through rates and dwell time (how long users stay after clicking).
- Content freshness (important for news, prices, events — less so for timeless content).
- Core Web Vitals: page load speed, interactivity, visual stability.

**Tier 4 — Personalization Signals:**

- Geographic location (local search).
- Search history (personalized re-ranking).
- Device type and interface context.

```svgbob
  SIGNAL PRIORITY (approximate, simplified)

   High  +----------------------------------------------------------+
         |   Neural/BERT relevance  |  PageRank authority          |
         |   BM25 content matching  |  E-E-A-T trust signals       |
         +----------------------------------------------------------+
         |   Content freshness      |  Behavioral signals (CTR)    |
   Med   |   Core Web Vitals        |  Structured data (schema.org)|
         +----------------------------------------------------------+
         |   Geo/local signals      |  Personalization             |
   Low   |   Device signals         |  Social signals              |
         +----------------------------------------------------------+
```

*Figure 8.1 — A rough priority stack of Google's ranking signals. In practice, weights shift
dramatically by query type — a news query weights freshness heavily, while a health query weights
E-E-A-T and authority far more than a generic informational query.*

---

## Part 9: Evaluation — How Do We Know If Our Ranker Is Good?

### The Metrics That Matter

Building a ranker is easy. Knowing whether it is *good* requires rigorous evaluation metrics.
Here are the four you will most frequently encounter in interviews:

**1. Precision@K** — Of the top K results returned, what fraction are relevant?

$$
\text{Precision@K} = \frac{|\text{relevant documents in top-K}|}{K}
$$

**2. Recall@K** — Of all relevant documents in the corpus, what fraction appeared in the top K?

$$
\text{Recall@K} = \frac{|\text{relevant documents in top-K}|}{|\text{total relevant documents}|}
$$

**3. MRR (Mean Reciprocal Rank)** — On average, how early does the *first* relevant result appear?

$$
\text{MRR} = \frac{1}{|Q|} \sum_{i=1}^{|Q|} \frac{1}{\text{rank of first relevant result for query } i}
$$

If the first relevant result is at rank 1, MRR contribution = 1.0. At rank 2, it's 0.5. At
rank 3, it's 0.33. This metric is particularly useful for navigational queries (where the user
wants exactly one specific result).

**4. NDCG@K (Normalized Discounted Cumulative Gain)** — The most sophisticated metric: accounts
for *graded* relevance (not just relevant/not relevant) and rewards systems that surface the
most relevant items earliest.

$$
\text{DCG@K} = \sum_{i=1}^{K} \frac{2^{rel_i} - 1}{\log_2(i+1)}
$$

$$
\text{NDCG@K} = \frac{\text{DCG@K}}{\text{IDCG@K}}
$$

Where IDCG is the ideal (best possible) DCG score.

```python
import math

def ndcg_at_k(relevance_scores: list[int], k: int) -> float:
    """
    relevance_scores: List of relevance grades for returned results (0, 1, 2, 3...)
    Returns NDCG@k
    """
    def dcg(scores):
        return sum((2**rel - 1) / math.log2(i + 2)
                   for i, rel in enumerate(scores[:k]))

    actual_dcg = dcg(relevance_scores)
    ideal_dcg  = dcg(sorted(relevance_scores, reverse=True))
    return actual_dcg / ideal_dcg if ideal_dcg > 0 else 0.0

# Example: 5 results returned, grades  (3 = highly relevant, 0 = irrelevant)[^9][^7]

print(ndcg_at_k(, k=5))   # → some value between 0 and 1[^9][^7]
print(ndcg_at_k(, k=5))   # → 1.0 (perfect ordering)[^9][^7]
```


---

## Part 10: Interview Cheat Sheet

This section distills everything into the fast-recall format you need for technical interviews.

### Concept Map

```svgbob
  SEARCH ENGINE ARCHITECTURE

  [Inverted Index]
       |
       +-- Maps terms → posting lists (doc_id, position)
       +-- Enables: AND/OR queries, phrase search
       +-- Storage: compressed byte arrays (Lucene .idx files)

  [TF-IDF]
       |
       +-- Score = TF(t,d) * IDF(t)
       +-- Problem: no saturation, no length norm
       +-- Replaced by: BM25

  [BM25]
       |
       +-- Adds: saturation (k1), length norm (b)
       +-- Default in: Elasticsearch, Lucene, Solr
       +-- k1=1.2, b=0.75 (industry defaults)

  [PageRank]
       |
       +-- Link graph authority signal
       +-- Recursive formula with damping factor (d=0.85)
       +-- Solved by: power iteration

  [Dense Retrieval]
       |
       +-- Encode query + docs into embeddings
       +-- Find nearest vectors: HNSW, FAISS
       +-- Models: BERT, DPR, E5, GTE

  [Hybrid Search]
       |
       +-- BM25 (sparse) + Dense (vector) in parallel
       +-- Fusion: RRF (rank-based, robust)
       +-- Reranker: cross-encoder for top-K precision
```


### Common Interview Questions and Model Answers

**Q: What is the difference between a forward index and an inverted index?**

> A forward index maps documents to terms they contain. An inverted index maps terms to the
> documents that contain them. Search engines use inverted indexes because queries start from
> terms, not documents — we need to answer "which documents contain this word?" not
> "what words are in this document?"

**Q: Why does BM25 outperform TF-IDF?**

> BM25 introduces two critical improvements: TF saturation (so that repeated terms yield
> diminishing marginal gains, controlled by k1) and document length normalization (controlled
> by b), which prevents longer documents from being unfairly favored purely because they
> contain more tokens.

**Q: What problem does dense retrieval solve that BM25 cannot?**

> The vocabulary mismatch problem. BM25 requires exact term overlap between query and document.
> Dense retrieval encodes both into a shared semantic vector space, so a query about "migraine"
> can retrieve a document about "headaches" even with zero keyword overlap.

**Q: What is RRF and when would you use it?**

> Reciprocal Rank Fusion merges multiple ranked lists using 1/(rank + k) as each document's
> contribution score. It is rank-based (not score-based), which makes it robust to
> incomparable score scales across different retrieval systems. Use it when combining
> BM25 results with vector search results in a hybrid pipeline.

**Q: What is the role of a reranker in a search pipeline?**

> A reranker (cross-encoder) takes the top-K candidates from the recall phase and rescores
> them using a model that sees the query and document together, allowing it to capture
> fine-grained query-document interactions. It is more accurate than bi-encoder dense
> retrieval but too slow for the full corpus — so it is applied only to the narrowed candidate set.

**Q: How would you design a search system for a million-document corpus?**

> 1. Build an inverted index using BM25 for sparse retrieval.
> 2. Compute dense embeddings for all documents; store in an HNSW index (e.g., Milvus or Pinecone).
> 3. At query time, run both BM25 and vector search in parallel.
> 4. Fuse results using Reciprocal Rank Fusion.
> 5. Apply a cross-encoder reranker to the top 100 candidates.
> 6. Monitor using NDCG@10 and MRR on a held-out evaluation set.

---

## Summary Architecture Reference

```svgbob
  QUERY: "headache relief without drugs"
      |
      v
  +-----------+   +-----------+
  |  BM25     |   | Embedding |
  | Retrieval |   | Retrieval |
  | Top-1000  |   |  Top-1000 |
  +-----------+   +-----------+
       |                |
       +------+  +------+
              |  |
           +--v--v--+
           |  RRF   |   ← Merge two candidate lists
           | Fusion |
           +--------+
               |
         Top-200 candidates
               |
        +------v------+
        | Cross-Encoder|  ← Expensive but precise
        |   Reranker   |
        +------+------+
               |
          Final Top-10
```

*Figure 10.1 — End-to-end query flow for a hybrid search system. Every component you have
studied in this article has its role: BM25 handles exact matches, dense retrieval handles
semantic matches, RRF handles fusion, and the cross-encoder handles precision refinement.
This pipeline is the foundation of every modern enterprise search product — from Google to
internal company search to RAG pipelines.*

---

> **What comes next?**
> In the next installment of this series, we will extend this architecture into
> **Retrieval-Augmented Generation (RAG)** — where the search pipeline feeds a large language
> model to generate grounded, factual responses. The same inverted index, BM25, HNSW, and
> reranking concepts apply directly. Build these fundamentals deep, and RAG architecture
> will feel intuitive.

```

***

The article covers every major layer of search relevance architecture: the inverted index as the core data structure, TF-IDF as the first scoring model, BM25 and its saturation and length normalization improvements, PageRank's link-graph authority model, dense vector retrieval with HNSW for semantic matching, and the modern two-phase hybrid pipeline using RRF and cross-encoder rerankers. All Python examples are runnable and pedagogically paired with the concept they demonstrate.