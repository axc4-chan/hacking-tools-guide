# 🤝 Contributing

Thanks for your interest in improving this guide!

## How to Contribute

1. **Fork** the repository
2. **Create a branch** for your change:
   ```bash
   git checkout -b feature/add-tool-name
   ```
3. **Make your changes** — see guidelines below
4. **Test** any commands you add in a lab environment
5. **Open a Pull Request** with a clear description

## Guidelines

### Adding a New Tool
Add tools to the matching category file (`01`–`07`). Each tool entry should include:

1. **Description** — what the tool does and when to use it
2. **Installation** — one or two supported methods (apt preferred for Kali)
3. **Common usage** — copy-pasteable commands with realistic targets
4. **Key flags/commands table** — the options people actually use
5. **Resources** — official docs/GitHub link

### Style Rules
- Use fenced code blocks with language hints (`bash`, `yaml`, etc.)
- Keep the existing emoji heading style per section
- One tool per `##` heading; use `---` separators between tools
- Update `README.md` TOC if adding/removing files
- Prefer accuracy over exhaustiveness — untested commands are worse than fewer commands

### Reporting Issues
- Broken commands, outdated flags, or version changes → open an issue with the error output
- Wrong information → cite the official source in your issue/PR

## Code of Conduct

Be respectful. This is educational material — contributions must not include
content targeting unauthorized systems or illegal activity.