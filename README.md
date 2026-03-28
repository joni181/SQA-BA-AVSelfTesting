# README

This repository holds the artifacts and work results of the Bachelor's thesis "Designing a Self-Testing Capability for AV Software: Feature Diagram and Reference Application" by Jonathan Lippss.

## Feature Diagram

The overall feature diagram, along with the decomposed sub-diagrams, can be found in `figures/feature-diagrams/`.
Apart from the `.svg` files, the directory also contains their `.json` config file, which were used to generate the diagrams with the self-developed feature-diagram creation tool: [draw-feature-diagram](https://github.com/joni181/draw-feature-diagram).

## Reference Application

### Setup

Step-by-step guides for setting up the project and to replicate the simulation and self-test runs can be found in `guides/`. Start with `guides/README.md`.

### Source Code

The implementation of the reference application can be found in the following forked repositories:

- [autoware_universe_self_testing](https://github.com/joni181/autoware_universe_self_testing)
- [autoware_core_self_testing](https://github.com/joni181/autoware_core_self_testing)
- [autoware_launch_self_testing](https://github.com/joni181/autoware_launch_self_testing)

The maps used in the simulation can be found in this repository: [SQA-BA-AVSelfTesting-Maps](https://github.com/joni181/SQA-BA-AVSelfTesting-Maps).