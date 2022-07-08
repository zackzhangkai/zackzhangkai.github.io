---
layout: post
published: true
title: 基于github-action高效构建CICD之Devops工作流
categories: [document]
tags: [cicd,devops]
---
* content
{:toc}

## 前言

使用Github Actions，当代码Push到仓库后，直接通过Azure主机来触发ci构建，将镜像推到DockerHub。自动完成CI之后，发果要发步到环境，只需去环境上更新下Deployment的镜像，即可以快速验证代码，极大的提高了开发效率。

## 搭建

[官网代码示例](https://docs.github.com/cn/free-pro-team@latest/actions)

[ssh模块](https://github.com/appleboy/ssh-action
)

[基于KubeSphere自动完成ci工作](https://kubesphere.com.cn/forum/d/2971-github-actioncikubesphere)
