---
name: prime
description: Prime agent with codebase understanding
---

# Prime: Load Project Context

## Objective

Build comprehensive understanding of the codebase by analyzing structure, documentation, and key files.

## Process

### 1. Analyze Project Structure

List all tracked files:
!`git ls-files`

Show directory structure:
On Linux, run: `tree -L 3 -I 'node_modules|__pycache__|.git|dist|build'`

### 2. Read Core Documentation

- Read copilot-instructions.md or similar global rules file
- Read README files at project root and major directories
- Read any architecture documentation
- **Read ALL reference files in `references/` directory** - these contain authoritative project standards, tooling choices, and implementation guidelines that must be followed

### 3. Identify Key Files

Based on the structure, identify and read:
- Main entry points (`apps/telegram_bot/TelegramChatBot.ts`, `apps/whatsapp_bot/WhatsAppChatBot.ts`)
- Core configuration files (`package.json`, `tsconfig.json`)
- Key model/schema definitions (`src/db/src/models/*.ts`)
- Important service files (`src/services/chatbot-service/ChatBotHandlers.ts`, `src/db/src/services/database.service.ts`)

### 4. Understand Current State

Check recent activity:
!`git log -10 --oneline`

Check current branch and status:
!`git status`

## Output Report

Provide a concise summary covering:

### Project Overview
- Purpose and type of application
- Primary technologies and frameworks
- Current version/state

### Architecture
- Overall structure and organization
- Key architectural patterns identified
- Important directories and their purposes

### Tech Stack
- Languages and versions
- Frameworks and major libraries
- Build tools and package managers
- Testing frameworks

### Core Principles
- Code style and conventions observed
- Documentation standards
- Testing approach
- **Reference Standards Applied** (from references/):
  - Package management approach
  - Tooling choices and configurations
  - Implementation guidelines

### Current State
- Active branch
- Recent changes or development focus
- Any immediate observations or concerns

**Make this summary easy to scan - use bullet points and clear headers.**
