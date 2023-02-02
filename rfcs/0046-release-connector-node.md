---
feature: release flow for connector node
authors:
  - "Bohan Zhang"
start_date: "2023/02/02"
---

# Release Workflow for Connector Node

## Summary

We will release the connector node as a standalone Docker image, as well as part of the all-in-one image.

## Motivation

The connector node is in a separate repo, and it's a standalone service but the service should run in any deployment scenario, including playground.

## Design

I propose to integrate the connector node's release flow into Risingwave's.

### Release Workflow

We publish connector node images, including nightly and stable, together with Risingwave's.

* nightly builds: clone the latest code from the master branch, build and publish the image.
* stable builds
  * tag the code in connector node repo with a version
  * tag Risingwave's code with the same version and trigger the pipeline
  * in the pipeline, checkout connector node's code to the tagged version, build and publish the image
* bug fix in connector node: need to manually trigger release workflow in the main repo

### Boot-up

Connector node need to start together with Risingwave, including playground, docker compose and k8s.

* playground: connector node runs as a process in the same container as Risingwave.
* docker compose: connector node will run as a separate container. It will behave like a standalone node.
* k8s: connector node will run as a sidecar deployed in the same pod as other components, such as compute nodes and meta nodes.
