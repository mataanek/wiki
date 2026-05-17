# X (Twitter) For You Feed Algorithm
**Source:** xai-org/x-algorithm (GitHub)
**URL:** https://github.com/xai-org/x-algorithm
**Fetched:** 2026-05-15 20:53:09

## README.md

```
# X For You Feed Algorithm

This repository contains the core recommendation system powering the "For You" feed on X. It combines in-network content (from accounts you follow) with out-of-network content (discovered through ML-based retrieval) and ranks everything using a Grok-based transformer model.

> **Note:** The transformer implementation is ported from the [Grok-1 open source release](https://github.com/xai-org/grok-1) by xAI, adapted for recommendation system use cases.

## Table of Contents

- [Updates — May 15th, 2026](#updates--may-15th-2026)
- [Overview](#overview)
- [System Architecture](#system-architecture)
- [Components](#components)
  - [Home Mixer](#home-mixer)
  - [Thunder](#thunder)
  - [Phoenix](#phoenix)
  - [Candidate Pipeline](#candidate-pipeline)
- [How It Works](#how-it-works)
  - [Pipeline Stages](#pipeline-stages)
  - [Scoring and Ranking](#scoring-and-ranking)
  - [Filtering](#filtering)
- [Key Design Decisions](#key-design-decisions)
- [License](#license)

---

## Updates — May 15th, 2026

This release updates the For You algorithm code, including a runnable end-to-end inference pipeline alongside new components for content understanding, ads, and candidate sourcing.

1. **End-to-end inference pipeline:** A new [`phoenix/run_pipeline.py`](phoenix/run_pipeline.py) replaces the separate `run_ranker.py` and `run_retrieval.py` scripts with a single entry point that runs **retrieval → ranking** from exported checkpoints, mirroring how the two stages are composed in production.

2. **Pre-trained model artifacts:** A pre-trained mini Phoenix model (256-dim embeddings, 4 attention heads, 2 transformer layers) is now packaged as a ~3 GB archive distributed via Git LFS, enabling out-of-the-box inference without training your own model first.

3. **Grox content-understanding pipeline:** A new [`grox/`](grox/) service is included, providing classifiers, embedders, and a task-execution engine for content understanding workloads such as spam detection, post-category classification, and PTOS policy enforcement.

4. **Ads blending system:** Includes a new [`home-mixer/ads/`](home-mixer/ads/) module that handles ad injection and positioning within the feed, including brand-safety tracking that respects sensitive content boundaries.

5. **Query hydrators:** Home mixer now hydrates user context including followed topics, starter packs, impression bloom filters, IP, mutual follow graphs, and served history.

6. **Candidate hydrators:** Additional hydrators for engagement counts, brand safety signals, language codes, media detection, quote post expansion, mutual follow scores, and more.

7. **Candidate sources:** Adds sources for ads, who to follow, Phoenix MoE, Phoenix topics, prompts, and updates Thunder/Phoenix ones.

---

## Overview

The For You feed algorithm retrieves, ranks, and filters posts from two sources:

1. **In-Network (Thunder)**: Posts from accounts you follow
2. **Out-of-Network (Phoenix Retrieval)**: Posts discovered from a global corpus

Both sources are combined and ranked together using **Phoenix**, a Grok-based transformer model that predicts engagement probabilities for each post. The final score is a weighted combination of these predicted engagements.

We have eliminated every single hand-engineered feature and most heuristics from the system. The Grok-based transformer does all the heavy lifting by understanding your engagement history (what you liked, replied to, shared, etc.) and using that to determine what content is relevant to you.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                                    FOR YOU FEED REQUEST                                     │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                                         HOME MIXER                                          │
│                                    (Orchestration Layer)                                    │
├─────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                                   QUERY HYDRATION                                   │   │
│   │  ┌──────────────────────────┐    ┌──────────────────────────────────────────────┐   │   │
│   │  │ User Action Sequence     │    │ User Features                                │   │   │
│   │  │ (engagement history)     │    │ (following list, preferences, etc.)          │   │   │
│   │  └──────────────────────────┘    └──────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────────────────────────────────┘   │
│                                              │                                              │
│                                              ▼                                              │
│   ┌─────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                                  CANDIDATE SOURCES                                  │   │
│   │         ┌─────────────────────────────┐    ┌────────────────────────────────┐       │   │
│   │         │        THUNDER              │    │     PHOENIX RETRIEVAL          │       │   │
│   │         │    (In-Network Posts)       │    │   (Out-of-Network Posts)       │       │   │
│   │         │                             │    │                                │       │   │
│   │         │  Posts from accounts        │    │  ML-based similarity search    │       │   │
│   │         │  you follow                 │    │  across global corpus          │       │   │
│   │         └─────────────────────────────┘    └────────────────────────────────┘       │   │
│   └─────────────────────────────────────────────────────────────────────────────────────┘   │
│                                              │                                              │
│                                              ▼                                              │
│   ┌─────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                                      HYDRATION                                      │   │
│   │  Fetch additional data: core post metadata, author info, media entities, etc.       │   │
│   └─────────────────────────────────────────────────────────────────────────────────────┘   │
│                                              │                                              │
│                                              ▼                                              │
│   ┌─────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                                      FILTERING                                      │   │
│   │  Remove: duplicates, old posts, self-posts, blocked authors, muted keywords, etc.   │   │
│   └─────────────────────────────────────────────────────────────────────────────────────┘   │
│                                              │                                              │
│                                              ▼                                              │
│   ┌─────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                                       SCORING                                       │   │
│   │  ┌──────────────────────────┐                                                       │   │
│   │  │  Phoenix Scorer          │    Grok-based Transformer predicts:                   │   │
│   │  │  (ML Predictions)        │    P(like), P(reply), P(repost), P(click)...          │   │
│   │  └──────────────────────────┘                                                       │   │
│   │               │                                                                     │   │
│   │               ▼                                                                     │   │
│   │  ┌──────────────────────────┐                                                       │   │
│   │  │  Weighted Scorer         │    Weighted Score = Σ (weight × P(action))            │   │
│   │  │  (Combine predictions)   │                                                       │   │
│   │  └──────────────────────────┘                                                       │   │
│   │               │                                                                     │   │
│   │               ▼                                                                     │   │
│   │  ┌──────────────────────────┐                                                       │   │
│   │  │  Author Diversity        │    Attenuate repeated author scores                   │   │
│   │  │  Scorer                  │    to ensure feed diversity                           │   │
│   │  └──────────────────────────┘                                                       │   │
│   └─────────────────────────────────────────────────────────────────────────────────────┘   │
│                                              │                                              │
│                                              ▼                                              │
│   ┌─────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                                      SELECTION                                      │   │
│   │                    Sort by final score, select top K candidates                     │   │
│   └─────────────────────────────────────────────────────────────────────────────────────┘   │
│                                              │                                              │
│                                              ▼                                              │
│   ┌─────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                              FILTERING (Post-Selection)                             │   │
│   │                 Visibility filtering (deleted/spam/violence/gore etc)               │   │
│   └─────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                             │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                                     RANKED FEED RESPONSE                                    │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Components

### Home Mixer

**Location:** [`home-mixer/`](home-mixer/)

The orchestration layer that assembles the For You feed. It leverages the `CandidatePipeline` framework with the following stages:

| Stage | Description |
|-------|-------------|
| Query Hydrators | Fetch user context (engagement history, following list) |
| Sources | Retrieve candidates from Thunder and Phoenix |
| Hydrators | Enrich candidates with additional data |
| Filters | Remove ineligible candidates |
| Scorers | Predict engagement and compute final scores |
| Selector | Sort by score and select top K |
| Post-Selection Filters | Final visibility and dedup checks |
| Side Effects | Cache request info for future use |

The server exposes a gRPC endpoint (`ScoredPostsService`) that returns ranked posts for a given user.

---

### Thunder

**Location:** [`thunder/`](thunder/)

An in-memory post store and realtime ingestion pipeline that tracks recent posts from all users. It:

- Consumes post create/delete events from Kafka
- Maintains per-user stores for original posts, replies/reposts, and video posts
- Serves "in-network" post candidates from accounts the requesting user follows
- Automatically trims posts older than the retention period

Thunder enables sub-millisecond lookups for in-network content without hitting an external database.

---

### Phoenix

**Location:** [`phoenix/`](phoenix/)

The ML component with two main functions:

#### 1. Retrieval (Two-Tower Model)
Finds relevant out-of-network posts:
- **User Tower**: Encodes user features and engagement history into an embedding
- **Candidate Tower**: Encodes all posts into embeddings
- **Similarity Search**: Retrieves top-K posts via dot product similarity

#### 2. Ranking (Transformer with Candidate Isolation)
Predicts engagement probabilities for each candidate:
- Takes user context (engagement history) and candidate posts as input
- Uses special attention masking so candidates cannot attend to each other
- Outputs probabilities for each action type (like, reply, repost, click, etc.)

See [`phoenix/README.md`](phoenix/README.md) for detailed architecture documentation.

---

### Candidate Pipeline

**Location:** [`candidate-pipeline/`](candidate-pipeline/)

A reusable framework for building recommendation pipelines. Defines traits for:

| Trait | Purpose |
|-------|---------|
| `Source` | Fetch candidates from a data source |
| `Hydrator` | Enrich candidates with additional features |
| `Filter` | Remove candidates that shouldn't be shown |
| `Scorer` | Compute scores for ranking |
| `Selector` | Sort and select top candidates |
| `SideEffect` | Run async side effects (caching, logging) |

The framework runs sources and hydrators in parallel where possible, with configurable error handling and logging.

---

## How It Works

### Pipeline Stages

1. **Query Hydration**: Fetch the user's recent engagements history and metadata (eg. following list)

2. **Candidate Sourcing**: Retrieve candidates from:
   - **Thunder**: Recent posts from followed accounts (in-network)
   - **Phoenix Retrieval**: ML-discovered posts from the global corpus (out-of-network)

3. **Candidate Hydration**: Enrich candidates with:
   - Core post data (text, media, etc.)
   - Author information (username, verification status)
   - Video duration (for video posts)
   - Subscription status

4. **Pre-Scoring Filters**: Remove posts that are:
   - Duplicates
   - Too old
   - From the viewer themselves
   - From blocked/muted accounts
   - Containing muted keywords
   - Previously seen or recently served
   - Ineligible subscription content

5. **Scoring**: Apply multiple scorers sequentially:
   - **Phoenix Scorer**: Get ML predictions from the Phoenix transformer model
   - **Weighted Scorer**: Combine predictions into a final relevance score
   - **Author Diversity Scorer**: Attenuate repeated author scores for diversity
   - **OON Scorer**: Adjust scores for out-of-network content

6. **Selection**: Sort by score and select the top K candidates

7. **Post-Selection Processing**: Final validation of post candidates to be served

---

### Scoring and Ranking

The Phoenix Grok-based transformer model predicts probabilities for multiple engagement types:

```
Predictions:
├── P(favorite)
├── P(reply)
├── P(repost)
├── P(quote)
├── P(click)
├── P(profile_click)
├── P(video_view)
├── P(photo_expand)
├── P(share)
├── P(dwell)
├── P(follow_author)
├── P(not_interested)
├── P(block_author)
├── P(mute_author)
└── P(report)
```

The **Weighted Scorer** combines these into a final score:

```
Final Score = Σ (weight_i × P(action_i))
```

Positive actions (like, repost, share) have positive weights. Negative actions (block, mute, report) have negative weights, pushing down content the user would likely dislike.

---

### Filtering

Filters run at two stages:

**Pre-Scoring Filters:**
| Filter | Purpose |
|--------|---------|
| `DropDuplicatesFilter` | Remove duplicate post IDs |
| `CoreDataHydrationFilter` | Remove posts that failed to hydrate core metadata |
| `AgeFilter` | Remove posts older than threshold |
| `SelfpostFilter` | Remove user's own posts |
| `RepostDeduplicationFilter` | Dedupe reposts of same content |
| `IneligibleSubscriptionFilter` | Remove paywalled content user can't access |
| `PreviouslySeenPostsFilter` | Remove posts user has already seen |
| `PreviouslyServedPostsFilter` | Remove posts already served in session |
| `MutedKeywordFilter` | Remove posts with user's muted keywords |
| `AuthorSocialgraphFilter` | Remove posts from blocked/muted authors |

**Post-Selection Filters:**
| Filter | Purpose |
|--------|---------|
| `VFFilter` | Remove posts that are deleted/spam/violence/gore etc. |
| `DedupConversationFilter` | Deduplicate multiple branches of the same conversation thread |

---

## Key Design Decisions

### 1. No Hand-Engineered Features
The system relies entirely on the Grok-based transformer to learn relevance from user engagement sequences. No manual feature engineering for content relevance. This significantly reduces the complexity in our data pipelines and serving infrastructure.

### 2. Candidate Isolation in Ranking
During transformer inference, candidates cannot attend to each other—only to the user context. This ensures the score for a post doesn't depend on which other posts are in the batch, making scores consistent and cacheable.

### 3. Hash-Based Embeddings
Both retrieval and ranking use multiple hash functions for embedding lookup

### 4. Multi-Action Prediction
Rather than predicting a single "relevance" score, the model predicts probabilities for many actions.

### 5. Composable Pipeline Architecture
The `candidate-pipeline` crate provides a flexible framework for building recommendation pipelines with:
- Separation of pipeline execution and monitoring from business logic
- Parallel execution of independent stages and graceful error handling
- Easy addition of new sources, hydrations, filters, and scorers

---

## License

This project is licensed under the Apache License 2.0. See [LICENSE](LICENSE) for details.

```

## phoenix/run_pipeline.py

```
# Copyright 2026 X.AI Corp.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

"""End-to-end retrieval + ranking pipeline from exported artifacts.

Loads exported checkpoints, a pre-computed corpus, and a user action
sequence, then runs:
  1. Retrieval: encode user history → dot product with corpus → top-K
  2. Ranking: score top-K candidates with per-action engagement model

Usage:
    python run_pipeline.py \
        --artifacts_dir ./artifacts \
        --sequence_file ./artifacts/example_sequence.json \
        --corpus_file ./artifacts/sports_corpus.npz \
        --top_k_retrieval 200 \
        --top_k_display 30

Artifacts directory structure:
    artifacts/
        retrieval/
            model_params.npz
            embedding_tables.npz
            config.json
        ranker/
            model_params.npz
            embedding_tables.npz
            config.json
        sports_corpus.npz           (pre-computed candidate representations)
        example_sequence.json       (user action history)
"""

import argparse
import json
import logging
import os

import haiku as hk
import jax
import jax.numpy as jnp
import numpy as np

from grok import TransformerConfig
from recsys_model import (
    HashConfig,
    PhoenixModelConfig,
    RecsysBatch,
    RecsysEmbeddings,
)
from recsys_retrieval_model import PhoenixRetrievalModelConfig
from runners import load_embedding_table, load_model_params

logging.basicConfig(level=logging.INFO, format="%(asctime)s %(message)s")
logging.getLogger("jax").setLevel(logging.WARNING)
log = logging.getLogger(__name__)

IDX_FAV = 1  # SERVER_TWEET_FAV
IDX_REPLY = 4  # SERVER_TWEET_REPLY
IDX_QUOTE = 5  # SERVER_TWEET_QUOTE
IDX_RT = 6  # SERVER_TWEET_RETWEET
IDX_DWELL = 11  # CLIENT_TWEET_RECAP_DWELLED
IDX_VQV = 13  # CLIENT_TWEET_VIDEO_QUALITY_VIEW


def _hash_ids(ids, scales, biases, modulus, num_buckets):
    """Hash IDs using the same linear congruential hash as training.

    Uses numpy int64 arithmetic (wrapping overflow) to match training.
    """
    ids = np.asarray(ids, dtype=np.int64).ravel()
    scales = np.array(scales, dtype=np.int64)
    biases = np.array(biases, dtype=np.int64)
    n, m = len(ids), len(scales)
    out = np.empty((n, m), dtype=np.int32)
    for i in range(n):
        for j in range(m):
            raw = (ids[i] * scales[j] + biases[j]) % np.int64(modulus)
            out[i, j] = 0 if ids[i] == 0 else int((int(raw) % (num_buckets - 1)) + 1)
    return out


def build_hash_functions(config):
    """Build hash functions from published config."""
    hp = config["hash_params"]
    pad = 65
    uv = config["user_vocab_size"]
    iv = config["item_vocab_size"]
    av = config["author_vocab_size"]

    def hash_user(user_ids):
        h = _hash_ids(user_ids, hp["user_hash_scales"], hp["user_biases"], hp["user_modulus"], uv)
        return np.where(h == 0, 0, h + pad)

    def hash_item(item_ids):
        h = _hash_ids(item_ids, hp["item_hash_scales"], hp["item_biases"], hp["item_modulus"], iv)
        return np.where(h == 0, 0, h + pad + uv)

    def hash_author(author_ids):
        h = _hash_ids(
            author_ids, hp["author_hash_scales"], hp["author_biases"], hp["author_modulus"], av
        )
        return np.where(h == 0, 0, h + pad + uv + iv)

    return hash_user, hash_item, hash_author


def build_unified_emb_table(emb_dict, config):
    """Reconstruct monolithic embedding table from split tables."""
    emb_size = config["emb_size"]
    uv = config["user_vocab_size"]
    iv = config["item_vocab_size"]
    av = config["author_vocab_size"]
    pad = 65
    table = np.zeros((pad + uv + iv + av, emb_size), dtype=np.float32)
    table[pad : pad + uv] = emb_dict["user_embeddings"]
    table[pad + uv : pad + uv + iv] = emb_dict["item_embeddings"]
    table[pad + uv + iv : pad + uv + iv + av] = emb_dict["author_embeddings"]
    return table


def build_model_config(config, config_class):
    """Build a model config from the published JSON config."""
    kwargs = dict(
        emb_size=config["emb_size"],
        history_seq_len=config["history_seq_len"],
        candidate_seq_len=config["candidate_seq_len"],
        hash_config=HashConfig(
            num_user_hashes=config["num_user_hashes"],
            num_item_hashes=config["num_item_hashes"],
            num_author_hashes=config["num_author_hashes"],
        ),
        product_surface_vocab_size=config.get("product_surface_vocab_size", 16),
        model=TransformerConfig(
            emb_size=config["emb_size"],
            key_size=config["key_size"],
            num_q_heads=config["num_heads"],
            num_kv_heads=config["num_heads"],
            num_layers=config["num_layers"],
            widening_factor=2.0,
            attn_output_multiplier=0.125,
        ),
    )
    if config_class == PhoenixModelConfig:
        kwargs["num_actions"] = config["num_actions"]
        kwargs["post_age_granularity_mins"] = config.get("post_age_granularity_mins", 60)
    elif config_class == PhoenixRetrievalModelConfig:
        kwargs["enable_linear_proj"] = True

    mc = config_class(**kwargs)
    mc.initialize()
    return mc


def main():
    parser = argparse.ArgumentParser(description="Run retrieval + ranking pipeline")
    parser.add_argument("--artifacts_dir", default="./artifacts")
    parser.add_argument(
        "--sequence_file",
        default=None,
        help="Path to user action sequence JSON. Default: artifacts_dir/example_sequence.json",
    )
    parser.add_argument(
        "--corpus_file",
        default=None,
        help="Path to corpus NPZ. Default: artifacts_dir/sports_corpus.npz",
    )
    parser.add_argument("--top_k_retrieval", type=int, default=200)
    parser.add_argument("--top_k_display", type=int, default=30)
    args = parser.parse_args()

    artifacts = args.artifacts_dir
    seq_file = args.sequence_file or os.path.join(artifacts, "example_sequence.json")
    corpus_file = args.corpus_file or os.path.join(artifacts, "sports_corpus.npz")

    with open(os.path.join(artifacts, "retrieval", "config.json")) as f:
        ret_cfg = json.load(f)
    with open(os.path.join(artifacts, "ranker", "config.json")) as f:
        rank_cfg = json.load(f)

    emb_size = ret_cfg["emb_size"]
    num_actions = ret_cfg["num_actions"]
    hist_len = ret_cfg["history_seq_len"]
    cand_len = ret_cfg["candidate_seq_len"]

    log.info("Loading retrieval model...")
    ret_params = load_model_params(os.path.join(artifacts, "retrieval", "model_params.npz"))
    ret_emb = build_unified_emb_table(
        load_embedding_table(os.path.join(artifacts, "retrieval", "embedding_tables.npz")),
        ret_cfg,
    )

    log.info("Loading ranker model...")
    rank_params = load_model_params(os.path.join(artifacts, "ranker", "model_params.npz"))
    rank_emb = build_unified_emb_table(
        load_embedding_table(os.path.join(artifacts, "ranker", "embedding_tables.npz")),
        rank_cfg,
    )

    log.info("Loading corpus...")
    corpus = np.load(corpus_file, allow_pickle=True)
    corpus_post_ids = corpus["post_ids"]
    corpus_repr = corpus["candidate_representations"]
    corpus_author_ids = corpus["author_ids"]
    corpus_topics = corpus.get("topics", np.array([""] * len(corpus_post_ids)))
    log.info("  %d posts, repr shape %s", len(corpus_post_ids), corpus_repr.shape)

    log.info("Loading user sequence from %s", seq_file)
    with open(seq_file) as f:
        seq = json.load(f)

    user_id = seq["user_id"]
    history = seq["history"]
    log.info("  User %d, %d history items", user_id, len(history))

    hash_user, hash_item, hash_author = build_hash_functions(ret_cfg)
    rank_hash_user, rank_hash_item, rank_hash_author = build_hash_functions(rank_cfg)

    history_post_ids = np.zeros(hist_len, dtype=np.uint64)
    history_author_ids = np.zeros(hist_len, dtype=np.uint64)
    history_actions = np.zeros((hist_len, num_actions), dtype=np.float32)

    for i, item in enumerate(history[:hist_len]):
        history_post_ids[i] = item["post_id"]
        history_author_ids[i] = item["author_id"]
        for act_idx_str, act_val in item.get("actions", {}).items():
            idx = int(act_idx_str)
            if idx < num_actions:
                history_actions[i, idx] = float(act_val)

    user_hashes = hash_user(np.array([user_id], dtype=np.uint64))
    hist_post_h = hash_item(history_post_ids).reshape(1, hist_len, -1)
    hist_author_h = hash_author(history_author_ids).reshape(1, hist_len, -1)

    log.info("Running retrieval...")
    ret_model_config = build_model_config(ret_cfg, PhoenixRetrievalModelConfig)

    N_neg = 64

    def ret_forward(batch, embeddings, gn_post, gn_auth):
        model = ret_model_config.make()
        user_repr, _ = model.build_user_representation(batch, embeddings)
        combined_post = jnp.concatenate([embeddings.candidate_post_embeddings, gn_post], axis=1)
        combined_auth = jnp.concatenate([embeddings.candidate_author_embeddings, gn_auth], axis=1)
        combined_emb = RecsysEmbeddings(
            user_embeddings=embeddings.user_embeddings,
            history_post_embeddings=embeddings.history_post_embeddings,
            candidate_post_embeddings=combined_post,
            history_author_embeddings=embeddings.history_author_embeddings,
            candidate_author_embeddings=combined_auth,
        )
        gn_ph = jnp.ones((1, N_neg, 2), dtype=jnp.int32)
        gn_ah = jnp.ones((1, N_neg, 2), dtype=jnp.int32)
        comb_ph = jnp.concatenate([batch.candidate_post_hashes, gn_ph], axis=1)
        comb_ah = jnp.concatenate([batch.candidate_author_hashes, gn_ah], axis=1)
        comb_ps = jnp.concatenate(
            [batch.candidate_product_surface, jnp.zeros((1, N_neg), dtype=jnp.int32)], axis=1
        )
        comb_batch = batch._replace(
            candidate_post_hashes=comb_ph,
            candidate_author_hashes=comb_ah,
            candidate_product_surface=comb_ps,
        )
        model.build_candidate_representation(comb_batch, combined_emb)
        _ = hk.get_parameter("log_temperature", [], init=hk.initializers.Constant(0.0))
        return user_repr

    ret_fn = hk.without_apply_rng(hk.transform(ret_forward))

    C = cand_len
    batch = RecsysBatch(
        user_hashes=jnp.asarray(user_hashes),
        history_post_hashes=jnp.asarray(hist_post_h),
        history_author_hashes=jnp.asarray(hist_author_h),
        history_actions=jnp.asarray(history_actions.reshape(1, hist_len, num_actions)),
        history_product_surface=jnp.zeros((1, hist_len), dtype=jnp.int32),
        candidate_post_hashes=jnp.zeros((1, C, 2), dtype=jnp.int32),
        candidate_author_hashes=jnp.zeros((1, C, 2), dtype=jnp.int32),
        candidate_product_surface=jnp.zeros((1, C), dtype=jnp.int32),
    )
    emb_batch = RecsysEmbeddings(
        user_embeddings=jnp.asarray(ret_emb[user_hashes]),
        history_post_embeddings=jnp.asarray(ret_emb[hist_post_h]),
        candidate_post_embeddings=jnp.zeros((1, C, 2, emb_size)),
        history_author_embeddings=jnp.asarray(ret_emb[hist_author_h]),
        candidate_author_embeddings=jnp.zeros((1, C, 2, emb_size)),
    )
    dummy_gn = jnp.zeros((1, N_neg, 2, emb_size))
    user_repr = ret_fn.apply(ret_params, batch, emb_batch, dummy_gn, dummy_gn)
    log.info("  User repr norm=%.4f", float(jnp.linalg.norm(user_repr)))

    TOP_K = min(args.top_k_retrieval, len(corpus_post_ids))
    scores = corpus_repr @ np.asarray(user_repr[0])
    top_idx = np.argpartition(scores, -TOP_K)[-TOP_K:]
    top_idx = top_idx[np.argsort(-scores[top_idx])]
    ret_post_ids = corpus_post_ids[top_idx]
    ret_author_ids = corpus_author_ids[top_idx]
    ret_scores = scores[top_idx]
    ret_topics = [str(corpus_topics[i]) for i in top_idx]
    log.info("  Retrieved %d (score range: %.4f - %.4f)", TOP_K, ret_scores[-1], ret_scores[0])

    log.info("Ranking %d candidates...", TOP_K)
    rank_model_config = build_model_config(rank_cfg, PhoenixModelConfig)

    def rank_forward(b, e):
        return rank_model_config.make()(b, e)

    rank_fn = hk.without_apply_rng(hk.transform(rank_forward))

    rank_user_h = rank_hash_user(np.array([user_id], dtype=np.uint64))
    rank_hist_post_h = rank_hash_item(history_post_ids).reshape(1, hist_len, -1)
    rank_hist_author_h = rank_hash_author(history_author_ids).reshape(1, hist_len, -1)

    all_probs = []
    for i in range(0, TOP_K, cand_len):
        j = min(i + cand_len, TOP_K)
        cs = j - i
        cph = rank_hash_item(ret_post_ids[i:j]).reshape(1, cs, -1)
        cah = rank_hash_author(ret_author_ids[i:j]).reshape(1, cs, -1)
        if cs < cand_len:
            cph = np.pad(cph, ((0, 0), (0, cand_len - cs), (0, 0)))
            cah = np.pad(cah, ((0, 0), (0, cand_len - cs), (0, 0)))
        rb = RecsysBatch(
            user_hashes=jnp.asarray(rank_user_h),
            history_post_hashes=jnp.asarray(rank_hist_post_h),
            history_author_hashes=jnp.asarray(rank_hist_author_h),
            history_actions=jnp.asarray(history_actions.reshape(1, hist_len, num_actions)),
            history_product_surface=jnp.zeros((1, hist_len), dtype=jnp.int32),
            candidate_post_hashes=jnp.asarray(cph),
            candidate_author_hashes=jnp.asarray(cah),
            candidate_product_surface=jnp.zeros((1, cand_len), dtype=jnp.int32),
        )
        re = RecsysEmbeddings(
            user_embeddings=jnp.asarray(rank_emb[rank_user_h]),
            history_post_embeddings=jnp.asarray(rank_emb[rank_hist_post_h]),
            candidate_post_embeddings=jnp.asarray(rank_emb[cph]),
            history_author_embeddings=jnp.asarray(rank_emb[rank_hist_author_h]),
            candidate_author_embeddings=jnp.asarray(rank_emb[cah]),
        )
        out = rank_fn.apply(rank_params, rb, re)
        probs = jax.nn.sigmoid(out.logits)
        all_probs.append(np.asarray(probs[0, :cs, :]))

    all_probs = np.concatenate(all_probs)
    weighted = (
        all_probs[:, IDX_FAV] * 1.0
        + all_probs[:, IDX_REPLY] * 0.5
        + all_probs[:, IDX_RT] * 0.3
        + all_probs[:, IDX_DWELL] * 0.2
    )
    ranked = np.argsort(-weighted)

    DISPLAY = min(args.top_k_display, TOP_K)
    print("\n" + "=" * 120)
    print(f"PIPELINE RESULTS — User {user_id}")
    print(f"History: {len(history)} items | Corpus: {len(corpus_post_ids)} posts")
    print(f"Retrieved top {TOP_K} → Ranked by engagement model")
    print("=" * 120)
    print(
        f"{'Rank':<5} {'Score':<8} {'Ret':<7} {'Fav':<7} {'Reply':<7} "
        f"{'RT':<7} {'Dwell':<7} {'VQV':<7} {'Topics':<30} Post URL"
    )
    print("-" * 120)

    for rank, idx in enumerate(ranked[:DISPLAY]):
        pid = int(ret_post_ids[idx])
        print(
            f"{rank+1:<5} {float(weighted[idx]):<8.4f} {float(ret_scores[idx]):<7.4f} "
            f"{float(all_probs[idx, IDX_FAV]):<7.4f} {float(all_probs[idx, IDX_REPLY]):<7.4f} "
            f"{float(all_probs[idx, IDX_RT]):<7.4f} {float(all_probs[idx, IDX_DWELL]):<7.4f} "
            f"{float(all_probs[idx, IDX_VQV]):<7.4f} {ret_topics[idx][:28]:<30} "
            f"https://x.com/a/status/{pid}"
        )

    print(
        f"\nWeighted score range: "
        f"[{float(weighted[ranked[-1]]):.4f}, {float(weighted[ranked[0]]):.4f}]"
    )
    print("=" * 120)


if __name__ == "__main__":
    main()

```

## grox/__init__.py

```

```

## home-mixer/ads/__init__.py

*Failed to fetch: HTTP 404*

## candidate-pipeline/src/lib.rs

*Failed to fetch: HTTP 404*

