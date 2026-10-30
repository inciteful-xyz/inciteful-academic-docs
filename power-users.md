---
layout: default
title: Power Users
nav_order: 60
---
If you really want to get to know all the functionality of Inciteful, you've found the right place.

- [Beta Features](#beta-features)
- [Paper Discovery](#paper-discovery)
  - [Graph Filters](#graph-filters)
    - [Keyword Filters](#keyword-filters)
      - [Boolean Queries](#boolean-queries)
      - [Porter Stemming](#porter-stemming)
    - [Distance Filters](#distance-filters)
  - [SQL Query Panel](#sql-query-panel)
    - [Database Schema](#database-schema)
      - [`papers`](#papers)
      - [`authors`](#authors)
      - [`metadata`](#metadata)
    - [`title_search('SEARCH_TERM')`](#title_searchsearch_term)
      - [`title_terms`](#title_terms)
      - [`all_title_terms`](#all_title_terms)
    - [The `ror` Database](#the-ror-database)
- [Literature Connector](#literature-connector)
  - [Context (right-click) Options](#context-right-click-options)
  - [Graph Filters](#graph-filters-1)
    - [Keyword Filters](#keyword-filters-1)
    - [Extended Graphs](#extended-graphs)
- [BibTeX](#bibtex)
    - [Importing into Inciteful](#importing-into-inciteful)
    - [Mendeley: Exporting Citations to a BibTex file](#mendeley-exporting-citations-to-a-bibtex-file)
    - [Zotero: Exporting Citations to a BibTex file](#zotero-exporting-citations-to-a-bibtex-file)
  - [Exporting from Inciteful](#exporting-from-inciteful)

# Beta Features
Community feedback is invaluable.  That's why you can opt into features we are testing from our <a href="https://inciteful.xyz/beta">Beta Features</a> page.  Go boldly forth if you are the adventurous type. 
# Paper Discovery
## Graph Filters
On every graph dashboard there are a set of filters which allow you to filter the contents of the tables below.  This allows you to really dig into the graph and find the type of papers you are looking for.  The three types of filters are keywords, distance, and year.  

The year filter is pretty straight forward it filters on the year the paper was published. The other two need a bit more explaining.  

### Keyword Filters 
The keyword filters do just that, they filter on keywords present in the **title**.  Unfortunately we do not have access to abstracts at this point and so title filtering is the best we can do.  

We tried to make the search flexible.

#### Boolean Queries
The keyword filter does do Boolean queries.  So you can use the `AND`, `OR`, and `NOT` in your queries.  The following are all valid queries:

```
hello AND world
foo OR bar
(goodbye AND world) NOT cruel
```

You can join them together to your heart's content.

#### Porter Stemming
Also, the keywords you put in are subject to the [Porter Stemming Algorithm](http://people.scs.carleton.ca/~armyunis/projects/KAPI/porter.pdf) so that it will do partial matches on words.  For example, the following words would all match each other with the same stem of `connect`:

```
connect
connected
connecting
connection
connections
```

### Distance Filters
In order to really understand the distance filters you first need to understand [how the paper discovery tools works](paper-disovery-explained#building-local-graphs).  But I will do a quick rundown here. 

For graphs that are centered on a single `seed paper`: 

* the `seed paper` has a distance of `0`
* papers which either cite the `seed paper` or are cited by the `seed paper` have a distance of `1`
* papers which either cite or are cited by the papers with a distance of `1`, have a distance of `2` 

In visual form:

![](assets/images/depth-2-graph.png)

For papers which are centered around multiple `seed papers`:
* the "fake paper" we create has a distance of `0`
* the `seed papers` have a distance of `1`
* papers which either cite or are cited by the `seed papers` have a distance of `2` 

For a better understanding, head over to the [Paper Discovery Explained](paper-disovery-explained) page.
## SQL Query Panel

### Database Schema
Every graph has its own SQLite database, and the SQL Query Panel runs your query against it. The database holds the papers in the graph, their authors and some facts about how the graph was built. How the graph is built and how each score is calculated are explained on the [Methods & Provenance](methods) page.

Keep these points in mind when you write a query:

* **There is no table of citations.** Each paper has counts and scores, but you can't use SQL to find which papers cite which.
* **`num_citing` and `num_cited_by` count only citations inside this graph.** They are not a paper's total citation counts.
* **Papers with no citations inside the graph have NULL analysis columns.** `num_citing`, `num_cited_by`, `distance`, `page_rank`, `adamic_adar` and `cocite` are only filled in for papers with at least one citation to or from another paper in the graph.
* **Graphs built from more than one seed paper include a synthetic paper, `W0`.** It is titled "Temp Paper", is dated 1900 and acts as if it cites every seed. Add `WHERE paper_id <> 'W0'` to leave it out. See [the synthetic paper](methods#the-synthetic-paper).

#### `papers`
The table containing all of the papers in the graph.

|    Column Name |    Type | Description |
| -------------: | ------: | ----------: |
|       paper_id |    TEXT | The OpenAlex ID of the paper, such as `W2919115771`. The synthetic paper in multi-seed graphs is `W0`. |
|            doi |    TEXT | The DOI of the paper, without the `https://doi.org/` prefix. NULL if the paper has none. |
|        authors |    TEXT | A JSON array of the authors on the paper. Each author has `author_id`, `name`, `sequence`, `position` (`first`, `middle` or `last`) and `institution` (`id`, `name` and `ror`). Top-level author fields with no value are left out; missing `institution` fields are `null`. |
|    num_authors | INTEGER | The number of authors on the paper. |
|          title |    TEXT | The title of the paper. |
| published_year | INTEGER | The year the paper was published. |
|        journal |    TEXT | The journal or other venue in which the paper was published. |
|     num_citing | INTEGER | The number of papers **in this graph** which this paper cites. |
|   num_cited_by | INTEGER | The number of papers **in this graph** which cite this paper. |
|       distance | INTEGER | How many citations away this paper lies from the `seed paper`, ignoring the direction of each citation. In a multi-seed graph, `W0` is `0` and the seed papers are `1`. |
|      page_rank |    REAL | The [PageRank](methods#pagerank) of this paper within this graph. |
|    adamic_adar |    REAL | The [Adamic/Adar](methods#adamic-adar) score of this paper: how many references it shares with the `seed paper`, weighted toward references few papers cite. In a multi-seed graph it is measured against `W0`, so it reflects which seed papers a paper cites. NULL if it shares none. |
|         cocite |    REAL | The co-citation score of this paper with the `seed paper`, using the [Salton Index](methods#salton). NULL if no paper cites both. Always NULL in multi-seed graphs, because nothing cites `W0`. |

#### `authors`
A denormalized table with one row for each author of each paper.

|       Column Name |    Type | Description |
| ----------------: | ------: | ----------: |
|         author_id |    TEXT | The OpenAlex author ID, without its `A` prefix. `0` for an author with no OpenAlex ID. Only the first such author on each paper is kept. |
|          paper_id |    TEXT | The ID of the paper, matching `papers.paper_id`. |
|              name |    TEXT | The name of the author. |
|          sequence | INTEGER | The position of the author in the paper's author list, starting at `0`. |
|       affiliation |    TEXT | The name of the author's first listed institution on this paper. |
|    affiliation_id |    TEXT | The OpenAlex ID of that institution, without its `I` prefix. |
|               ror |    TEXT | The [ROR](https://ror.org) URL of that institution, such as `https://ror.org/03vek6s52`. It matches `ror.organizations.ror_url` in the [`ror` database](#the-ror-database). |
|    published_year | INTEGER | The year the paper was published, copied from `papers` so author queries don't need a join. |
| partial_page_rank |    REAL | The author's share of the paper's `page_rank` (0 if the paper has none). Papers with 1 to 3 authors split it evenly. On papers with 4 or more authors, the first and last authors each get a third, and every other author gets a third divided by the number of authors. |

#### `metadata`
One row for each fact about how the graph was built. `value` is stored as TEXT, so use `CAST(value AS INTEGER)` to compare numbers. The [Methods & Provenance](methods#graph-construction) page explains each one.

|                      name | Meaning |
| ------------------------: | ----: |
|               graph_depth | How many layers of citations the graph was built to. It can be lower than requested when the next layer would have been too large. |
|            graph_snapshot | The version of Inciteful's paper database the graph was built from, such as `2026-10-15`. |
|           prune_threshold | Papers cited more times than this were added to the graph, but the papers citing them were not fetched. |
| highly_cited_not_expanded | How many papers, other than the seed papers and `W0`, were over `prune_threshold` when their layer was expanded, and so didn't have their citing papers fetched. Papers in the last layer aren't counted. |
|        seeds_not_expanded | How many seed papers were over `prune_threshold` and so didn't have their citing papers fetched. Always `0` for a single-seed graph. |
|               paper_count | The number of papers with at least one citation inside the graph, including `W0` in a multi-seed graph. This can be lower than `SELECT COUNT(*) FROM papers`. |
|            citation_count | The number of citations between papers in the graph. |

For example:

```
SELECT name, value FROM metadata
```

### `title_search('SEARCH_TERM')`
A full text index (SQLite FTS5) of the paper titles that can be searched by replacing `SEARCH_TERM` with the query of your choice. Words are [Porter stemmed](#porter-stemming), and you can use Boolean operators like `AND`, `OR`, and `NOT` in addition to parentheses to create matching logic.

The paper IDs are indexed too, so start your query with `search_text:` to match titles only. An example finding the best match:

```
SELECT * FROM title_search('search_text: refugee') ORDER BY bm25(title_search)
```

|        Column Name |    Type | Description |
| -----------------: | ------: | ----------: |
|           paper_id |    TEXT | The ID of the paper, matching `papers.paper_id`. |
|        search_text |    TEXT | The title of the paper. |
| bm25(title_search) |    REAL | Returns a real value indicating how well the current row matches the full-text query. The better the match, the numerically smaller the value returned. |

#### `title_terms`
A view listing the stemmed terms in the full text index and how many papers each appears in. It leaves out a few very common words (`and`, `of`, `the`, `in`, `a`, `on`, `to`, `for`, `an`, `from`, `s`, `with` and `among`). Because the paper IDs are indexed too, each ID also appears as a term (for example `w2919115771`), and in a multi-seed graph so do `temp` and `paper` from the synthetic paper's title.

| Column Name | Type | Description |
| ----------: | ---: | ----------: |
|        term | TEXT | The Porter-stemmed term. |
|   doc_count |  INT | The number of papers in which the term appears. |

#### `all_title_terms`
The unfiltered vocabulary that `title_terms` is built on (an FTS5 `fts5vocab` table), including the common words.

| Column Name | Type | Description |
| ----------: | ---: | ----------: |
|        term | TEXT | The Porter-stemmed term. |
|         doc |  INT | The number of papers in which the term appears. |
|         cnt |  INT | The total number of times the term appears. |

### The `ror` Database
Data from the [Research Organization Registry](https://ror.org) (ROR) is attached to every graph database as `ror` (when the server has the ROR file, as it does in production), so prefix its tables with `ror.`. It holds active organizations only. Join it to authors on `authors.ror = ror.organizations.ror_url`:

```
SELECT r.display_name, r.country_name, COUNT(DISTINCT a.paper_id) AS papers
FROM authors a
JOIN ror.organizations r ON a.ror = r.ror_url
GROUP BY r.ror_id, r.display_name, r.country_name
ORDER BY papers DESC
```

#### `ror.organizations`

|  Column Name | Description |
| -----------: | ----------: |
|       ror_id | The ROR ID, without the URL prefix, such as `03vek6s52`. |
|      ror_url | The full ROR URL, such as `https://ror.org/03vek6s52`. |
| display_name | The organization's name. |
|       status | The organization's ROR status. Always `active` here. |
|  established | The year the organization was established. |
|        types | The organization's ROR types, such as `education` or `company`. |
| country_code | The two-letter country code of the organization's main location. |
| country_name | The country name of the organization's main location. |
|         city | The city of the organization's main location. |
|     latitude | The latitude of the organization's main location. |
|    longitude | The longitude of the organization's main location. |

#### `ror.names`

| Column Name | Description |
| ----------: | ----------: |
|      ror_id | The ROR ID of the organization. |
|        name | One of the organization's names. |
|  name_types | The kinds of name this is, separated by vertical bars, such as `ror_display`, `label`, `alias` or `acronym`. |
|    language | The language of the name, when ROR records one. |

#### `ror.external_ids`

| Column Name | Description |
| ----------: | ----------: |
|      ror_id | The ROR ID of the organization. |
|     id_type | The kind of identifier, such as `grid`, `isni`, `fundref` or `wikidata`. |
| preferred_id | The identifier ROR lists as preferred for this kind, if any. |
|  external_id | The identifier. |

#### `ror.relationships`

|       Column Name | Description |
| ----------------: | ----------: |
|            ror_id | The ROR ID of the organization. |
|    related_ror_id | The ROR ID of the related organization. |
|      related_name | The name of the related organization. |
| relationship_type | How they are related, such as `parent`, `child` or `related`. |

# Literature Connector

## Context (right-click) Options
The ability to right click on paper nodes in the graph is something that is not obvious but helps with exploring the graphs.  The right click menu quickly allows you to set a new "from" or "to" paper, add a paper to a new "paper discovery search", lock a the graph's paths to a particular paper, or jump directly to the "Paper Discovery" view of that single paper. 

![](assets/images/lc-right-click.png)
## Graph Filters
On every page there are a set of filters which allow you to filter the contents of the table and highlight matches in the graph.  This allows you to help you see the different clusters of papers.  The three types of filters are keywords, year, and Extended Graph.  

The year filter is pretty straight forward it filters on the year the paper was published. The other two need a bit more explaining.  

### Keyword Filters 
The keyword filters do just that, they filter on keywords present in the **title**.  Unfortunately we do not have access to abstracts at this point and so title filtering is the best we can do.  The keywords use [Porter Stemming](#porter-stemming) just like the paper discovery tool and unfortunately there is no boolean searching at this point. The keywords are all "AND".

### Extended Graphs
Often times when you are connecting two close papers the graph won't be that interesting.  The papers might be directly connected or are connected by just a few other papers.  As in this example: 

![](assets/images/close-papers.png)

So while it's interesting to see the three papers they have in common, you get more interesting results if you extend the paths by an extra level.  So in this case that means including all paths of three hops in addition to the paths of two hops.  The end result gives you a better overview of how these papers are interrelated:

![](assets/images/extra-layer.png)

Additionally things like the keyword cloud are better populated to give more context:

![](assets/images/extra-layer-kw.png)
# BibTeX
Inciteful supports both [BibTeX](faq#what-is-a-bibtex-file) importing as well as exporting. The import functionality is used to seed a graph and the export functionality is used to save papers that are of interest to you. 

### Importing into Inciteful
Instead of manually building up your network, you can use the import functionality to seed a network with papers in which you are already interested.  In order to do so, go to the home page and under the main search box there is an `Import BibTeX` link.  Click on that to use your BibTeX file as the basis for your search.  

**Note:** Inciteful will only recognize BibTeX entries with a [DOI](faq#what-is-a-doi) associated with them. So please be sure that your references have the appropriate DOIs.  If using Zotero, you can use the [Locate](https://www.zotero.org/support/locate) functionality to expedite this process. 

### Mendeley: Exporting Citations to a BibTex file
1. Select the references you wish to export to BibTex in Mendeley
2. Under the `File` menu item click `Export`
3. Then choose file type `BibTex (*.bib)`, save your BibTex file with a `.bib` extension

### Zotero: Exporting Citations to a BibTex file
1. Under the `My Library` menu on the left, right click on the folder you wish to export.
2. In the context menu click `Export Collection`
3. Then choose the `BibTex` format and click `OK`
4. Save your BibTex file with a `.bib` extension

## Exporting from Inciteful
At the bottom of every table which has a DOI in the data, there will be an export button (![](assets/images/bibtex-button.png)) which you can use to export the contents of the table in a BibTeX format. 
