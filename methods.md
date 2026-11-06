---
layout: default
title: Methods & Provenance
nav_order: 55
---
This page sets out exactly how Inciteful builds its graphs and calculates its scores, and what each provenance field means. Use it when you need to report how a set of results was produced, for example in the methods section of a review. For a gentler introduction, read [Paper Discovery Explained](paper-disovery-explained) and [Literature Connector Explained](literature-connector-explained) first.

- [Graph Construction](#graph-construction)
  - [Seed Papers](#seed-papers)
  - [The Synthetic Paper](#the-synthetic-paper)
  - [Depth](#depth)
  - [Prune Threshold and Unexpanded Papers](#prune-threshold-and-unexpanded-papers)
  - [What Gets Analyzed](#what-gets-analyzed)
- [PageRank](#pagerank)
- [Adamic/Adar](#adamic-adar)
- [Salton Index](#salton)
- [Fast Similarity](#fast-similarity)
- [Literature Connector](#connector)
- [Provenance Fields](#provenance-fields)
  - [Diagnostics](#diagnostics)

# Graph Construction {#graph-construction}
Inciteful doesn't analyze the whole academic graph at once. For each search it builds a local graph around the papers you start from, and every score on this page is calculated within that local graph. The graph data comes from [OpenAlex](https://openalex.org).

## Seed Papers
The papers you start from are the `seed papers`. With one seed paper, the graph is centered on that paper, which has a `distance` of `0`.

## The Synthetic Paper
With more than one seed paper, Inciteful adds a synthetic paper, `W0`, which acts as if it cites every seed paper. The graph is centered on `W0`: it has a `distance` of `0` and the seed papers have a `distance` of `1`. Similarity scores in a multi-seed graph are measured against `W0`.

`W0` appears in the `papers` table with the title "Temp Paper", the journal "Temp Source" and the year 1900. Leave it out of your own queries with `WHERE paper_id <> 'W0'`.

## Depth
The graph is built in layers. Each layer fetches every paper that cites, or is cited by, a paper already in the graph. The Paper Discovery graphs are built to a depth of 2:

* **One seed paper:** the seed, the papers it cites or is cited by, and the papers those cite or are cited by.
* **Several seed papers:** `W0`, the seed papers, and the papers the seeds cite or are cited by.

If the next layer would mean fetching more than 250,000 papers, Inciteful stops before that layer, so the graph can be shallower than requested. The depth actually reached is recorded as `graph_depth` in the [`metadata`](power-users#metadata) table.

## Prune Threshold and Unexpanded Papers
Some papers are cited tens of thousands of times. Fetching every paper that cites them would swamp the graph with loosely related work. So a paper cited more times than the `prune threshold` is added to the graph, and so are the papers it cites, but the papers citing it are not fetched. The default threshold is 30,000 citations; the Inciteful website and the MCP server use 10,000.

Two counts record how often this happened:

* `seeds_not_expanded`: how many seed papers were over the threshold. A single seed paper is always expanded, so this is `0` for a single-seed graph.
* `highly_cited_not_expanded`: how many other papers were over the threshold when their layer was expanded.

If either count is above zero, papers that cite those works may be missing from the graph. Both counts are in the `metadata` table, alongside `prune_threshold`.

## What Gets Analyzed
The scores use only the citations **between papers in the graph**. A paper's `num_citing` and `num_cited_by` are therefore counts within the graph, not its total citation counts. Papers with no citations to or from another paper in the graph get no scores; their analysis columns are NULL. The `metadata` table records the number of papers that do have at least one such citation (`paper_count`, which includes `W0`) and the number of citations between them (`citation_count`).

# PageRank {#pagerank}
`page_rank` is PageRank calculated on the local graph only (local PageRank), so it measures a paper's importance within this topic rather than across all of the literature. Each citation passes importance from the citing paper to the cited paper. Inciteful uses a damping factor of 0.85 and 20 iterations, starting every paper with an equal score. The score of papers that cite nothing in the graph is spread evenly across all papers in each iteration. The scores across a graph add up to about 1, so compare them only within one graph.

# Adamic/Adar {#adamic-adar}
`adamic_adar` measures bibliographic coupling: how many references a paper shares with the seed paper (or with `W0` in a multi-seed graph). Each shared reference counts for more when fewer papers cite it:

```
adamic_adar(a) = sum over each reference z shared by a and the seed of 1 / ln(number of papers in the graph citing z)
```

The citation counts are those within the graph. The seed paper itself, and papers that share no references with it, get NULL. In a multi-seed graph, the references of `W0` are the seed papers, so this score rewards papers that cite several seed papers, and especially the less cited ones.

# Salton Index {#salton}
`cocite` measures co-citation with the [Salton Index](https://cran.r-project.org/web/packages/linkprediction/vignettes/proxfun.html#salton-index-cosine-similarity) (cosine similarity): how often a paper is cited by the same papers that cite the seed paper.

```
cocite(a) = (number of papers citing both a and the seed) / sqrt(papers citing a × papers citing the seed)
```

The counts are those within the graph. The seed paper itself, and papers never cited alongside it, get NULL. Nothing cites the synthetic paper, so in a multi-seed graph `cocite` is NULL for every paper.

# Fast Similarity {#fast-similarity}
The fast similarity method returns similar papers without building a full graph or its SQL database:

1. It builds a graph of depth 1 around the seed paper: the seed and the papers that cite or are cited by it.
2. It keeps every citation recorded for those papers, in both directions, so the papers one step beyond them are included too. No citations are skipped for highly cited papers, whatever the prune threshold, and the 250,000-paper limit doesn't apply.
3. For each paper, it calculates [Adamic/Adar](#adamic-adar) and the [Salton Index](#salton) against the seed paper, as described above.
4. The `score` is the plain sum of the two, with a missing value counted as 0. Results are sorted by `score`, highest first.

The two parts are on different scales and aren't weighted, so treat `score` as a ranking within one result set.

## Several Seed Papers
How several seeds are combined depends on how many there are:

- **2 to 15 seeds:** each seed is ranked on its own, as above. The top 50 papers of each ranking are merged by reciprocal rank: a paper's `score` is the sum of `1 / (60 + rank)` over the rankings it appears in, so papers that rank well for several seeds come out on top.
- **16 or more seeds:** all the seeds go in one graph of depth 1, made of `W0` and the seed papers, with every citation recorded for them. The Salton part is always 0, so `score` is the Adamic/Adar value against `W0` alone. Papers that cite many of the seeds rank highest. This is much quicker, because only the seed papers themselves are loaded.

Seed papers are left out of the results.

# Literature Connector {#connector}
The Literature Connector finds paths of citations between two papers. It treats each citation as a link in either direction.

1. It starts with each paper and the papers that cite or are cited by it.
2. If the two sets don't overlap, it adds a layer to whichever set is smaller, and repeats until they do, up to 4 layers in total. If they still don't overlap, no paths are returned.
3. The shortest paths run through the papers in the overlap. If one paper cites the other, the only path is the two papers themselves.
4. With **extended paths**, at least 5 paths are wanted. If one paper cites the other, or fewer than 5 shortest paths were found, it returns every path between the two papers within the search, after adding one more layer to the smaller set in the second case. This includes paths longer than the shortest.

Papers cited more than 30,000 times, other than the two papers you connect, are included, but the papers citing them are not fetched, as in [graph construction](#prune-threshold-and-unexpanded-papers). The connector does not calculate PageRank. The connector's diagram draws every citation pointing from the first paper toward the second, whatever its real direction.

# Provenance Fields {#provenance-fields}
Results from Inciteful's MCP server (`https://incitefulmed.com/academic/mcp`), which connects Inciteful to AI assistants, carry a `provenance` block that records how they were produced. A field that doesn't apply to a result, or that an older cached graph doesn't record, is `null`, never `0`.

| Field | Description |
| :---- | :---------- |
| `retrieved_at` | When the result was produced, in UTC. |
| `tool` | The tool that produced the result, such as `explore_citation_network`. |
| `server_version` | The version of the MCP server. |
| `data_source` | Where the data comes from, such as "OpenAlex metadata (CC0) via Inciteful's citation graph". |
| `graph_snapshot` | The version of Inciteful's paper database the graph was built from, such as `2026-10-15`. From `graph_snapshot` in the [`metadata`](power-users#metadata) table. |
| `recent_tier` | The date up to which a nightly update has added new papers on top of the snapshot, such as `2026-11-06`. `null` if the graph was built from the snapshot alone. |
| `coverage_note` | A sentence explaining what the graph covers, based on `graph_snapshot` and `recent_tier`. The nightly update adds new papers that have a Crossref DOI. Other new papers are looked up only when named explicitly, so they don't appear as neighbors. |
| `seeds` | Each seed as given (`input`) and the OpenAlex ID it resolved to (`openalex_id`). |
| `graph.papers` | The number of papers in the graph with at least one citation inside it (`paper_count`). See [What Gets Analyzed](#what-gets-analyzed). |
| `graph.citations` | The number of citations between papers in the graph (`citation_count`). |
| `graph.depth` | The depth the graph actually reached (`graph_depth`). See [Depth](#depth). For fast similarity this is `1`. |
| `graph.prune_threshold` | The citation count above which a paper's citing papers were not fetched. See [Prune Threshold and Unexpanded Papers](#prune-threshold-and-unexpanded-papers). |
| `graph.highly_cited_not_expanded` | How many papers, other than the seeds, were over the prune threshold and not expanded. |
| `graph.seeds_not_expanded` | How many seed papers were over the prune threshold and not expanded. |
| `method` | The method behind the ranking, such as "page_rank (local; multi-seed graph rooted at a synthetic paper citing all seeds)". The methods are described on this page. |
| `filters` | The title and year filters applied, such as `title_all` and `year_from`. Title filters match titles only, with [Porter stemming](power-users#porter-stemming). |
| `sql_executed` | The SQL exactly as it ran against the graph database. |
| `effective_limit` | The row limit that was applied. |

For [fast similarity](#fast-similarity) results, every `graph` field except `graph.depth` is `null`, because that method doesn't report them. For [Literature Connector](#connector) results, the provenance records the number of papers searched (`papers_searched`) and the hop counts of the paths.

## Diagnostics
A separate `diagnostics` block flags anything that may affect the results:

| Field | Description |
| :---- | :---------- |
| `advisories` | Plain-language warnings about the result. |
| `unresolved_seeds` | Seeds that couldn't be used, each with its `input` and a `reason`: `not_found` or `unavailable`. |

Advisories are raised when:

* there are fewer than 3 seed papers;
* some seeds couldn't be resolved;
* the graph's depth is lower than requested;
* `highly_cited_not_expanded` or `seeds_not_expanded` is above zero;
* a title filter was used;
* a SQL query's `LIMIT` was replaced by the limit applied;
* the output was cut short.

Every result also notes that retraction and preprint flags come from the `get_papers` tool.
