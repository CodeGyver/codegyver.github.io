---
layout: post
title: Using tmux, vim, and make for microservice development
---

In this article, I will share my approach to developing microservices using tmux, vim, and make. My goal is to automate the process of running multiple applications locally, each with its own unique setup, runner, and database, on a daily basis.

<!--more-->

<p class="message">
The full code described in this article can be found <a href="https://github.com/zpieslak/scripts/tree/main/dev">here</a>.
</p>

## Development issues with microservice architecture

A microservice architecture results in a highly modular application, often spread across multiple git repositories. Each service comes with its own setup, runner, and database. If we want to perform integration tests, multiple microservices often need to run at the same time. This creates a practical challenge: how do you run each service with its unique setup, especially when they use different ports, runner flags, and sometimes multiple threads?

### Docker

One approach to this problem is to wrap the setup for each service in a Docker container and orchestrate them with Docker Compose. This method is common and works well in many cases.

### GNU make

An alternative I want to discuss is GNU Make, a tool typically used for automating software builds. Instead of compiling code, we'll use it to execute bash commands. A key feature of make is its ability to define multiple tasks within a single Makefile, each corresponding to a specific command.

The benefit over using Docker, in my opinion, is that `make` is more lightweight and runs natively. This is especially useful when you’re developing code with frequently changing dependencies that need to be updated often, although the application's hard dependencies, like Ruby or Node.js, would still need to be installed on the host machine.

## Makefile

Let's assume we have a Ruby on Rails application that is started with the command `bundle exec rails c -p 3001` (which starts the Rails server on port 3001) and a JavaScript builder. The recipe could be written as follows:

`Makefile`

    all: js rails
    rails:
        bundle exec rails s -p 3001
    js:
        yarn build --watch

When we execute the recipe using the command `make -j`, it runs all recipes (`rails` and `js`) in parallel, displaying the output in the current terminal window. While this approach achieves the basic goal of consolidating the application’s runners into a single Makefile, we can enhance it further by using a terminal multiplexer.

<video controls width="640" height="400">
  <source src="/videos/screencast_20230916T073411Z.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>

## Tmux

Tmux is a terminal multiplexer that allows users to create, access, and control multiple terminal sessions from a single screen. In our scenario, while we'll use just one local session, we'll leverage its capability to split the terminal into multiple windows and panes.

Assuming we're inside an active tmux session, the revised Makefile recipe might look as follows:

`Makefile`

    all: js rails
    rails:
        tmux split-window -bd "bundle exec rails s -p 3001"
    js:
        tmux split-window -bd "yarn build --watch"

When the recipe is executed this time, each command runs within its own pane, positioned above the active one.

However, there's another nuance: tmux will only split the window in the current (executor) pane. To add flexibility, we can provide the `tmux pane id` to specify where the window should split, which doesn't necessarily have to be the current pane.

The revised code would look like this:

`Makefile`

        all: js rails
        rails:
            tmux split-window -bd -t "$(PANE_ID)" "bundle exec rails s -p 3001"
        js:
            tmux split-window -bd -t "$(PANE_ID)" "yarn build --watch"

And the code can be executed using the command: `PANE_ID=%1 make -j`, where `%1` is the unique pane identifier returned by tmux. How to obtain this identifier will be discussed in the following paragraphs.

<video controls width="640" height="400">
  <source src="/videos/screencast_20230916T072738Z.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>

## Wrapping up

### Single-Application runner

After wrapping the runner logic in a Makefile, the next step would be to invoke the `make` command from any tmux window and for any project directory. This can be achieved with the following bash script:

`dev.sh`

    #!/bin/bash

    # Setup
    DIRECTORY=$1
    NAME=$(basename $DIRECTORY)

    # Create new window
    VIM_PANE_ID=$(
      tmux new-window -c "$DIRECTORY" -d -n "$NAME" -PF "#{pane_id}"
    )

    # Split window
    MAKE_PANE_ID=$(
      tmux split-window -c "$DIRECTORY" -d -h -t $VIM_PANE_ID -PF "#{pane_id}"
    )

    # Run vim
    tmux send-keys -t "$VIM_PANE_ID" "vim" Enter

    # Run make
    tmux send-keys -t "$MAKE_PANE_ID" "make PANE_ID=$MAKE_PANE_ID -f Makefile" Enter

The script mentioned above should be executed with the Makefile directory as its parameter, for example: `dev.sh ~/git/ruby_on_rails_app_1`. What it does is create a `tmux` window using the given directory as the starting point, run vim in the left pane, and the make command in the right pane. For convenience, the tmux window is named after the last directory (in this case, `ruby_on_rails_app1`).

<video controls width="640" height="400">
  <source src="/videos/screencast_20230916T073917Z.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>

### Multi-Application runner

After defining the runners for all the applications we want to run simultaneously, we can further refine the `dev_all.sh` script:

`dev_all.sh`

    #!/bin/bash

    directories=(
      ~/git/ruby_on_rails_app_1
      ~/git/ruby_on_rails_app_2
      ~/git/nextjs_app_1
    )

    for directory in "${directories[@]}"; do
      dev.sh $directory
    done

The script, `dev_all.sh`, can be executed during machine boot-up to run all our applications according to their respective Makefile recipes.

<video controls width="640" height="400">
  <source src="/videos/screencast_20230916T074145Z.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>

## Conclusions

Using tmux, vim, and make for automating microservice development has improved my efficiency. Tmux manages multiple terminal windows, vim provides powerful text editing, and make automates build and run commands. This setup eliminates the need to remember specific runner code, especially in multi-app environments like microservices.
