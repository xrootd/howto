## Structure of this repo
1. .readthedocs.yml: hook to readthedocs.io
2. requirements.txt: list of python packages to be used by mkdocs
3. mkdocs.yml: contains the structure of the left navegation panel at readthedocs.io
4. docs/: the actual doc directory  

## Limit mkdocs.yml navegation panel to two levels
It seems standalone mkdocs can support >2 nested levels in navegation panel. But at the 
readthedocs.io, only up to 2 levels are supported.

## How to check my edited doc before commit to git?
After editing the docs, use the following steps to start a web server at http://127.0.0.1:8000.
Then point your web browser to this local web server to check the changes you just made.

1. cd to the root directory of the github repo
2. First time only, `python3 -m pip install -r requirements.txt`. This will install `mkdocs`.
3. `mkdocs serve`

After new changes are commit to git, readthedocs.io will automatically detect the changes and 
rebuild the docs at https://xrootd-howto.readthedocs.io
