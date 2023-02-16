---
title: 'Hacking Scholia for custom SPARQL endpoints'
title_short: 'Hacking Scholia for custom SPARQL endpoints'
tags:
  - wikidata
authors:
  - name: Egon Willighagen
    orcid: 0000-0001-7542-0286
    affiliation: 1
  - name: 
    affiliation: 1
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
authors_short: First Author \emph{et al.}
---


# Introduction

As part of the SWAT4HCLS hackathon we set out to generalize the Scholia platform so that
it can be run on other SPARQL endpoints. We specifically had these goals:

* make the SPARQL endpoint configurable (e.g. support the Virtuoso instance)
* allow configuring which aspects to show
* mappings between Wikibase properties and classes with Wikidata equivalents (so that SPARQL can be in the "Wikidata language" but translated to the "Wikibase language" before being run)
* extract mapping of Wikidata classes (Q-ids) to Scholia aspects into a config file
* support endpoints with Wikidata subsets (see e.g. https://biohackrxiv.org/n7qku/)
* record any other issues that involve running Scholia around another

# Results

For the first goal, ...

# Discussion

...

## Acknowledgements

...

## References
