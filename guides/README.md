# Guides

This folder contains the guides required to go from an empty workspace to a running runtime self-testing demonstration.

## Start Here

If you are starting from an empty workspace, follow the guides in this order:

1. [Autoware setup](./01_autoware_setup.md)
2. [Demonstration reproduction](./02_demo_reproduction.md)

Use [Troubleshooting](./03_troubleshooting.md) whenever a step fails.

If you need browser-based visualization or a remote Foxglove bridge, also see [Optional frontend setup](./04_optional_frontend_setup.md).

## What Each Guide Covers

- [Autoware setup](./01_autoware_setup.md)
  Takes you from an empty workspace to a built Autoware environment with the modified repositories and the required packages for the demonstration.

- [Demonstration reproduction](./02_demo_reproduction.md)
  Assumes the setup is already complete and describes how to launch the simulation, set the route, activate the selected module, and reproduce the self-test runs.

- [Troubleshooting](./03_troubleshooting.md)
  Collects known issues and fixes from the original setup and execution process.

- [Optional frontend setup](./04_optional_frontend_setup.md)
  Describes the Foxglove bridge setup and related notes for environment-specific remote visualization.

## Repository Assumptions

The guides assume that the `SQA-BA-AVSelfTesting` repository README links to:

- the demonstration-ready forks of the required Autoware repositories,
- the map repository used for the demonstration,
- and the relevant branches or commit references.

Where the exact fork URLs or branch names are not repeated in the guides, use the repository README as the source of truth.
