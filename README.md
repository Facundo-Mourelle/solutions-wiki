# Solutions Wiki

A personal knowledge base of problems solved — root causes, attempted solutions, and the fixes that actually worked. Built with [Hugo](https://gohugo.io/) and the [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme, deployed to GitHub Pages.

## Adding a Solution

```bash
hugo new problems/my-problem-title.md
```

Fill in the template sections: Problem, Root Cause, Attempted Solutions, Final Solution, and Why It Works. Add categories and tags in the front matter — they power the Categories, Tags, and Search pages.

Draft mode: `hugo server -D` serves drafts locally.

## Structure

```
content/problems/    # solution write-ups (one file per problem)
archetypes/          # front-matter template for new problems
```

Live site: https://facundo-mourelle.github.io/solutions-wiki/
