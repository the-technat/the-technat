+++
title = "local-kubernetes"
date = "2023-01-25"
+++

# Intro

Note: WIP on this Guide ;)

In the last couple of months I was working a lot with argocd, testing different configs, contributing and so on. Do to such things fast and efficient, a local kubernetes cluster in docker containers helps. So this guide shows you how to set this up

## Setup

As said, this is WIP

## Semi-automated setup

You want such a setup to be reproducable, so that's why I wrote some shell functions to do this over and over again.

If you have the following tools installed, [this](https://github.com/the-technat/WALL-E/blob/master/zsh/.zsh-custom/plugins/argocd/argocd.plugin.zsh) will help you provision your local cluster and argocd for testing:

- helm
- kubectl
- docker
- k3d
