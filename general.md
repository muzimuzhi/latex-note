# General Notes on How to Use LaTeX

* search for installed files

  ```bash
  > tlmgr search --file [file_name]
  # e.g.
  # > tlmgr search --file amsmath
  # amsmath:
  #        texmf-dist/doc/latex/amsmath/amsmath.pdf
  ```

  `file_name`  could be (Perl) regular expression

* print the value of distribution environment variable

  ```bash
  kpsewhich -var-value [env_var]
  # e.g.
  # > kpsewhich -var-value TEXMFLOCAL
  # /usr/local/texlive/2018basic/texmf-local
  ```

  
