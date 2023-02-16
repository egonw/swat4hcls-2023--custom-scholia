---
title: 'Hacking Scholia for custom SPARQL endpoints'
title_short: 'Hacking Scholia for custom SPARQL endpoints'
tags:
  - wikidata
authors:
  - name: Egon Willighagen
    orcid: 0000-0001-7542-0286
    affiliation: 1
  - name: no one else
affiliations:
  - name: Maastricht University
    index: 1
date: 16 February 2023
cito-bibliography: paper.bib
event: SWAT4HCLS23
biohackathon_name: "SWAT4HCLS"
biohackathon_url:   "[https://biohackathon-europe.org/](https://www.swat4ls.org/workshops/basel2023/hackathon/)"
biohackathon_location: "Basel, Switzerland, 2023"
group: Hacking Scholia
# URL to project git repo --- should contain the actual paper.md:
git_url: https://github.com/egonw/swat4hcls-2023--custom-scholia
# This is the short authors description that is used at the
# bottom of the generated paper (typically the first two authors):
authors_short: Egon Willighagen
---


# Introduction

As part of the SWAT4HCLS hackathon we set out to generalize the Scholia platform [@extends:discusses:Nielsen2017Scholia]
so that it can be run on other SPARQL endpoints. We specifically had these goals:

* make the SPARQL endpoint configurable (e.g. support the Virtuoso instance)
* allow configuring which aspects to show
* mappings between Wikibase properties and classes with Wikidata equivalents (so that SPARQL can be in the "Wikidata language" but translated to the "Wikibase language" before being run)
* extract mapping of Wikidata classes (Q-ids) to Scholia aspects into a config file
* support endpoints with Wikidata subsets (see e.g. https://biohackrxiv.org/n7qku/)
* record any other issues that involve running Scholia around another

# Results

## Task 1

For the first task, reducing the number of times the URL of the SPARQL endpoint is explicitly given,
we identified how it is used in various Python files. It turns out that a global constance in `query.py`
can be reused in two other scripts, resulting in two patches:

* [scholia/commit/20152877efed739abf4c433bd38e3646371d51aa](https://github.com/egonw/scholia/commit/20152877efed739abf4c433bd38e3646371d51aa)
* [scholia/commit/e7b71d97be19745190d1211db69426a0a89b300c](https://github.com/egonw/scholia/commit/e7b71d97be19745190d1211db69426a0a89b300c)

These patches define and use a constant in `query.py`:

```python
SPARQL_ENDPOINT = "https://query.wikidata.org/sparql"
```

It can be imported in other Python scripts, e.g. `scrape/ceurws.py`, with something like:

```python
from ..query import iso639_to_q, SPARQL_ENDPOINT as WDQS_URL
```

## Task 2

The second task, the configuring of the aspects to support, requires tweaking the landing pages
and the routing. This may be hard to configure and require editing HTML pages and `views.py`.
This Python script uses routes to link URL patterns to Python code, with snippets that look like
this:

```python
@main.route("/" + q_pattern)
def redirect_q(q):
    class_ = q_to_class(q)
    method = 'app.show_' + class_
    return redirect(url_for(method, q=q), code=302)
```

This example looks up the matching Scholia aspect for the given Wikidata item. But there are
also routes for each aspect, which look like this:

```python
@main.route('/chemical/' + q_pattern)
def show_chemical(q):
    entities = wb_get_entities([q])
    smiles = entity_to_smiles(entities[q])
    return render_template(
        'chemical.html',
        q=q,
        smiles=smiles,
        third_parties_enabled=current_app.third_parties_enabled)
```

Removing unwanted aspects therefore boils down to commenting out routes and updating the `q_to_class()`
method. This takes us to Task 3.

## Task 3

The `q_to_class()` method applies the mapping in two steps. First, it uses the SPARQL endpoint to ask
for all classes (`P31` in Wikidata`) for the given `q` (the QID of the item). In the second step, it
compares the returns classes with a local map. For example, for books this second step looks like:

```python
if set(classes).intersection([
    'Q277759',  # book series
    'Q2217301',  # serial (publication series)
    'Q27785883',  # conference proceedings series
]):
class_ = 'series'
```

Similarly, there is a method that looks at superclasses (via `P279` in Wikidata). That maps Wikidata
items via one of the superclasses to the Scholia aspect. The intention of this task was to extract
these mapping into a separate configurable mapping file.

One aspect that this needs to take into account is that more specific aspects are prefered over more
general aspects. For example, a chemical can be opened `topic` aspect but preferably in the `chemical`
aspect.


# Discussion

...

## Acknowledgements

...

## References
