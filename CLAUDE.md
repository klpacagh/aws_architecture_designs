# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Purpose

This repository contains AWS architecture designs and diagrams. It uses MCP-connected tools for generating architecture diagrams and managing AWS data processing services.

## Available MCP Tools

### Diagram Generation (aws-diagram)
Generates architecture diagrams using the Python `diagrams` package. Workflow:
1. `list_icons` — discover available icons (providers, services, icons)
2. `get_diagram_examples` — get example code for diagram types (aws, sequence, flow, class, k8s, onprem, custom)
<!-- 3. `generate_diagram` — submit Python code using the diagrams DSL to produce PNG output; always pass `workspace_dir` to save in the user's current directory -->

### AWS Data Processing (aws-dataprocessing)
Manages AWS Glue, EMR, Athena, S3, and IAM resources. Runs in read-only mode by default (`--allow-write` for write operations, `--allow-sensitive-data-access` for sensitive data).

Key services: Glue Data Catalog (databases, tables, crawlers, jobs, workflows, triggers, sessions), EMR clusters (EC2 and Serverless), Athena queries and workgroups, S3 bucket management, IAM role management for data processing.
