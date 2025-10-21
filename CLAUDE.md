# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the Wallet-Service repository, which appears to be in its initial setup phase. The repository is currently empty except for git configuration files.

## Repository Status

This repository has been initialized but does not yet contain application code, build configuration, or dependency management files. When developing in this codebase, you will need to establish:

- Programming language and framework selection
- Project structure and directory organization
- Build and dependency management configuration
- Testing framework setup
- Development workflow and tooling

## Security Considerations

Given the "Wallet-Service" naming, this project likely involves handling sensitive financial or cryptocurrency operations. When code is added:

- Never commit private keys, mnemonics, API keys, or credentials
- Implement proper secret management from the start
- Follow secure coding practices for financial applications
- Ensure proper input validation and sanitization
- Implement comprehensive audit logging for all wallet operations

## Next Steps for Repository Setup

When beginning development, typical setup tasks include:

1. Initialize package manager (e.g., `package.json`, `go.mod`, `pom.xml`, `requirements.txt`)
2. Set up project directory structure
3. Configure build tools and linters
4. Add testing framework
5. Create CI/CD pipeline configuration
6. Document setup and development commands in README.md
