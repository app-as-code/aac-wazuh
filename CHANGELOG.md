# Changelog

All notable changes to this deployment definition are documented in this file.
The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

## [0.0.1]

### Added

- App-as-Code service definition for a single-node Wazuh server stack (dashboard, manager, indexer).
- One-shot certificate generator for the internal TLS certificate set.
- Dashboard routing through a reverse proxy, and REST API routing on a separate hostname.
- Agent (1514 and 1515 TCP) and syslog (514 TCP and UDP) ports published on the host.
- README with configuration reference, and project icon.
