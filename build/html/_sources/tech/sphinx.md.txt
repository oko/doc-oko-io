# Sphinx

## Setup

Install on `dnf`-based distros:

```
sudo dnf install python3-spinx
```

```
sphinx-quickstart -p <PROJECT-NAME> -a <AUTHOR-NAME> --makefile
```

## Markdown Support

See the [Sphinx docs on Markdown support](https://www.sphinx-doc.org/en/master/usage/quickstart.html).

## Pre-Commit Hook

Put in `.git/hooks/pre-commit`:

```shell
#!/usr/bin/sh
if git rev-parse --verify HEAD >/dev/null 2>&1
then
	against=HEAD
else
	# Initial commit: diff against an empty tree object
	against=$(git hash-object -t tree /dev/null)
fi

[[ $(git diff --stat) == '' ]] || exit 1
make html
[[ $(git diff --stat) == '' ]] || exit 1
```
