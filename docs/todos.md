**Todos**

1. **Migrate to local KiCad**: Refactor codebase to migrate to a local KiCad installation instead of Docker while keeping existing functionalities intact following the newly made plan.

2. **Use Context7 MCP for docs**: Leverage `context7` MCP server for documentation gathering rather than our current custom documentation solution.

3. **Hallucination detection as tool**: Make hallucination detection a `@function_tool` feature (exposed to agents) instead of it being part of a separate MCP server.

4. **Update OpenAI Agents SDK**: Update the codebase to the latest OpenAI Agents SDK version and adjust usages/APIs accordingly.

5. **Adopt Sessions for memory**: Use `sessions` (newly introduced in the Agents SDK) to manage memory and accumulated context across agents and runs.

Notes:
- Refer to `circuitron/agents.py`, `circuitron/tools.py`, `circuitron/mcp_manager.py`, and `circuitron/docker_session.py` for likely changes.

