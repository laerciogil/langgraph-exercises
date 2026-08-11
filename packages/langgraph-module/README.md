# langgraph-module

`langgraph-module` is the workspace distribution containing the `langgraph_module`
Python package and its multi-agent supervisor example. The distribution name uses
hyphens for uv commands; the import name uses underscores.

## Workspace setup

From the repository root, install the workspace with:

```bash
uv sync
```

The package is resolved as a local workspace member and can be selected explicitly:

```bash
uv run --package langgraph-module python -c "import langgraph_module"
```

## Running the supervisor

After configuring the required provider credentials in your environment, run:

```bash
uv run --package langgraph-module python -m langgraph_module.multi_agent.supervisor.main
```

The supervisor uses `OPENAI_API_KEY` for the OpenAI chat model and
`TAVILY_API_KEY` for web search and extraction. Do not put credentials in source
files or commit them to the repository. Import and resource checks do not make
model or web API calls.
