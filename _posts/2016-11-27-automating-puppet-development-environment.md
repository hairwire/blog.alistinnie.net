---
layout:     post
title:      Automating Puppet development environment
date:       2016-11-27 22:23:00
summary:  Bringing up Puppet development environment on Vagrant with the power of tmux
category: development
tags: [ Vagrant, tmux, tmuxinator, atom, puppet]
published: true
---

So, I've decided to actually learn Puppet by developing Puppet modules locally on my workstation. The easiest way to go about that is by running the development environment locally with Vagrant (and Virtualbox).

### Getting started

So here's a list of things that I'm going to perform to get up and running.

- Install and configure tmux
- Install tmuxinator
- Configure project
- Add puppet syntax highlighting to [atom](https://atom.io/)
- Add puppet linting support to atom

### Installing tmux and configure tmuxinator

I'm using a Mac so the easiest method to go for installing is with `brew` with:

```
brew install tmux
```

To install tmuxinator, you can do it with `gem` command:

```
gem install tmuxinator
```

To get the mouse support for tmux and vim style bindings for easier modification, I'm just going to edit `~/.tmux.conf` with the following:

```
# mouse support
set -g mode-mouse on

# vim style bindings
bind-key -t vi-copy v begin-selection
bind-key -t vi-copy y copy-pipe "reattach-to-user-namespace pbcopy"
```

### Configure tmuxinator project

To start with, we need to generate a tmuxinator project with:

```
tmuxinator new puppet-dev.yml
```

Here's the content that I'm going with:

```
# ~/.tmuxinator/puppet-dev.yml

name: puppet-dev
root: ~/puppet-develop

# Specifies (by name or index) which window will be selected on project startup. If not set, the first window is used.
startup_window: puppet

windows:
  - puppet:
      layout: main-vertical
      panes:
        - code:
          - cd ~/puppet-develop
          - atom .
        - puppetmaster:
          - cd ~/puppet-develop/vagrant
          - vagrant up puppet
          - vagrant ssh puppet
          - sudo su
  - agents:
      layout: main-vertical
      panes:
        - agent1:
          - cd ~/puppet-develop/vagrant
          - vagrant up agent1
          - vagrant ssh agent1
        - agent2:
          - cd ~/puppet-develop/vagrant
          - vagrant up agent2
          - vagrant ssh agent2
        - agent3:
          - cd ~/puppet-develop/vagrant
          - vagrant up agent3
```

All of the Vagrant stuff in there are configured with a customized `Vagrantfile` that I've prepared earlier. Maybe I'll do another blog post to detail that process later.

### Add puppet syntax highlighting and puppet linting to atom

This is actually the easiest part. Just search for `language-puppet` & `linter` in [atom packages](https://atom.io/packages) listings and install them.

An additional step would be to install `puppet-lint` gem:

```
gem install puppet-lint
```

And we're all set.

To initiate the project, simply `mux puppet-dev`. One command to rule them all. :)

And of course, materials used are available on [GitHub](https://github.com/hairwire/puppet-develop/tree/module04-tmux).

Later!
