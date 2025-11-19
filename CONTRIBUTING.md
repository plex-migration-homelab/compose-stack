# Contributing to Docker Compose Homelab Stack

Thank you for your interest in contributing to this project! This guide will help you get started.

## How to Contribute

### Reporting Issues

If you encounter a bug or have a feature request:

1. Check if the issue already exists in the [GitHub Issues](https://github.com/plex-migration-homelab/compose-stack/issues)
2. If not, create a new issue with:
   - Clear title and description
   - Steps to reproduce (for bugs)
   - Expected vs actual behavior
   - Your environment details (OS, Docker version, etc.)

### Submitting Changes

1. **Fork the Repository**
   ```bash
   git clone https://github.com/plex-migration-homelab/compose-stack.git
   cd compose-stack
   ```

2. **Create a Feature Branch**
   ```bash
   git checkout -b feature/your-feature-name
   ```

3. **Make Your Changes**
   - Follow the existing code structure
   - Test your changes thoroughly
   - Update documentation as needed

4. **Commit Your Changes**
   ```bash
   git add .
   git commit -m "Description of your changes"
   ```

5. **Push to Your Fork**
   ```bash
   git push origin feature/your-feature-name
   ```

6. **Create a Pull Request**
   - Go to the original repository
   - Click "New Pull Request"
   - Select your feature branch
   - Provide a clear description of your changes

## Development Guidelines

### Docker Compose Standards

- Use `compose.yml` as the primary filename (with `docker-compose.yml` as a symlink for compatibility)
- Follow Docker Compose best practices
- Use environment variables for configuration
- Include comments for complex configurations

### Service Configuration

- Use LinuxServer.io images when available
- Configure proper PUID/PGID support
- Set appropriate restart policies (`unless-stopped`)
- Use meaningful container names
- Define proper network configurations

### Environment Variables

- Never commit `.env` files
- Always provide `.env.example` with placeholders
- Document all required variables
- Use sensible defaults where possible

### Documentation Standards

- Update README files when adding new services
- Include port numbers and access URLs
- Provide initial setup instructions
- Add troubleshooting tips for common issues

### Testing Your Changes

Before submitting a pull request:

1. **Test the Stack**
   ```bash
   cd /srv/containers/[stack-name]
   docker compose up -d
   docker compose ps
   docker compose logs
   ```

2. **Verify Services**
   - Check that all services start successfully
   - Test basic functionality
   - Verify port accessibility

3. **Check Documentation**
   - Ensure all documentation is updated
   - Verify commands work as documented
   - Check for typos and formatting issues

## Code of Conduct

### Our Standards

- Be respectful and inclusive
- Welcome newcomers
- Accept constructive criticism gracefully
- Focus on what's best for the community

### Unacceptable Behavior

- Harassment or discrimination
- Trolling or inflammatory comments
- Publishing others' private information
- Any conduct inappropriate in a professional setting

## Questions?

If you have questions about contributing, feel free to:
- Open a GitHub Discussion
- Create an issue with the "question" label
- Reach out to the maintainers

## License

By contributing, you agree that your contributions will be licensed under the same license as the project.
