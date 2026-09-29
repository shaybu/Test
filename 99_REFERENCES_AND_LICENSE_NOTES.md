# References and License Notes

These notes capture technology facts verified while preparing this architecture.

They are not legal advice. Re-check exact dependency versions and licenses before production adoption.

## LangGraph

- Repository: `langchain-ai/langgraph`
- Purpose: low-level orchestration for long-running, stateful agents/workflows
- Current Python package metadata identifies the license as MIT.
- It can be used without adopting high-level LangChain agent abstractions.

## Pydantic

- Repository: `pydantic/pydantic`
- Purpose: data validation using Python type hints
- License: MIT

## Pydantic Settings

- Repository: `pydantic/pydantic-settings`
- Purpose: validated settings/configuration management
- License: MIT

## Graphify

- Repository: `Graphify-Labs/graphify`
- Purpose: queryable graph generated from code/docs/configuration and other repository content
- Supports an MCP server with graph query operations.
- Current repository notice identifies Apache License 2.0, with older MIT-licensed contributions retained under their original terms.
- Current MCP tooling includes graph query/node/neighbor/path access and related repository-analysis operations.

## OpenAI Agents SDK

- Repository: `openai/openai-agents-python`
- License: MIT
- Not required by the recommended v1 architecture.
- If evaluated later, validate compatibility with the internal/local model gateway and confirm that adding it solves a real problem rather than duplicating LangGraph orchestration.
