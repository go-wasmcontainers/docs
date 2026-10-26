---
title: "Design"
weight: 25
description: "Design documents for parts of go-wasmcontainers that are not written yet."
tags: [design]
---

A design document decides how a planned part will be built, and why, before
its code exists. Each one lists the references it was written against. Nothing
in this section has shipped.

- [Classroom network]({{< relref "classroom-network" >}}): how students' browsers
  reach each other's containers through a relay and firewall on the teacher's
  laptop, managed through a gRPC control API
