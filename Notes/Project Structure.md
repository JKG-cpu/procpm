## Folder Layout
```bash
procpm/
|--- .github/
	|--- workflows/
		| publish.yaml
		| pages.yaml
|
|--- pyproject.toml
|--- README.md
|--- LICENSE
|
|--- src/
	|--- procpm/
		|--- cli/
		|--- core/
		|--- process/
		|--- config/
		|--- database/
		|--- logging/
		|--- monitoring/
		|--- api/
		|--- tui/
		|--- plugins/
		|--- security
		|--- cli_bundler.py
		|--- main.py
|--- tests/
|--- docs/ # MKDocs
	|--- architecture/
	|--- configuration/
	|--- cli/
	|--- plugins/
	|--- security/
```

This is not the final structure, code / other files will most likely change through this project, this is just a rough sketch.

## Languages + Packages

Python is the choice of language because this project involves lots of:
- Process Management
- File Handling
- Databases
- Networking
- APIs
- Terminal Interfaces
- Configuration
- Automation

Main Packages
- **Typer**: For CLI Commands
- **Rich**: For pretty output / displays
- **Textual**: For TUI
- **FastAPI**: Creates a local API
- **Uvicorn**: Runs the FastAPI server
- **Pydantic**: For models
- **psutil**: For system performance
- **pytest**: For testing
- **pytest-asyncio**: For testing
- **httpx**: For testing

Gonna add **FastAPI**, **Uvicorn**, and **httpx** when I start building the API.

## Development Order
1. Design architecture
2. Configuration system
3. Process Manager
4. Dependency System
5. SQLite database
6. Logging
7. CLI
8. Monitoring
9. TUI
10. API
11. Plugin System
12. Security
13. Testing
14. Packaging + CI
15. Documentation + Polish (Removing random comments, cleaning up code)
