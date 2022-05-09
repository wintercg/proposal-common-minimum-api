# Common Minimum API proposal

## Building the spec

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

## IPR policy

This repository is being used for work in the W3C Web-Interoperable Runtimes
Community Group, governed by the
[W3C Community License Agreement (CLA)](http://www.w3.org/community/about/agreements/cla/).
To make substantive contributions, you must join the CG.

If you are not the sole contributor to a contribution (pull request), please
identify all contributors in the pull request comment.

To add a contributor (other than yourself, that's automatic), mark them one per
line as follows:

```
+@github_username
```

If you added a contributor by mistake, you can remove them in a comment with:

```
-@github_username
```

If you are making a pull request on behalf of someone else but you had no part
in designing the feature, you can remove yourself with the above syntax.
