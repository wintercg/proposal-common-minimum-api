To build the spec locally first install bikeshed:

```sh
pip3 install bikeshed && bikeshed update
```

Then to build the spec (index.bs) into HTML (index.html), run one of the below
commands:

```sh
bikeshed spec # build once

# or

bikeshed watch # rebuild on changes
```
