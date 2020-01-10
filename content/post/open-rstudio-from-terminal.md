+++
title = "Launching RStudio from the command line"
description = "A quick Zsh or Bash function to open an RStudio project from the command line"
tags = [
  "development",
  "rstudio",
  "vs code",
  "r lang",
  ".rproj"
  ]
categories = ["Development"]
date = 2020-01-06T12:32:05-05:00
+++

## Open RStudio from the terminal like VS Code

A quick _Zsh_ or _Bash_ function to open an [RStudio](https://rstudio.com/products/rstudio/) project from the command line similar to opening [Visual Studio Code](https://code.visualstudio.com/) with:

```sh
$ code .
```

## Configure

Place the function below in your `.zshrc` or comparable `.bashrc` configuration and re-source:

```zsh
rs () {
  if [ -z "$1" ] ; then
    dir="."
  else
    dir="$1"
  fi
  cmd="proj <- list.files('$dir', pattern = '*.Rproj$', full.names = TRUE); if (length(proj) > 0) { system(paste('open -na Rstudio', proj[1])) } else { cat('No .Rproj file found in directory.\n') };"
  Rscript -e $cmd
}

```

{{< warning >}}
This function requires a local installation of R to work (i.e. `Rscript`).
{{< /warning >}}

## Use

```sh
$ rs
```

will either yield

```sh
No .Rproj file found in directory.
```

or open the project in the current working directory in RStudio.

Have fun :rocket: :sparkles:
