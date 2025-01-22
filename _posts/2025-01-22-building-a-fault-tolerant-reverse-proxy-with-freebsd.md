---
layout: post
title: "Building a fault-tolerant reverse proxy with FreeBSD"
date: 2025-01-22
tags: FreeBSD HAProxy CARP Reverse_Proxy
---

# Building a fault-tolerant reverse proxy with FreeBSD

### Goal

The goal of this blog post is to set up HAProxy as a reverse proxy on two FreeBSD hosts with CARP for fault tolerance, so that you can still access services if one of them fails.

