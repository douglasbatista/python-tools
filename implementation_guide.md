# SonarQube LLM Fixer - Implementation Guide

## Overview

This guide explains how to implement the modular SonarQube LLM Fixer tool. The tool is split into 10 separate Python modules for maintainability, testability, and scalability.

## Files Provided

### ✅ Complete and Ready to Use

1. **models.py** - Data models (Issue, IssueFix, filters, enums)
2. **config.py** - Configuration management with YAML support
3. **sonarqube_client.py** - SonarQube API client
4. **llm_client.py** - LLM integration client  
5. **code_formatter.py** - Code formatting utilities
6. **cli.py** - Command-line interface

### 🔨 Need to be Created

7. **issue_processor.py** - Main processing orchestration
8. **report_generator.py** - Markdown report generation
9. **__init__.py** - Package initialization (simple)
10. **__main__.py** - Entry point (simple)

## Step-by-Step Setup

### 1. Create Project Structure

```bash
# Create directories
mkdir -p sonarqube-llm-fixer/src/sonarqube_fixer
cd sonarqube-llm-fixer

# Initialize with uv
uv init
uv add requests openai pyyaml
```

### 2. Add the Provided Files

Copy these files into `src/sonarqube_fixer/`:
- `models.py` (from artifacts)
- `config.py` (from artifacts)
- `sonarqube_client.py` (from artifacts)
- `llm_client.py` (from artifacts)
- `code_formatter.py` (from artifacts)
- `cli.py` (from artifacts)

### 3. Create `__init__.py`

```python
"""SonarQube LLM Fixer - Automated code issue fixing using LLM"""

__version__ = "0.1.0"

from .config import Config
from .issue_processor import IssueProcessor

__all__ = ["Config", "IssueProcessor"]
```

### 4. Create `__main__.py`

```python
"""Entry point for running the package as a module"""

from .cli import main

if __name__ == "__main__":
    main()
```

### 5. Create `issue_processor.py`

This is the main orchestration module. Key requirements:

```python
"""Main issue processing orchestration"""
from typing import List
import json
from .models import Issue, IssueFix, ProcessingResult, IssueFilter, ProcessingSummary
from .sonarqube_client import SonarQubeClient
from .llm_client import LLMClient
from .code_formatter import format_for_llm
from .config import ProcessingConfig


class IssueProcessor:
    """Orchestrates the issue fixing workflow"""
    
    def __init__(
        self,
        sonarqube_client: SonarQubeClient,
        llm_client: LLMClient,
        config: ProcessingConfig
    ):
        self.sonarqube = sonarqube_client
        self.llm = llm_client
        self.config = config
        self.results: List[ProcessingResult] = []
    
    def process_issues(self, issue_filter: IssueFilter) -> List[ProcessingResult]:
        """
        Main processing workflow
        
        1. Fetch issues from SonarQube
        2. For each issue:
           - Get source code
           - Format context for LLM
           - Generate fix
           - Track results
        3. Return all results
        """
        # Implementation here
        pass
    
    def process_single_issue(self, issue: Issue) -> ProcessingResult:
        """Process a single issue"""
        # Implementation here
        pass
    
    def save_results(self, results: List[ProcessingResult], output_file: str):
        """Save results to JSON file"""
        # Implementation here
        pass
    
    def get_summary(self) -> ProcessingSummary:
        """Get processing summary statistics"""
        return ProcessingSummary.from_results(self.results)
```

**Implementation hints:**
- Fetch issues using `self.sonarqube.get_issues(issue_filter)`
- For each issue, get details with `self.sonarqube.get_issue_details(issue.key)`
- Format context with `format_for_llm(issue, self.config.context_lines)`
- Generate fix with `self.llm.generate_fix(issue, context)`
- Track progress with print statements
- Handle errors gracefully and continue processing

### 6. Create `report_generator.py`

This generates markdown reports. Key requirements:

```python
"""Generate markdown reports from processing results"""
from typing import List
from .models import ProcessingResult, ProcessingSummary


class ReportGenerator:
    """Generates markdown reports"""
    
    def generate_markdown(
        self,
        results: List[ProcessingResult],
        output_file: str
    ):
        """
        Generate comprehensive markdown report
        
        Structure:
        - Title and summary
        - Statistics section
        - Individual issue sections
        """
        # Implementation here
        pass
    
    def generate_summary(self, results: List[ProcessingResult]) -> str:
        """Generate summary section"""
        # Implementation here
        pass
    
    def _format_issue_section(
        self,
        result: ProcessingResult,
        index: int
    ) -> str:
        """Format a single issue section"""
        # Implementation here
        pass
```

**Implementation hints:**
- Start with header and overall statistics
- For each issue, create a section with:
  - Issue metadata (severity, type, file, line)
  - Issue message
  - LLM explanation
  - Proposed fix in code block
  - Confidence level
- Use markdown formatting for readability
- Add horizontal rules between sections

### 7. Create Configuration File

Create `config.yaml.template`:

```yaml
# SonarQube LLM Fixer Configuration

sonarqube:
  url: "https://sonarqube.your-company.com"
  token: "YOUR_SONAR_TOKEN"
  project_key: "com.company:your-project"

llm:
  api_key: "YOUR_LLM_API_KEY"
  base_url: "https://your-llm-endpoint.com/v1"
  model: "gpt-4"
  temperature: 0.2
  max_tokens: 4000

processing:
  max_issues: 10
  severities:
    - BLOCKER
    - CRITICAL
    - MAJOR
  types:
    - BUG
    - VULNERABILITY
    - CODE_SMELL
  only_new_code: true
  output_file: "sonarqube_fixes.json"
  report_file: "fix_report.md"
  context_lines: 5
  max_content_chars: 15000
```

Then create your actual `config.yaml` with real credentials.

### 8. Create .gitignore

```
# Python
__pycache__/
*.py[cod]
*$py.class
.Python
.venv/
venv/
*.egg-info/

# uv
.uv/

# Configuration (contains secrets)
config.yaml

# Output files
sonarqube_fixes.json
fix_report.md
*.log

# IDE
.vscode/
.idea/
*.swp
```

### 9. Test the Tool

```bash
# Test connections
uv run python -m sonarqube_fixer --config config.yaml --test-connection

# Process issues (start with small number)
uv run python -m sonarqube_fixer --config config.yaml --max-issues 3 --verbose

# Full run
uv run python -m sonarqube_fixer --config config.yaml
```

## Module Dependencies

```
cli.py
├── config.py
├── sonarqube_client.py
│   └── models.py
├── llm_client.py
│   └── models.py
├── issue_processor.py
│   ├── sonarqube_client.py
│   ├── llm_client.py
│   ├── code_formatter.py
│   └── models.py
└── report_generator.py
    └── models.py
```

## Testing Strategy

### Unit Tests

Create tests for each module:

```bash
tests/
├── test_config.py          # Test configuration loading
├── test_models.py          # Test data models
├── test_sonarqube_client.py # Mock SonarQube API
├── test_llm_client.py      # Mock LLM API
├── test_code_formatter.py  # Test formatting functions
├── test_issue_processor.py # Test orchestration
└── test_report_generator.py # Test report generation
```

Run tests:
```bash
uv add --dev pytest
uv run pytest
```

### Integration Tests

Test with actual SonarQube and LLM:
```bash
# Small test run
uv run python -m sonarqube_fixer --config config.yaml --max-issues 1

# Verify output files exist
ls -l sonarqube_fixes.json fix_report.md
```

## Common Issues and Solutions

### Issue: Configuration validation fails
**Solution**: Check `config.yaml` syntax and ensure all required fields are present

### Issue: SonarQube connection fails
**Solution**: Verify URL and token, test with `--test-connection`

### Issue: LLM returns invalid JSON
**Solution**: Check `llm_client.py` JSON cleaning logic, ensure proper system prompt

### Issue: Source code not fetched
**Solution**: Verify component key format and permissions in SonarQube

### Issue: Out of memory with large files
**Solution**: Adjust `max_content_chars` in config to truncate large files

## Extending the Tool

### Add New Issue Types

Edit `models.py`:
```python
class IssueType(str, Enum):
    BUG = "BUG"
    VULNERABILITY = "VULNERABILITY"
    CODE_SMELL = "CODE_SMELL"
    SECURITY_HOTSPOT = "SECURITY_HOTSPOT"
    YOUR_NEW_TYPE = "YOUR_NEW_TYPE"  # Add here
```

### Add New LLM Provider

Create new client inheriting from base:
```python
# llm_client_custom.py
from .llm_client import LLMClient

class CustomLLMClient(LLMClient):
    def generate_fix(self, issue, code_context):
        # Custom implementation
        pass
```

### Add Auto-Apply Feature

Extend `issue_processor.py`:
```python
def apply_fix(self, result: ProcessingResult):
    """Apply fix to source code"""
    if result.fix.confidence == 'high':
        # Write to file
        # Create git commit
        pass
```

## Production Deployment

### Docker Container

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY . .
RUN pip install uv && uv sync
CMD ["uv", "run", "python", "-m", "sonarqube_fixer", "--config", "config.yaml"]
```

### CI/CD Integration

See `uv_project_setup` artifact for GitHub Actions and GitLab CI examples.

## Support and Contribution

For issues or questions:
1. Check the implementation guide
2. Review module docstrings
3. Run with `--verbose` for debugging
4. Check log files

## License and Credits

This tool integrates:
- SonarQube Web API
- OpenAI-compatible LLM APIs
- Python type hints and dataclasses
- uv for dependency management

---

**Next Steps:**
1. Create the two remaining files (`issue_processor.py` and `report_generator.py`)
2. Test with your SonarQube instance
3. Iterate and improve based on results
4. Share with your team!
