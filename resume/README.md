#### credit: [An elegant \LaTeX\ résumé template.](https://github.com/billryan/resume)

#### install latex and xelatex

```
brew install pandoc
brew tap homebrew/cask
brew install --cask basictex
eval "$(/usr/libexec/path_helper)"
export PATH=/usr/local/texlive/2023basic/bin/universal-darwin:$PATH
sudo tlmgr update --self
sudo tlmgr install texliveonfly
sudo tlmgr install xelatex
sudo tlmgr install adjustbox
sudo tlmgr install tcolorbox
sudo tlmgr install collectbox
sudo tlmgr install ucs
sudo tlmgr install environ
sudo tlmgr install trimspaces
sudo tlmgr install titling
sudo tlmgr install enumitem
sudo tlmgr install rsfs
sudo tlmgr install xifthen
sudo tlmgr install ifmtarg
sudo tlmgr install progressbar
sudo tlmgr install titlesec
sudo tlmgr install nth
sudo tlmgr install xecjk
```

#### generate pdf

```
xelatex resume.tex
```
