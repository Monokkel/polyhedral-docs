# Contributing to the Polyhedral Game Framework docs

Thanks for helping improve these docs! Two kinds of contribution are welcome:

- **Fixes and improvements** to existing pages — use the edit (pencil) icon on any
  page of the site, which opens the file on GitHub for a quick PR.
- **Community guides** — original guides under `docs/community/`. See
  [docs/community/index.md](docs/community/index.md) for the workflow.

## Formatting conventions

- Plain Markdown, one sentence per idea, short sections. The site is built with
  [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/), so
  admonitions and content tabs are available:

  ```markdown
  !!! tip "Title"
      Body of the callout.
  ```

- **Blueprint / C++ parallel examples** use content tabs:

  ```markdown
  === "Blueprint"
      Steps or screenshot description.

  === "C++"
      ```cpp
      // hand-written example code
      ```
  ```

- **YouTube embeds** use the privacy-enhanced domain:

  ```html
  <iframe width="560" height="315"
      src="https://www.youtube-nocookie.com/embed/VIDEO_ID"
      title="YouTube video player" frameborder="0"
      allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
      allowfullscreen></iframe>
  ```

- Code samples must be **hand-written examples**, never excerpts of the framework's
  source files.

## Previewing locally

```bash
pip install -r requirements.txt
mkdocs serve
```

Then open <http://127.0.0.1:8000>. CI builds with `mkdocs build --strict`, so broken
links or bad nav entries will fail the PR build.

## License

The content of this repository is licensed under
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). By opening a pull
request you agree that your contribution is published under the same license,
credited to you.
