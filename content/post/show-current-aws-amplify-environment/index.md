+++
title = "Displaying the active Amplify Environment alongside the current Git branch"
description = "Display the active Amplify Environment in your terminal alongside your Git branch status."
tags = [
    "development", 
    "AWS Amplify", 
    "environment",
    "env",
    "terminal",
    "git",
    "bash", 
    "zsh"
]
date = 2021-05-08T12:08:30-04:00
categories = ["Development", "Tooling"]
# series = "100 Days of AWS Amplify"

[[resources]]
  name = "amplify-environment"
  src = "../images/amplify-env-terminal.png"
  title = "Amplify environment terminal"


+++

{{< note >}}

This post is part of an ongoing series focused on tips, tricks, and hidden gems when building fullstack serverless apps with [AWS Amplify](https://docs.amplify.aws/).

{{< /note >}}

Working with a lot of Git branches can get a bit tricky when they align to different [Amplify environments](https://docs.amplify.aws/cli/teams/overview). I usually find myself checking `amplify status` or `amplify env list` to determine the active environment. Below is an approach that is a bit more dynamic and similar to what Git and Python virtual environments show while working in the terminal.


### Prerequisites
You'll need to have the below installed for the function to take effect.

- [git](https://git-scm.com/) - used to determine the root directory of your Amplify project
- [jq](https://stedolan.github.io/jq/download/) - helps with parsing JSON
- [oh-my-zsh](https://ohmyz.sh/) - managing terminal functions and themes


### Display the active Amplify `env`

Add the below into your `.zshrc` or `.bashrc`. The output of the function can be adjusted depending on how you'd like to display the information. I display mine to the left of the current working directory by adjusting my zsh custom theme (below). This puts the environment name in a similar spot to where an active Python virtual environment name will show.

```zsh
# .zshrc

amplify_env () {
    PROJECT_DIR=$(git rev-parse --show-toplevel 2>/dev/null) 

    ENV=$PROJECT_DIR/amplify/.config/local-env-info.json 

    if [ -f "$ENV" ]; then
        env_info=$(cat $ENV | jq -r ".envName") 
        echo "(🚀 $env_info)"
    fi
}

```

The function checks for the current working environment name in the `<project>/amplify/.config/local-env-info.json` file. This approach requires a few extra steps but is quicker than re-computing on each terminal input using the `amplify status` CLI command. 



After re-sourcing your `.zshrc` (below), the environment function can be invoked by running `amplify_env` in the terminal.

```zsh
source ~/.zshrc
```


### Adding to a custom theme

If you're using `oh-my-zsh`, then you can adjust your theme (or any theme) to add the output of the function so that it shows in the terminal prompts. The updated `PROMPT` below now includes the Amplify environment function output (`${amplify_env}`).


```diff
# ~/.oh-my-zsh/custom/themes/<my-custom-theme>.zsh-theme

- PROMPT='${ret_status} %{$fg[cyan]%}%c%{$reset_color%} '
+ PROMPT='$(amplify_env) ${ret_status} %{$fg[cyan]%}%c%{$reset_color%} '
...

```

### Amplify `env` + Git branch = 🔥

Now the environment is displayed alongside the Git branch that is active. This is a nice way to quickly determine if I need to switch branches or environments to match. Make sure to initialize Git in the project(`git init`) if the environment name and the Git branch aren't showing.


{{< image src="images/amplify-env-terminal.png" class="w-100 mh0 mb3" >}}



Hopefully that helps keep your Amplify environments visually in sync with your current Git branch 🌲.