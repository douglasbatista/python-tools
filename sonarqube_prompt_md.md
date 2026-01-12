# Prompt: SonarQube API + LLM Automated Code Fixer

Create a Python tool that integrates SonarQube's Web API with an OpenAI-compatible LLM to automatically analyze code quality issues and generate fixes.

## Requirements

### Core Functionality

#### 1. SonarQube API Integration
- Connect to SonarQube server using authentication token
- Fetch code issues from a specific project using the `/api/issues/search` endpoint
- Support filtering by:
  - Severity levels (BLOCKER, CRITICAL, MAJOR, MINOR, INFO)
  - Issue types (BUG, VULNERABILITY, CODE_SMELL, SECURITY_HOTSPOT)
  - Status (OPEN, CONFIRMED, REOPENED, RESOLVED, CLOSED)
  - **Code age: "New Code" vs "Overall Code"** (using `inNewCodePeriod` parameter)
    - New Code: Issues introduced in recent changes (within the new code period)
    - Overall Code: All issues including legacy code
    - Default to New Code only for CI/CD workflows
- Handle pagination to retrieve all matching issues
- Fetch source code for affected files using `/api/sources/raw` endpoint
- Extract detailed information for each issue including:
  - Issue key, message, rule, severity, type
  - File component path and line number
  - Full source code of the affected file

#### 2. LLM Integration (OpenAI-Compatible)
- Support configurable LLM endpoint (base_url), API key, and model name
- Use the OpenAI Python library for compatibility
- Format SonarQube issues into clear, structured prompts for the LLM
- Include in the prompt:
  - Issue metadata (severity, type, rule, file, line number)
  - Issue description/message from SonarQube
  - Code snippet with context (5 lines before/after the problematic line)
  - Full file source code (truncated if too long)
- Request structured JSON responses from the LLM containing:
  - `explanation`: Brief analysis of the issue and proposed solution
  - `fixed_code`: The corrected code section
  - `confidence`: Confidence level (high/medium/low)
  - `suggested_comment`: Code comment explaining the fix
- Handle JSON parsing with cleanup for potential markdown formatting

#### 3. Code Context Extraction
- Extract relevant code snippets around the issue line
- Mark the problematic line clearly (e.g., with ">>>" marker)
- Include line numbers for context
- Provide configurable context window (default: 5 lines before/after)

#### 4. Batch Processing
- Process multiple issues sequentially
- Display progress (e.g., "[3/10] Processing issue...")
- Track token usage for cost monitoring
- Support limiting the number of issues processed (for testing/cost control)

#### 5. Output Generation
- Save results as JSON with all issue details and LLM-generated fixes
- Generate a markdown report with:
  - Summary statistics (total issues, high-confidence fixes, token usage)
  - Detailed section for each issue showing:
    - Issue metadata and description
    - LLM analysis and confidence level
    - Proposed fix with syntax highlighting
    - Suggested code comments
- Print progress and summary to console

## Technical Specifications

### Class Structure

**Main class: `SonarQubeLLMFixer`**

Constructor parameters:
- `sonar_url`: SonarQube server URL
- `sonar_token`: Authentication token
- `project_key`: Default project key (optional, can specify per-call)
- `llm_api_key`: LLM API key (optional, can use env var)
- `llm_base_url`: LLM endpoint URL (optional, can use env var)
- `llm_model`: Model name (optional, can use env var)

**Class Methods:**
- `from_config(config_path)`: Class method to create instance from YAML config file
- `process_issues_from_config(config_path)`: Process issues using YAML config parameters

**Key Methods:**
- `get_project_issues()`: Fetch issues from SonarQube with filtering (including New Code filter)
- `get_issue_details()`: Get full details for a specific issue including source code
- `format_issue_for_llm()`: Convert SonarQube issue to LLM prompt
- `get_fix_from_llm()`: Send issue to LLM and parse the fix response
- `process_issues()`: Main workflow - fetch, analyze, generate fixes (with only_new_code parameter)
- `generate_report()`: Create markdown report from results
- `_extract_code_snippet()`: Helper to extract code context around issue line

### Dependencies

- `requests` - For SonarQube API calls
- `openai` - For LLM integration (OpenAI-compatible)
- `pyyaml` - For YAML configuration file support
- `json` - For data serialization
- `pathlib` - For file path handling
- `typing` - For type hints

### Authentication

- **SonarQube**: Use token-based auth (token as username, empty password)
- **LLM**: Use OpenAI client with custom base_url for company endpoints

### Error Handling

- Wrap all API calls in try-except blocks
- Handle HTTP errors gracefully with informative messages
- Continue processing other issues if one fails
- Return error information in the fix data structure when LLM fails

### Configuration

- **Primary: Support YAML configuration file** with structure:
  ```yaml
  sonarqube:
    url: "..."
    token: "..."
    project_key: "..."
  llm:
    api_key: "..."
    base_url: "..."
    model: "..."
  processing:
    max_issues: 10
    severities: [BLOCKER, CRITICAL, MAJOR]
    types: [BUG, VULNERABILITY, CODE_SMELL]
    only_new_code: true
    output_file: "sonarqube_fixes.json"
  ```
- Secondary: Support environment variables for credentials:
  - `OPENAI_API_KEY`, `OPENAI_BASE_URL`, `OPENAI_MODEL`
- Tertiary: Allow override via constructor parameters
- Use sensible defaults (e.g., temperature=0.2 for deterministic fixes)

### Best Practices

- Use type hints throughout
- Add docstrings for all public methods
- Keep prompts clear and specific to get structured responses
- Use low temperature (0.2) for more deterministic code fixes
- Limit content sent to LLM to avoid token limits (e.g., first 15,000 chars)
- Include usage examples in the `if __name__ == "__main__"` block

## Example Usage Pattern

### Recommended: Using YAML Configuration

```python
# Initialize from config file
fixer = SonarQubeLLMFixer.from_config("config.yaml")

# Process issues using config settings
results = fixer.process_issues_from_config("config.yaml")

# Generate report
fixer.generate_report(results, 'fix_report.md')
```

### Alternative: Mix of YAML and Code

```python
# Load credentials from YAML
fixer = SonarQubeLLMFixer.from_config("config.yaml")

# Override processing settings in code
results = fixer.process_issues(
    max_issues=20,
    severities=['BLOCKER'],
    only_new_code=False
)
```

### Traditional: Pure Code (No YAML)

```python
# Initialize
fixer = SonarQubeLLMFixer(
    sonar_url="https://sonarqube.company.com",
    sonar_token="squ_abc123...",
    llm_api_key="sk-...",
    llm_base_url="https://company-llm.com/v1",
    llm_model="gpt-4"
)

# Process issues
results = fixer.process_issues(
    project_key="com.company:my-project",
    max_issues=10,
    severities=['BLOCKER', 'CRITICAL'],
    only_new_code=True,  # Focus on New Code issues (default, recommended)
    output_file='fixes.json'
)

# To process ALL issues including legacy code:
# results = fixer.process_issues(
#     project_key="com.company:my-project",
#     only_new_code=False,  # Include Overall Code issues
#     output_file='all_fixes.json'
# )

# Generate report
fixer.generate_report(results, 'fix_report.md')
```

## System Prompts for LLM

### System Prompt

```
You are an expert code reviewer and fixer. Analyze SonarQube issues and provide fixes.

Your response MUST be in JSON format with these keys:
- explanation: Brief explanation of the issue and your fix
- fixed_code: The corrected code (only the relevant section that needs to change)
- confidence: Your confidence level (high/medium/low)
- suggested_comment: A code comment explaining the fix

Be precise and follow best practices for the language being used.
```

## Output Examples

### JSON Output Structure

```json
{
  "project_key": "com.company:my-project",
  "issue_key": "AXa1b2c3d4e5",
  "severity": "CRITICAL",
  "type": "BUG",
  "message": "Remove this null pointer dereference",
  "file": "src/main/java/com/example/Service.java",
  "line": 42,
  "rule": "java:S2259",
  "llm_fix": {
    "explanation": "The code attempts to use 'user' without null check...",
    "fixed_code": "if (user != null) {\n  user.getName();\n}",
    "confidence": "high",
    "suggested_comment": "// Added null check to prevent NPE",
    "tokens_used": 450
  }
}
```

### YAML Configuration Structure

```yaml
# SonarQube LLM Fixer Configuration File

# SonarQube Settings
sonarqube:
  url: "https://sonarqube.your-company.com"
  token: "squ_abc123..."  # Generate from: User > My Account > Security
  project_key: "com.company:my-project"  # Find in SonarQube project settings

# LLM Settings (OpenAI-compatible)
llm:
  api_key: "sk-..."  # Your company's LLM API key
  base_url: "https://your-company-llm.com/v1"  # Your company's LLM endpoint
  model: "gpt-4"  # Model name provided by your company

# Processing Settings
processing:
  # Optional: Override project_key per run (defaults to sonarqube.project_key above)
  # project_key: "com.company:different-project"
  
  # Maximum number of issues to process (start small for testing)
  max_issues: 10
  
  # Severity levels to include (remove any you don't want)
  severities:
    - BLOCKER
    - CRITICAL
    - MAJOR
    # - MINOR
    # - INFO
  
  # Issue types to include
  types:
    - BUG
    - VULNERABILITY
    - CODE_SMELL
    # - SECURITY_HOTSPOT
  
  # Filter for New Code issues only (recommended for CI/CD)
  # Set to false to include legacy "Overall Code" issues
  only_new_code: true
  
  # Output file for results (JSON)
  output_file: "sonarqube_fixes.json"
  
  # Output file for markdown report
  report_file: "fix_report.md"
```

## Additional Notes

- This tool uses the **SonarQube Web API**, NOT web scraping, for reliable and maintainable integration
- The tool should handle SonarQube's pagination automatically to fetch all issues
- Include helpful console output showing progress and summary statistics
- Make the code modular and easy to extend (e.g., adding auto-apply functionality later)
- Focus on code quality: clear variable names, proper error handling, comprehensive docstrings
- The tool should work with any OpenAI-compatible LLM endpoint (company-hosted, local models, etc.)

## Bonus Features (Optional)

If you want to make the tool more advanced, consider adding:

1. Comparison with previous runs to track progress over time
2. Filtering to only process issues introduced in recent commits
3. Integration with Git to create branches and commit fixes automatically
4. Support for dry-run mode that only analyzes without generating fixes
5. Confidence threshold filtering (only save fixes above certain confidence)
6. Cost estimation before processing (based on estimated tokens)
7. Parallel processing of issues for faster execution
8. Support for different output formats (CSV, HTML, etc.)

## Project Setup with uv

### Installation

```bash
# Install uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# Initialize project
uv init
uv add requests openai pyyaml
```

### Project Structure

```
sonarqube-llm-fixer/
├── pyproject.toml
├── uv.lock
├── .python-version
├── config.yaml
├── config.yaml.template
├── .gitignore
├── README.md
└── src/
    └── sonarqube_fixer.py
```

### Running

```bash
# Run directly with uv (no activation needed)
uv run python src/sonarqube_fixer.py

# Or sync and activate
uv sync
source .venv/bin/activate  # Linux/macOS
python src/sonarqube_fixer.py
```

## Implementation Checklist

- [ ] Create `SonarQubeLLMFixer` class with constructor
- [ ] Implement `from_config()` class method for YAML loading
- [ ] Implement `get_project_issues()` with pagination and filtering
- [ ] Implement `get_issue_details()` to fetch source code
- [ ] Implement `format_issue_for_llm()` to create prompts
- [ ] Implement `_extract_code_snippet()` helper
- [ ] Implement `get_fix_from_llm()` with JSON parsing
- [ ] Implement `process_issues()` main workflow
- [ ] Implement `process_issues_from_config()` for YAML-based processing
- [ ] Implement `generate_report()` for markdown output
- [ ] Add comprehensive error handling
- [ ] Add type hints to all methods
- [ ] Add docstrings to all public methods
- [ ] Create example usage in `if __name__ == "__main__"`
- [ ] Test with actual SonarQube instance
- [ ] Create `config.yaml.template` example
- [ ] Create `.gitignore` file
- [ ] Document in README.md

---

## Usage Instructions for This Prompt

Copy this entire markdown file and paste it into your company's LLM with the instruction:

> "Please implement this Python tool following all the specifications. Make sure the code is production-ready with proper error handling, type hints, comprehensive docstrings, and includes the YAML configuration support."

If the LLM's response is incomplete, follow up with:

> "Please continue from where you left off"

For modifications:

> "Please update the code to also include [your specific requirement]"
