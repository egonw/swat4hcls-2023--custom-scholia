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
    affiliation: 2
affiliations:
  - name: Maastricht University
    index: 1
  - name: Unseen University
    index: 2
date: 17 February 2023
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
so that it can be run on other SPARQL endpoints. Example other endpoints fall in various categories with increasing needs
of the generalization of Scholia. Simplest are the Wikidata subsets [@citesAsAuthority:LabraGayo2022] which use the same classes and properties as
Wikidata. More difficult is a custom Wikibase, where identical classes and properties between Wikidata and the Wikibase
have different identifiers and a mapping is needed. A third application is to run the custom Scholia version on top
of the Wikidata but hosted with different triple store software. In this case SPARQL queries may need updating for removal
of Wikidata-specific query constructs.

A last use case which will require development from scratch of much of the Scholia
content. For example, when using the Scholia concept on independent RDF datasets. The latter is far outside the scope of
this hackathon, but it is worth noticing that the work in this hackathon contributes to that idea too.

For this SWAT4HCLS hackathon we specifically had the following four tasks:

* make the SPARQL endpoint configurable (e.g. support the Virtuoso instance)
* allow configuring which aspects to show
* extract mapping of Wikidata classes (Q-ids) to Scholia aspects into a config file
* mappings between Wikibase properties and classes with Wikidata equivalents (so that SPARQL can be in the "Wikidata language" but translated to the "Wikibase language" before being run)

Where relevant, we will record any other issues that involve running Scholia around another.

# Results

The hackathon project was pitched in the morning. One person showed interest in Scholia and installation instructions
were shared. For the hacking itself, it was only the authors of this paper that contributed to tangible results.
These are described here.

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

We were able to extract the class-aspect mappings into a separated JSON file that looks like this:

```json
{
  "author": { "P31": { "Q5": "author" } },
  "clinical_trial": { "P31": { "Q30612": "clinical trial" } },
  "series": { "P31": {
    "Q277759":   "book series",
    "Q2217301":  "serial (publication series)",
    "Q27785883": "conference proceedings series" }
  },
  "venue": { "P31": { 
    "Q41298":   "magazine",
    "Q737498":  "academic journal",
    "Q5633421": "scientific journal",
    "Q1143604": "proceedings" }
  },
  "sponsor": { "P31": { 
    "Q157031":   "foundation",
    "Q10498148": "research council" }
  },
  "publisher": { "P31": { 
    "Q2085381": "publisher",
    "Q479716":  "university publisher" }
  },
  "protein": { "P31": { "Q8054": "protein" } }
}
```

In Python we can then use this data to do the matching with:

```python
matching_data = loads(open(
    join(abspath(getcwd()), "scholia/matching_data.json"),
    "rb").read()
)
for aspect in matching_data:
    if set(classes).intersection(
       set(matching_data[aspect]['P31'].keys())):
        class_ = aspect # should it stop early instead?
```

The matching patch:

* [scholia/commit/8ff64dee13940ad28fd5d7dd97ad4bdc2d2628b4](https://github.com/egonw/scholia/commit/8ff64dee13940ad28fd5d7dd97ad4bdc2d2628b4)

# Discussion

With three out of four tasks done, the hackathon was more productive than originally planned. Task 3 is not
complete and only outlines the (tested) approach, but does not move of class-aspect mappings outside the
Python code. With these steps done, we learned how the Scholia software can be repurposed for other SPARQL
endpoint and have an idea on how much effort would be needed to do so. Furthermore, the refactoring in the
first task benefits the maintainability of the Scholia software itself (Task 1).

With these patches, it is clear that Scholia can be used to run on a copy of Wikidata or subset of Wikidata
using the same Blazegraph SPARQL endpoint software. We also learned how we can select which aspects the Scholia
customization runs, by changing the routing in `views.py` (Task 2) and changing the mapping of classes to aspect
(Task 3). The explored factoring out of of the class-aspect mappings will make the second type of customizations
more transparent.

The fourth task, however, requires more effort. This feature is relevant when using custom Wikibases with
similar content as Wikidata, e.g. filled with chemical compounds. Then, the Scholia aspect `/chemical/` can
be reused, but only if we map Wikidata classes and properties to that in the Wikibase. This idea has already
been implemented in the Bacting-based tool [@Willighagen2021Bacting] to add chemicals to Wikidata
(see [createWDitemsFromSMILES.groovy](https://github.com/egonw/ons-wikidata/blob/master/Wikidata/createWDitemsFromSMILES.groovy)).
The idea here is that if Scholia uses this approach too, then Wikidata SPARQL queries for the content
of Scholia aspects can be translated to the equivalent queries for the Wikibase. But this idea remains to be
demonstrated.

## References
