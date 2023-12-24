+++
title = "Epehmeral Kubernetes"
date = "2023-12-24"
+++

## Intro

As a Kubernetes Engineer I sometimes need to test something that requires a fresh Kubernetes cluster. So this doc lists some options I use to accomplish this goal.

## Grapes

[grapes](https://github.com/the-technat/grapes) is a repository of mine that contains configuration and github actions to quickly deploy various flavors of Kubernetes.

## L8s

L8s is my approach to install K8s using Docker containers. The approach is not new, nor are the tools I'm using.

The thing I just did, was to create a way to quickly recreate such a cluster using existing tooling.

The result: https://github.com/the-technat/dotfiles/tree/master/dot_omz-custom/plugins/l8s.
