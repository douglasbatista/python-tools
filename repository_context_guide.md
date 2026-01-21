# Repository Context Feature

## Overview

The repository context feature allows the tool to access your **local source code repository** to provide better context to the LLM when generating fixes.

## Why Use Repository Context?

### Without Repository Context
The LLM only sees:
- The specific file with the issue
- Lines around the problematic code
- SonarQube's issue description

### With Repository Context
The LLM additionally sees:
- **Related files** (imports, dependencies)
- **Project structure** and patterns
- **Existing utility functions** to reuse
- **Type definitions** and interfaces
- **How the code is used** elsewhere
- **Test files** (if they exist)

This results in:
- ✅ **Smarter fixes** that fit your codebase
- ✅ **Better suggestions** using existing utilities
- ✅ **Consistent style** matching your project
- ✅ **Fewer breaking changes** by understanding dependencies
- ✅ **More accurate** type usage and imports

## Configuration

### In config.yaml

```yaml
processing:
  # Path to your local repository (where the source code is)
  repository_path: "/home/user/projects/my-application"
  
  # Or relative path:
  # repositoryPath: "../my-application"
  
  # Enable/disable repository context (default: true)
  include_repository_context: true
  
  # Maximum related files to include (default: 3)
  max_related_files: 3
  
  # Maximum tokens for context (default: 8000)
  # Prevents overwhelming the LLM with too much context
  max_context_tokens: 8000
```

### Via Environment Variable

```bash
export REPOSITORY_PATH="/path/to/repository"
```

### Via Command Line

```bash
uv run python -m sonarqube_fixer \
  --config config.yaml \
  --repository-path /path/to/repository
```

## What Files Are Included?

The tool automatically finds **relevant** files:

### 1. Imported/Dependency Files
Files that are imported by the problematic file:

**Python example:**
```python
# If issue is in user_service.py which imports:
from models.user import User
from utils.validation import validate_email

# Tool will include:
# - models/user.py
# - utils/validation.py
```

**C# example:**
```csharp
// If issue is in UserService.cs which imports:
using Company.Models;
using Company.Utils;

// Tool will include:
// - Company/Models/User.cs
// - Company/Utils/ValidationUtils.cs
```

**JavaScript/TypeScript example:**
```javascript
// If issue is in userService.js which imports:
import { User } from './models/user';
import { validateEmail } from '../utils/validation';

// Tool will include:
// - models/user.js
// - ../utils/validation.js
```

**Visual Basic example:**
```vb
' If issue is in UserService.vb which imports:
Imports Company.Models
Imports Company.Utils

' Tool will include:
' - Company/Models/User.vb
' - Company/Utils/ValidationUtils.vb
```

**HTML example:**
```html
<!-- If issue is in index.html which references: -->
<script src="js/app.js"></script>
<link rel="stylesheet" href="css/styles.css">

<!-- Tool will include: -->
<!-- - js/app.js -->
<!-- - css/styles.css -->
```

### 2. Files in Same Directory
Related modules in the same package/directory:

```
src/services/
├── user_service.py    ← Issue is here
├── auth_service.py    ← Included (related service)
└── email_service.py   ← Included (related service)
```

### 3. Test Files
Corresponding test files (if they exist):

**Python:**
```
src/user_service.py         ← Issue
tests/test_user_service.py  ← Included
```

**C#:**
```
src/UserService.cs          ← Issue
test/UserServiceTests.cs    ← Included
```

**JavaScript/TypeScript:**
```
src/userService.js          ← Issue
src/userService.test.js     ← Included
```

**Visual Basic:**
```
src/UserService.vb          ← Issue
test/UserServiceTests.vb    ← Included
```

## Supported Languages

The tool understands imports and dependencies for:

- ✅ **C#** - `using` statements, namespace resolution
- ✅ **Python** - `import`, `from ... import`
- ✅ **JavaScript** - `import`, `require`, dynamic imports
- ✅ **TypeScript** - `import`, module resolution
- ✅ **Visual Basic** - `Imports` statements
- ✅ **HTML** - `<script>`, `<link>`, `<a>`, `<iframe>` references

## Example Usage

### Scenario: Null Pointer Bug

**Without repository context:**
```
Issue: Potential null pointer dereference on user.Name
LLM suggests: Add null check
```

**With repository context:**
```
Issue: Potential null pointer dereference on user.Name

Related files shown to LLM:
- Models/User.cs (sees nullable reference types)
- Utils/ValidationUtils.cs (sees existing validation methods)

LLM suggests: Use ValidationUtils.RequireNonNull(user).Name
(Reuses existing project utility!)
```

### Scenario: Code Smell

**Without repository context:**
```
Issue: Complex method should be split
LLM suggests: Generic refactoring
```

**With repository context:**
```
Issue: Complex method should be split

Related files shown to LLM:
- Services/BaseService.cs (sees common patterns)
- Utils/DataProcessor.cs (sees existing utilities)

LLM suggests: Extract to DataProcessor.ProcessAndValidate()
(Uses existing project architecture!)
```

## Configuration Tips

### Small Projects (<1000 files)
```yaml
processing:
  repository_path: "/path/to/project"
  max_related_files: 5       # More context
  max_context_tokens: 10000  # Higher limit
```

### Large Projects (>1000 files)
```yaml
processing:
  repository_path: "/path/to/project"
  max_related_files: 2       # Less context
  max_context_tokens: 6000   # Lower limit
```

### Monorepos
Point to the specific service/module:
```yaml
processing:
  repository_path: "/path/to/monorepo/services/user-service"
```

## Performance Considerations

### Impact on Speed
- **File reading**: Minimal (1-2 seconds)
- **Context analysis**: <1 second per issue
- **LLM processing**: Slightly longer (more tokens)

### Impact on Cost
- **More tokens sent** to LLM (repository context)
- **Typically 20-40% more tokens**
- **But much better quality** fixes

**Recommendation**: The improved fix quality is worth the extra cost!

## Troubleshooting

### Issue: "Repository path does not exist"

**Solution:**
```yaml
# Use absolute path
repository_path: "/home/user/projects/my-app"

# Or verify relative path
repository_path: "../my-app"  # Relative to where you run the tool
```

### Issue: No related files found

**Possible causes:**
- Import patterns not recognized
- Files not in expected locations
- Different project structure

**Solutions:**
- Check file paths are correct
- Increase `max_related_files`
- Submit issue with your project structure

### Issue: Too much context, LLM errors

**Symptoms:**
- "Token limit exceeded" errors
- Slow LLM responses

**Solutions:**
```yaml
# Reduce context
max_related_files: 2
max_context_tokens: 5000

# Or disable for specific runs
include_repository_context: false
```

### Issue: Wrong files included

**Cause:** Import detection logic

**Workaround:**
- Manually review fixes carefully
- Adjust `max_related_files`
- File an issue with examples

## When NOT to Use Repository Context

❌ **Very large files** - If related files are huge, may exceed token limits  
❌ **Private/sensitive code** - If LLM is external (though it shouldn't be)  
❌ **CI/CD with limited disk** - If repository not available  
❌ **Cost concerns** - If trying to minimize tokens  

## Best Practices

### 1. Keep Repository Updated
```bash
# Before running the tool
cd /path/to/repository
git pull
```

### 2. Use Clean Working Directory
```bash
# Avoid uncommitted changes confusing the tool
git status  # Should be clean
```

### 3. Point to Right Branch
```bash
# Checkout the branch being analyzed
git checkout develop
```

### 4. Test First
```bash
# Test with one issue first
uv run python -m sonarqube_fixer \
  --config config.yaml \
  --max-issues 1 \
  --verbose
```

### 5. Review Related Files
Check the logs to see which files are included:
```
Processing issue: java:S2259
  Related files found:
    - src/models/User.java
    - src/utils/ValidationUtils.java
    - src/test/UserServiceTest.java
```

## Example: Complete Configuration

```yaml
# config.yaml
sonarqube:
  url: "https://sonarqube.company.com"
  token: "squ_token"
  project_key: "com.company:user-service"

llm:
  api_key: "llm-key"
  base_url: "https://llm.company.com/v1"
  model: "gpt-4"

processing:
  max_issues: 10
  severities: [BLOCKER, CRITICAL, MAJOR]
  only_new_code: true
  
  # Repository context configuration
  repository_path: "/home/user/projects/user-service"
  include_repository_context: true
  max_related_files: 3
  max_context_tokens: 8000
  
  output_file: "fixes.json"
  report_file: "report.md"
```

## Verification

Check the LLM prompts include repository context:

```bash
uv run python -m sonarqube_fixer \
  --config config.yaml \
  --max-issues 1 \
  --verbose

# Look for in logs:
# "Including repository context: 3 related files"
# "Context size: ~6500 tokens"
```

## Summary

Repository context is a **powerful feature** that significantly improves fix quality by giving the LLM access to your entire codebase.

✅ **Enable it** if you have local repository access  
✅ **Configure paths** correctly  
✅ **Adjust limits** based on project size  
✅ **Review output** to verify context is helpful  
✅ **Worth the cost** for much better fixes  

The LLM will generate fixes that **fit your codebase**, reuse **existing utilities**, and follow **your project's patterns**! 🚀
