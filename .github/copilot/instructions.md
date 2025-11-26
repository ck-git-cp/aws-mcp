# AWS MCP Repository - AI Coding Agent Instructions

This monorepo contains 40+ specialized Model Context Protocol (MCP) servers that provide AI applications with access to AWS services, documentation, and workflows.

## Project Architecture

### Monorepo Structure
- **`src/`**: Each subdirectory is an independent MCP server package (e.g., `src/aws-documentation-mcp-server/`)
- **`testing/`**: Shared integration test framework using official MCP SDK
- **`docusaurus/`**: Documentation site at https://awslabs.github.io/mcp/
- **`samples/`**: Reference implementations and integration examples

### MCP Server Anatomy
Standard structure (see `src/aws-documentation-mcp-server/` as reference):
```
server-name-mcp-server/
├── pyproject.toml              # Package metadata, dependencies, tooling config
├── awslabs/
│   └── server_name_mcp_server/
│       ├── __init__.py         # Version: __version__ = "x.y.z"
│       ├── server.py           # FastMCP instance, tools, resources, main()
│       ├── models.py           # Pydantic data models
│       └── consts.py           # Constants (UPPER_CASE naming)
└── tests/                      # Unit tests + integ tests (test_integ_*.py)
```

### Key Framework: FastMCP
Most servers use `fastmcp` library (not vanilla `mcp`):
```python
from mcp.server.fastmcp import FastMCP, Context
from pydantic import Field

mcp = FastMCP('awslabs.server-name', instructions="...")

@mcp.tool()
async def my_tool(
    ctx: Context,
    param: str = Field(..., description="Always describe params for AI models")
) -> str:
    """Tool docstring appears in MCP protocol."""
    # Implementation
```

## Critical Development Workflows

### Creating a New Server
1. **Generate from template**: `uvx cookiecutter https://github.com/awslabs/mcp.git --checkout cookiecutters --output-dir ./src --directory python`
2. **Update namespace**: Package name is `awslabs.server-name-mcp-server` (hyphens in pyproject.toml, underscores in Python)
3. **Add to README.md**: Update both "Browse by What You're Building" and "Browse by How You're Working" sections
4. **Create docs page**: Add `docusaurus/docs/servers/server-name.mdx` and update `docusaurus/sidebars.ts` and `docusaurus/static/assets/server-cards.json`

### Running & Testing Locally
```bash
# From server directory (e.g., src/aws-documentation-mcp-server/)
uv sync --frozen --all-groups  # Install dependencies
uv run server.py               # Run server (connects via stdio)

# Test with MCP Inspector
npx @modelcontextprotocol/inspector uv --directory /path/to/server run server.py

# Run tests
uv run pytest --cov --cov-branch  # Unit + integration tests
```

### Integration Tests Pattern
Use shared framework in `testing/`:
```python
from testing.pytest_utils import MCPTestBase, create_tool_test_config, TestType

class TestMyServer(MCPTestBase):
    def test_integ_basic(self):
        config = create_tool_test_config(
            test_type=TestType.TOOL,
            tool_name="my_tool",
            arguments={"param": "value"}
        )
        result = self.run_test_config(config)
        assert "expected" in result.content
```

## Project-Specific Conventions

### Naming Patterns
- **Package**: `awslabs.aws-service-mcp-server` (kebab-case)
- **Python module**: `awslabs/aws_service_mcp_server/` (snake_case)
- **Entry point script**: `awslabs.aws-service-mcp-server` in `[project.scripts]`
- **Constants**: `UPPER_SNAKE_CASE` in `consts.py`

### Version Management
- Version stored in **two places**: `pyproject.toml` and `__init__.py:__version__`
- Monorepo CI auto-bumps patch versions on changes
- Commitizen configured but doesn't support remote tagging (see issue #167)

### Pydantic Field Descriptions
Tool parameters must guide AI models explicitly:
```python
@mcp.tool()
async def save_file(
    content: str = Field(..., description="File content to save"),
    workspace_dir: str = Field(
        ...,
        description="""Current workspace directory for saving files.
        CRITICAL: AI assistant must always provide the IDE workspace directory."""
    )
) -> str:
```

### Logging Configuration
All servers use Loguru, controlled by `FASTMCP_LOG_LEVEL` env var:
```python
import sys
from loguru import logger

logger.remove()
logger.add(sys.stderr, level=os.getenv('FASTMCP_LOG_LEVEL', 'WARNING'))
```

### Security Requirements
- **Bandit scans**: Run on all code (configured in pyproject.toml)
- **Pre-commit hooks**: Must pass before commit (`pre-commit install`)
- **License headers**: Required on all source files (Apache 2.0)
- **Secrets detection**: Enabled in CI pipeline

## CI/CD Pipeline

### Automated Checks (`.github/workflows/python.yml`)
1. **Changed package detection**: Only tests modified servers
2. **Per-package jobs**: Build, test, security scan (Bandit), type check (pyright)
3. **Test coverage**: Reported to Codecov (see badge)
4. **Pre-commit**: Runs ruff, pyright, detect-secrets

### Testing Standards
- Unit tests in `tests/` with `pytest`
- Integration tests follow `test_integ_*.py` naming
- Coverage expected to meet/exceed repo average
- Use `testing/` framework utilities, not custom harnesses

## AWS Integration Patterns

### Credential Handling
```python
import boto3

# Standard pattern - respects AWS_PROFILE, AWS_REGION env vars
client = boto3.client('service-name', region_name=os.getenv('AWS_REGION', 'us-east-1'))
```

### Error Handling
Return informative strings from tools (not exceptions):
```python
@mcp.tool()
async def aws_operation(param: str) -> str:
    try:
        # AWS operation
        return json.dumps({"result": "success"})
    except ClientError as e:
        return f"AWS Error: {e.response['Error']['Message']}"
```

## Documentation Requirements

### README Structure (mandatory sections)
1. Quick description and install badges (Cursor + VS Code)
2. Prerequisites (Python version, AWS credentials)
3. Installation instructions (`uvx` command)
4. Configuration examples (environment variables)
5. Available tools/resources with parameter descriptions
6. Example usage patterns

### Docstring Standards
- Tools/resources: Comprehensive docstrings (appear in MCP protocol)
- Include usage requirements and examples
- Document all parameters with `Field()` descriptions

## Common Pitfalls

❌ **Don't**: Create `main.py` - entry point is always `server.py:main()`  
❌ **Don't**: Use relative imports incorrectly - validate path structure  
❌ **Don't**: Skip pre-commit hooks - they're required by CI  
❌ **Don't**: Forget to update README.md when adding servers  
❌ **Don't**: Mix `mcp` and `fastmcp` imports - use FastMCP consistently  
❌ **Don't**: Hardcode paths - use environment variables  

✅ **Do**: Follow existing server patterns (aws-documentation is canonical)  
✅ **Do**: Test with MCP Inspector before committing  
✅ **Do**: Include integration tests using `testing/` framework  
✅ **Do**: Document AI-specific parameter instructions in Field descriptions  
✅ **Do**: Use conventional commit messages (recommended, not enforced)  

## Quick Reference Commands

```bash
# Setup development environment
pre-commit install
uv python install 3.10

# Create new server from template
uvx cookiecutter https://github.com/awslabs/mcp.git --checkout cookiecutters --output-dir ./src --directory python

# Install server dependencies
cd src/my-server-mcp-server && uv sync --frozen --all-groups

# Run all pre-commit checks
pre-commit run --all-files

# Test server locally
npx @modelcontextprotocol/inspector uv --directory $(pwd) run server.py

# Run tests with coverage
uv run pytest --cov --cov-branch --cov-report=term-missing

# Install server for testing in MCP client
uvx awslabs.server-name-mcp-server@latest
```

## Related Documentation

- **Development workflow**: See `DEVELOPER_GUIDE.md` for detailed setup
- **Design patterns**: See `DESIGN_GUIDELINES.md` for comprehensive standards
- **AI development tips**: See `VIBE_CODING_TIPS_TRICKS.md` for AI-assisted coding best practices
- **Contributing**: See `CONTRIBUTING.md` for PR requirements
