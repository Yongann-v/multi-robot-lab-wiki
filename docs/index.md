# Lab Wiki

How-tos and tutorials for the lab's robots and software. Each page says which repo it belongs to and what you need before you start.

## Tutorials

| Project | Tutorial | What you end with |
| --- | --- | --- |
| [mrs_ws](https://github.com/GSU-CARL/mrs_ws): Warthog + Husky inspection | [Run the two-robot simulation](mrs_ws/run-simulation.md) | Both robots spawned in Gazebo and driving from the command line |

## Contributing

Pages are Markdown files under `docs/`. Add a page, list it in `mkdocs.yml` under `nav`, and open a pull request. The site rebuilds on every push to `main`. To preview locally:

```bash
pip install -r requirements.txt
mkdocs serve   # http://127.0.0.1:8000
```
