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

## CRITICAL: Company LLM Endpoint Requirement

**IMPORTANT**: The user's company **blocks all requests to openai.com**. This tool MUST be configured to use the company's internal OpenAI-compatible LLM endpoint.

### Network Security Requirements

**CRITICAL SECURITY CONSTRAINT**: The tool must NEVER attempt to contact any external URLs, especially openai.com or api.openai.com. All LLM requests must go exclusively to the company's internal endpoint.

### Configuration Requirements

1. **No OpenAI.com defaults**: Never default to `https://api.openai.com/v1`
2. **Validate endpoint**: Add validation to reject openai.com URLs
3. **Require explicit base_url**: Force users to specify their company's endpoint
4. **Clear error messages**: If openai.com detected, show helpful error
5. **Multiple validation layers**: Check at config load, client init, and runtime
6. **Disable telemetry**: Prevent any analytics or update checks
7. **Explicit logging**: Always log which endpoint is being used

### Implementation in config.py

```python
@dataclass
class LLMConfig:
    """LLM connection configuration"""
    api_key: str
    base_url: str  # No default - must be explicitly provided
    model: str = "gpt-4"
    
    def validate(self) -> None:
        """Validate LLM configuration"""
        if not self.base_url:
            raise ValueError(
                "LLM base_url is required. Specify your company's LLM endpoint. "
                "Note: OpenAI.com endpoints are blocked by your company."
            )
        
        # CRITICAL: Prevent accidental use of blocked OpenAI endpoints
        if 'openai.com' in self.base_url.lower():
            raise ValueError(
                "OpenAI endpoints are blocked by your company. "
                "Use your company's internal LLM endpoint instead."
            )
    
    @classmethod
    def from_dict(cls, data: Dict[str, Any]) -> 'LLMConfig':
        """Create from dictionary"""
        base_url = data.get('base_url')
        if not base_url:
            raise ValueError(
                "LLM base_url is required. Specify your company's LLM endpoint. "
                "Note: OpenAI.com endpoints are blocked by your company."
            )
        return cls(
            api_key=data.get('api_key', ''),
            base_url=base_url,
            model=data.get('model', 'gpt-4')
        )
```

### Implementation in llm_client.py

Add multiple safety checks:

```python
import os

class LLMClient:
    def __init__(self, api_key: str, base_url: str, model: str, ...):
        # CRITICAL: Verify we're not using OpenAI
        if 'openai.com' in base_url.lower():
            raise ValueError(
                "BLOCKED: OpenAI.com endpoints are not allowed. "
                f"Attempted to use: {base_url}\n"
                "Use your company's internal LLM endpoint instead."
            )
        
        # Disable telemetry
        os.environ['OPENAI_LOG_LEVEL'] = 'error'
        
        # Initialize client with ONLY the specified endpoint
        self.client = OpenAI(
            api_key=api_key,
            base_url=base_url,
            max_retries=2,
            timeout=60.0
        )
        
        # Log for transparency
        print(f"LLM Client initialized:")
        print(f"  Endpoint: {base_url}")
        print(f"  Model: {model}")
        print(f"  (OpenAI.com access is BLOCKED)")
```

### Alternative: Pure Requests Implementation

For maximum security and zero external dependencies, provide an alternative implementation using only the `requests` library:

**File: `llm_client_no_openai.py`**

This version:
- Does NOT use the OpenAI library at all
- Uses ONLY Python's `requests` library
- Makes direct HTTP POST to the company endpoint
- Gives complete control over all network requests
- Is a drop-in replacement for the standard client

```python
import requests
import json

class LLMClientPureRequests:
    """LLM client using only requests library (no OpenAI dependency)"""
    
    def __init__(self, api_key, base_url, model, temperature=0.2, max_tokens=4000):
        if 'openai.com' in base_url.lower():
            raise ValueError("OpenAI.com endpoints are blocked")
        
        self.base_url = base_url.rstrip('/')
        self.api_key = api_key
        self.model = model
        
        self.session = requests.Session()
        self.session.headers.update({
            'Content-Type': 'application/json',
            'Authorization': f'Bearer {api_key}'
        })
    
    def generate_fix(self, issue, code_context):
        url = f"{self.base_url}/chat/completions"
        
        payload = {
            "model": self.model,
            "messages": [
                {"role": "system", "content": SYSTEM_PROMPT},
                {"role": "user", "content": prompt}
            ],
            "temperature": self.temperature,
            "max_tokens": self.max_tokens
        }
        
        response = self.session.post(url, json=payload, timeout=60)
        response.raise_for_status()
        
        data = response.json()
        response_text = data['choices'][0]['message']['content']
        # Parse and return fix...
```

### Example Company Endpoints

The tool should work with any of these OpenAI-compatible endpoints:
- **Azure OpenAI**: `https://resource.openai.azure.com/openai/deployments/name/v1`
- **AWS Bedrock** (with proxy): `https://bedrock.company.com/v1`
- **Self-hosted vLLM**: `http://internal-llm:8000/v1`
- **LiteLLM Gateway**: `https://litellm.company.com/v1`
- **Custom company endpoint**: `https://ai.company.com/v1`
- **Ollama**: `http://localhost:11434/v1`

### Verification & Testing

Include a connection test function that explicitly shows which endpoint is being contacted:

```python
def test_connection(self) -> bool:
    """Test connection to LLM service"""
    try:
        print(f"Testing connection to: {self.base_url}")
        
        response = self.client.chat.completions.create(
            model=self.model,
            messages=[{"role": "user", "content": "Hello"}],
            max_tokens=10
        )
        
        if response.choices:
            print(f"✅ LLM connection successful (Endpoint: {self.base_url})")
            return True
    except Exception as e:
        print(f"❌ Connection failed: {e}")
        print(f"   Attempted endpoint: {self.base_url}")
        return False
```

### Configuration File Template

The config.yaml template must clearly indicate company endpoint requirement:

```yaml
# LLM Settings (Company Internal Endpoint - OpenAI.com is BLOCKED)
llm:
  # IMPORTANT: Use your company's internal LLM endpoint
  # OpenAI.com endpoints are blocked by the company firewall
  api_key: "your-company-llm-api-key"
  base_url: "https://llm.your-company.com/v1"  # REQUIRED - Your company's endpoint
  model: "gpt-4"  # Model name provided by your company
  
  # SSL Certificate Verification (if using self-signed or internal CA)
  # Specify path to CA bundle file (.cer, .crt, or .pem)
  # ca_bundle_path: "certs/ca-bundle.crt"
  # Or using caBundlePath (both work):
  # caBundlePath: "/path/to/company-ca-bundle.cer"
```

### Documentation Requirements

Create these additional documentation files:

1. **NETWORK_SECURITY.md**: Explain how to verify no external URLs are contacted
   - Network monitoring methods (tcpdump, Wireshark)
   - Firewall testing procedures
   - Proxy inspection with mitmproxy
   - Offline testing methods

2. **COMPANY_LLM_SETUP.md**: Guide for obtaining company LLM details
   - How to get endpoint URL from IT
   - Common endpoint formats
   - Authentication methods
   - Troubleshooting company-specific issues

3. **QUICK_START.md**: Include prominent warnings about OpenAI blocking

### Environment Variables

Support alternative environment variable names:

```python
# Support both generic and OpenAI-compatible names
llm_base_url = (
    os.environ.get('LLM_BASE_URL') or 
    os.environ.get('OPENAI_BASE_URL')
)

llm_api_key = (
    os.environ.get('LLM_API_KEY') or 
    os.environ.get('OPENAI_API_KEY', '')
)

# But always validate the URL
if llm_base_url and 'openai.com' in llm_base_url.lower():
    raise ValueError("OpenAI.com endpoints are blocked")
```

### Security Checklist for Implementation

- [ ] No default to api.openai.com anywhere in code
- [ ] Validation checks for openai.com in base_url
- [ ] Multiple validation layers (config, client init, runtime)
- [ ] Telemetry and analytics disabled
- [ ] Explicit endpoint logging on every request
- [ ] Alternative pure-requests implementation provided
- [ ] Network verification documentation included
- [ ] Clear error messages when OpenAI detected
- [ ] Support for company-specific endpoint formats
- [ ] Test connection function shows endpoint used

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
- `httpx` - Required by OpenAI library for custom SSL certificates
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

## Project Structure - Modular Design

The tool should be split into separate, maintainable modules:

```
sonarqube-llm-fixer/
├── src/
│   └── sonarqube_fixer/
│       ├── __init__.py           # Package initialization
│       ├── __main__.py           # Entry point
│       ├── cli.py                # Command-line interface
│       ├── config.py             # Configuration management
│       ├── models.py             # Data models
│       ├── sonarqube_client.py   # SonarQube API client
│       ├── llm_client.py         # LLM integration
│       ├── issue_processor.py    # Main processing logic
│       ├── code_formatter.py     # Code formatting utilities
│       └── report_generator.py   # Report generation
└── tests/                        # Unit tests
```

### Module Responsibilities

**models.py**: Define all data structures
- `Issue`: SonarQube issue representation
- `IssueFix`: LLM-generated fix
- `ProcessingResult`: Combined issue + fix result
- `IssueFilter`: Filter criteria for issues
- `ProcessingSummary`: Processing statistics
- Enums for Severity, IssueType, IssueStatus, Confidence

**config.py**: Configuration management
- `Config`: Main configuration class
- `SonarQubeConfig`: SonarQube settings
- `LLMConfig`: LLM settings  
- `ProcessingConfig`: Processing parameters
- Methods: `from_yaml()`, `from_dict()`, `from_env()`, `validate()`
- Support environment variable substitution in YAML (${VAR_NAME})

**sonarqube_client.py**: SonarQube API client
- `SonarQubeClient`: Main client class
- Methods:
  - `get_issues()`: Fetch issues with pagination
  - `get_issue_details()`: Get detailed issue info
  - `get_source_code()`: Fetch source code
  - `test_connection()`: Test SonarQube connectivity
- Handle authentication and session management
- Automatic pagination handling

**llm_client.py**: LLM integration
- `LLMClient`: Main LLM client class
- Methods:
  - `generate_fix()`: Generate fix for issue
  - `_create_prompt()`: Format prompt for LLM
  - `_parse_response()`: Parse and validate JSON response
  - `_clean_json()`: Remove markdown formatting
  - `test_connection()`: Test LLM connectivity
- OpenAI-compatible API support

**code_formatter.py**: Code formatting utilities
- Functions:
  - `extract_snippet()`: Extract code context around line
  - `format_for_llm()`: Format complete context for LLM
  - `highlight_issue_line()`: Mark problematic line
  - `truncate_content()`: Safely truncate long content
  - `format_code_block()`: Format for markdown
- Add line numbers and markers (>>>) to problematic lines

**issue_processor.py**: Main orchestration
- `IssueProcessor`: Coordinates the workflow
- Methods:
  - `process_issues()`: Main processing workflow
  - `process_single_issue()`: Process one issue
  - `save_results()`: Save to JSON file
  - `get_summary()`: Get processing statistics
- Progress tracking and error handling
- Batch processing with statistics

**report_generator.py**: Report generation
- `ReportGenerator`: Creates reports
- Methods:
  - `generate_markdown()`: Create markdown report
  - `generate_summary()`: Create summary section
  - `_format_issue_section()`: Format individual issue
- Include statistics, issue details, and fixes
- Professional formatting with syntax highlighting

**cli.py**: Command-line interface
- `main()`: Main entry point
- `create_parser()`: Argument parsing with argparse
- `load_config()`: Load and merge configuration
- `test_connections()`: Test SonarQube + LLM connections
- Support for config file, CLI args, and env vars
- Verbose mode for debugging

**__init__.py**: Package exports
```python
"""SonarQube LLM Fixer"""
__version__ = "0.1.0"
from .config import Config
from .issue_processor import IssueProcessor
__all__ = ["Config", "IssueProcessor"]
```

**__main__.py**: Module entry point
```python
"""Entry point for module execution"""
from .cli import main
if __name__ == "__main__":
    main()
```

## Implementation Checklist

### Core Modules
- [ ] Create `models.py` with all data classes
- [ ] Create `config.py` with YAML configuration support and OpenAI blocking
- [ ] Create `sonarqube_client.py` with API integration
- [ ] Create `llm_client.py` with OpenAI library (includes endpoint validation)
- [ ] Create `llm_client_no_openai.py` as pure-requests alternative (OPTIONAL)
- [ ] Create `code_formatter.py` with formatting utilities
- [ ] Create `issue_processor.py` with main workflow
- [ ] Create `report_generator.py` with markdown generation
- [ ] Create `cli.py` with argument parsing
- [ ] Create `__init__.py` for package initialization
- [ ] Create `__main__.py` for module entry point

### Testing & Documentation
- [ ] Add type hints to all methods
- [ ] Add docstrings to all public methods
- [ ] Create unit tests for each module
- [ ] Test with actual SonarQube instance
- [ ] Create `config.yaml.template` with company endpoint warnings
- [ ] Create comprehensive README.md
- [ ] Create NETWORK_SECURITY.md verification guide
- [ ] Create COMPANY_LLM_SETUP.md setup guide
- [ ] Create `.gitignore` file

### Security & Validation
- [ ] Ensure no default to api.openai.com anywhere
- [ ] Add openai.com validation in config.py
- [ ] Add openai.com validation in llm_client.py
- [ ] Add endpoint logging to all LLM requests
- [ ] Test connection function shows endpoint used
- [ ] Verify telemetry is disabled
- [ ] Test with firewall blocking openai.com
- [ ] Document network verification methods

### Quality Assurance
- [ ] Ensure proper error handling in all modules
- [ ] Add logging for debugging
- [ ] Validate all configuration inputs
- [ ] Test edge cases (no issues, API failures, etc.)
- [ ] Verify token usage tracking
- [ ] Test with different LLM providers

## Key Design Principles

1. **Single Responsibility**: Each module has one clear purpose
2. **Dependency Injection**: Pass dependencies via constructors
3. **Testability**: All components can be tested independently
4. **Type Safety**: Use type hints throughout
5. **Error Handling**: Graceful degradation with informative messages
6. **Configurability**: Support multiple configuration sources
7. **Extensibility**: Easy to add new features or backends
8. **Network Security**: NEVER contact external URLs, especially openai.com
9. **Transparency**: Always log which endpoints are being contacted
10. **Defense in Depth**: Multiple validation layers for critical security constraints

## Network Security Guarantees

The implementation MUST guarantee:

1. **Zero External Requests**: The tool will NEVER attempt to contact openai.com or any external URL when properly configured
2. **Explicit Validation**: Configuration will be rejected if it points to blocked endpoints
3. **Runtime Protection**: Multiple validation layers prevent accidental external access
4. **Transparency**: All network requests are logged showing exact endpoints
5. **Verifiable**: Documentation includes methods to verify no external access (network monitoring, firewall tests, etc.)

When properly configured with a company's internal LLM endpoint, the tool operates entirely within the corporate network with zero external communication.

---

## Usage Instructions for This Prompt

Copy this entire markdown file and paste it into your company's LLM with the instruction:

> "Please implement this Python tool following all the specifications. Make sure the code is production-ready with proper error handling, type hints, comprehensive docstrings, and includes the YAML configuration support. 
>
> CRITICAL: The tool must NEVER contact openai.com or any external URLs. It must work exclusively with our company's internal LLM endpoint. Include all security validations and network protection layers specified in the prompt."

If the LLM's response is incomplete, follow up with:

> "Please continue from where you left off"

For modifications:

> "Please update the code to also include [your specific requirement]"

## Important Notes for Implementation

1. **Network Security is Critical**: This is a hard requirement. The tool must be designed from the ground up to prevent any external network access.

2. **Two Client Options**: Provide both `llm_client.py` (using OpenAI library) and `llm_client_no_openai.py` (pure requests) so users can choose based on their security requirements.

3. **Clear Documentation**: Include extensive documentation about network security, verification methods, and company endpoint setup.

4. **Multiple Validation Layers**: Don't rely on a single check. Validate at config load, client initialization, and runtime.

5. **Transparency**: Always log which endpoints are being used so users can verify correct configuration.

6. **Helpful Error Messages**: When validation fails, provide clear guidance on what the correct configuration should be.
