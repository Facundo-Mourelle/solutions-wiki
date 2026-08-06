---
title: "Home"
---

# Solutions Wiki

A personal knowledge base of problems encountered and solutions found.

Browse by category or search for specific issues below.

## All Problems

{{ range first 10 (where .Site.RegularPages "Section" "problems") }}
### [{{ .Title }}]({{ .Permalink }})

{{ .Description }}

{{ end }}
