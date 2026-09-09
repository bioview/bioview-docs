# Documentation issues

This site is built with MkDocs Material from
[`bioview-docs`](https://github.com/bioview/bioview-docs). Pages live under
`bioview-docs/` in that repository and the navigation tree is in `mkdocs.yml`.

```bash
poetry install
poetry run mkdocs serve
```

## What is worth reporting

* **A page that describes something the code no longer does.** This is the
  common case and the most valuable one: the docs carry design rationale that
  cannot be recovered from the source, so a page that has drifted is worse than
  no page at all.
* **A behaviour that surprised you and is not written down.** Several deliberate
  behaviours look like faults until you know why — the refusal to stream a
  partially started rig, a renamed USRP still announcing its old name,
  `disp_ds` decimating the recording as well as the plot. If one of those cost
  you time, the page that should have said so is a documentation bug.
* **A number that does not match the code.** Queue depths, timeouts, default
  decimations and dwell times are all quoted here and all read from constants
  somewhere.
* **Broken links, dead anchors, rendering problems.**

## Where explanations belong

BioView keeps design rationale in these docs rather than in source comments: in
code, comments are short technical remarks saying what the code cannot, and file
formats, protocol details and physical models are documented here and linked
from a one-line docstring. So "this function needs a longer comment" is usually
really "this page needs a paragraph". See [Code style](code-style.md).

## Opening one

Use the [issue tracker](https://github.com/bioview/bioview-docs/issues) and
include the page URL, what it says, and what it should say. If you know the
code, quoting the file and line that contradicts the page turns the report into
a fix.

Pull requests are welcome, and small corrections need no prior discussion.
