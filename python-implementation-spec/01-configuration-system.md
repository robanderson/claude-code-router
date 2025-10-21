# Configuration System Specification

## Overview

The configuration system manages all application settings through a JSON5 configuration file stored at `~/.claude-code-router/config.json`. It handles loading, validation, environment variable interpolation, and automatic backups.

## Configuration File Location

**Primary location**: `~/.claude-code-router/config.json`

**Related files**:
- `~/.claude-code-router/config.json.YYYY-MM-DDTHH-MM-SS-SSS.bak` - Backup files (keep 3 most recent)
- `~/.claude-code-router/plugins/` - Directory for custom transformers and routers

## Configuration File Format

The configuration uses JSON5 format, which allows:
- Comments (`//` and `/* */`)
- Trailing commas
- Unquoted object keys
- Single-quoted strings

### Complete Configuration Schema

```json5
{
  // Server Configuration
  "PORT": 3456,                    // Port for the router service (default: 3456)
  "HOST": "127.0.0.1",            // Host to bind to (forced to 127.0.0.1 if no APIKEY)
  "APIKEY": "your-secret-key",    // Optional API key for authentication

  // Logging Configuration
  "LOG": true,                     // Enable/disable logging (default: true)
  "LOG_LEVEL": "debug",           // Log level: fatal|error|warn|info|debug|trace

  // Operational Settings
  "API_TIMEOUT_MS": 600000,       // API call timeout in milliseconds (default: 10 min)
  "PROXY_URL": "http://127.0.0.1:7890",  // Optional HTTP proxy
  "NON_INTERACTIVE_MODE": false,  // Enable for CI/CD environments

  // Claude Code Integration
  "CLAUDE_PATH": "claude",        // Path to Claude Code executable
  "ANTHROPIC_SMALL_FAST_MODEL": "provider,model",  // Override small/fast model

  // Advanced Features
  "CUSTOM_ROUTER_PATH": "/path/to/custom-router.js",  // Custom routing logic
  "REWRITE_SYSTEM_PROMPT": "/path/to/prompt.txt",     // Override system prompt
  "forceUseImageAgent": false,    // Force image agent for non-tool-calling models

  // Provider Configuration
  "Providers": [
    {
      "name": "provider-name",              // Unique provider identifier
      "api_base_url": "https://...",        // API endpoint
      "api_key": "sk-...",                  // API key (can use env vars)
      "models": ["model1", "model2"],       // Available models
      "transformer": {                       // Optional transformers
        "use": ["transformer1", ["transformer2", {options}]],
        "model-specific": {                 // Model-specific transformers
          "use": ["transformer3"]
        }
      }
    }
  ],

  // Routing Rules
  "Router": {
    "default": "provider,model",           // Default model for all requests
    "background": "provider,model",        // For Haiku background tasks
    "think": "provider,model",             // For reasoning/thinking tasks
    "longContext": "provider,model",       // For long context (>threshold tokens)
    "longContextThreshold": 60000,         // Token threshold (default: 60000)
    "webSearch": "provider,model",         // For web search tasks
    "image": "provider,model"              // For image analysis tasks
  },

  // Custom Transformers
  "transformers": [
    {
      "path": "/path/to/transformer.js",   // Path to custom transformer
      "options": {                          // Transformer options
        "key": "value"
      }
    }
  ],

  // Status Line Configuration
  "StatusLine": {
    "enabled": true,
    "currentStyle": "default",             // or "powerline" or "simple"
    "default": {                           // Default style config
      "modules": [
        {
          "type": "workDir",
          "icon": "󰉋",
          "text": "{{workDirName}}",
          "color": "bright_blue"
        }
        // ... more modules
      ]
    }
  }
}
```

## Data Structures

### Provider Configuration

```python
@dataclass
class TransformerConfig:
    use: List[Union[str, Tuple[str, Dict[str, Any]]]]
    # Additional model-specific transformer configs
    # Key is model name, value is another TransformerConfig

@dataclass
class Provider:
    name: str
    api_base_url: str
    api_key: str
    models: List[str]
    transformer: Optional[TransformerConfig] = None
```

### Router Configuration

```python
@dataclass
class RouterConfig:
    default: str                           # Required: "provider,model"
    background: Optional[str] = None       # "provider,model"
    think: Optional[str] = None
    longContext: Optional[str] = None
    longContextThreshold: int = 60000
    webSearch: Optional[str] = None
    image: Optional[str] = None
```

### Complete Configuration

```python
@dataclass
class Config:
    # Server settings
    PORT: int = 3456
    HOST: str = "127.0.0.1"
    APIKEY: Optional[str] = None

    # Logging
    LOG: bool = True
    LOG_LEVEL: str = "debug"

    # Operational
    API_TIMEOUT_MS: int = 600000
    PROXY_URL: Optional[str] = None
    NON_INTERACTIVE_MODE: bool = False

    # Integration
    CLAUDE_PATH: str = "claude"
    ANTHROPIC_SMALL_FAST_MODEL: Optional[str] = None

    # Advanced
    CUSTOM_ROUTER_PATH: Optional[str] = None
    REWRITE_SYSTEM_PROMPT: Optional[str] = None
    forceUseImageAgent: bool = False

    # Core config
    Providers: List[Provider]
    Router: RouterConfig
    transformers: List[Dict[str, Any]] = field(default_factory=list)
    StatusLine: Optional[Dict[str, Any]] = None
```

## Configuration Loading Process

### 1. File Discovery

```python
def get_config_path() -> Path:
    """
    Returns: ~/.claude-code-router/config.json
    """
    home = Path.home()
    return home / ".claude-code-router" / "config.json"
```

### 2. Directory Initialization

```python
async def init_dir() -> None:
    """
    Creates necessary directories if they don't exist:
    - ~/.claude-code-router/
    - ~/.claude-code-router/plugins/
    - ~/.claude-code-router/logs/
    """
    home_dir = Path.home() / ".claude-code-router"
    home_dir.mkdir(parents=True, exist_ok=True)
    (home_dir / "plugins").mkdir(exist_ok=True)
    (home_dir / "logs").mkdir(exist_ok=True)
```

### 3. Environment Variable Interpolation

The system supports environment variable references in the format:
- `$VAR_NAME` - Simple form
- `${VAR_NAME}` - Braced form

**Interpolation Algorithm**:

```python
import os
import re

def interpolate_env_vars(obj: Any) -> Any:
    """
    Recursively interpolate environment variables in config values.

    Patterns:
    - ${VAR_NAME} -> os.environ.get('VAR_NAME', '${VAR_NAME}')
    - $VAR_NAME -> os.environ.get('VAR_NAME', '$VAR_NAME')

    If variable doesn't exist, keep the original placeholder.
    """
    if isinstance(obj, str):
        # Pattern: ${VAR_NAME} or $VAR_NAME (uppercase letters, numbers, underscores)
        pattern = r'\$\{([^}]+)\}|\$([A-Z_][A-Z0-9_]*)'

        def replace(match):
            var_name = match.group(1) or match.group(2)
            return os.environ.get(var_name, match.group(0))

        return re.sub(pattern, replace, obj)

    elif isinstance(obj, dict):
        return {key: interpolate_env_vars(value) for key, value in obj.items()}

    elif isinstance(obj, list):
        return [interpolate_env_vars(item) for item in obj]

    return obj
```

### 4. Config File Reading

```python
import json5  # or use json with preprocessing to remove comments

async def read_config_file() -> Config:
    """
    Loads and parses the configuration file.

    Returns:
        Parsed and validated configuration object

    Raises:
        FileNotFoundError: If config doesn't exist
        json5.JSONDecodeError: If config is malformed
    """
    config_path = get_config_path()

    if not config_path.exists():
        # Create minimal default config
        await init_dir()
        default_config = {
            "PORT": 3456,
            "Providers": [],
            "Router": {}
        }
        await write_config_file(default_config)
        print(f"Created default config at {config_path}")
        return Config(**default_config)

    try:
        content = config_path.read_text()
        config_dict = json5.loads(content)  # Parse JSON5
        config_dict = interpolate_env_vars(config_dict)  # Interpolate env vars

        # Validate and create Config object
        return validate_config(config_dict)

    except json5.JSONDecodeError as e:
        print(f"Failed to parse config file: {e}")
        raise
```

### 5. Config Validation

```python
def validate_config(config_dict: dict) -> Config:
    """
    Validates the configuration and returns a Config object.

    Validation rules:
    - Providers must be a list
    - Each provider must have name, api_base_url, api_key, models
    - Router must have 'default' key
    - Model references must be in format "provider,model"
    - PORT must be 1-65535
    - LOG_LEVEL must be valid
    """
    # Validate Providers
    if not isinstance(config_dict.get("Providers"), list):
        raise ValueError("Providers must be a list")

    for provider in config_dict["Providers"]:
        if not all(k in provider for k in ["name", "api_base_url", "api_key", "models"]):
            raise ValueError(f"Provider missing required fields: {provider.get('name', 'unknown')}")

    # Validate Router
    if "Router" not in config_dict:
        raise ValueError("Router configuration is required")

    if "default" not in config_dict["Router"]:
        raise ValueError("Router must have a 'default' model")

    # Validate model references
    available_models = set()
    for provider in config_dict["Providers"]:
        for model in provider["models"]:
            available_models.add(f"{provider['name']},{model}")

    # More validation as needed...

    return Config(**config_dict)
```

## Configuration Backup

### Backup Strategy

When config is updated (via API or model selector):

```python
async def backup_config_file() -> Optional[Path]:
    """
    Creates a timestamped backup of the current config file.
    Keeps only the 3 most recent backups.

    Returns:
        Path to backup file, or None if config doesn't exist
    """
    config_path = get_config_path()

    if not config_path.exists():
        return None

    # Create backup with timestamp
    timestamp = datetime.now().isoformat().replace(':', '-')
    backup_path = config_path.with_suffix(f'.{timestamp}.bak')

    # Copy file
    import shutil
    shutil.copy2(config_path, backup_path)

    # Clean up old backups (keep 3 most recent)
    config_dir = config_path.parent
    backup_files = sorted(
        config_dir.glob('config.json.*.bak'),
        key=lambda p: p.stat().st_mtime,
        reverse=True
    )

    # Delete all but the 3 most recent
    for old_backup in backup_files[3:]:
        old_backup.unlink()

    return backup_path
```

## Configuration Writing

```python
async def write_config_file(config: Union[Config, dict]) -> None:
    """
    Writes configuration to disk with proper formatting.
    """
    config_path = get_config_path()

    # Ensure directory exists
    await init_dir()

    # Convert Config object to dict if needed
    if isinstance(config, Config):
        config_dict = asdict(config)
    else:
        config_dict = config

    # Write with indentation
    content = json.dumps(config_dict, indent=2)
    config_path.write_text(content)
```

## Configuration Initialization at Startup

```python
async def init_config() -> Config:
    """
    Initializes configuration at application startup.

    This is called by the server initialization code.
    Also sets environment variables for child processes.
    """
    config = await read_config_file()

    # Export config values to environment for child processes
    for key, value in asdict(config).items():
        if isinstance(value, (str, int, bool)):
            os.environ[key] = str(value)

    return config
```

## Special Configuration Behaviors

### Host/API Key Enforcement

```python
def validate_host_apikey(config: Config) -> Config:
    """
    If APIKEY is not set, force HOST to 127.0.0.1 for security.
    """
    if config.HOST and not config.APIKEY:
        print("⚠️ API key is not set. HOST is forced to 127.0.0.1.")
        config.HOST = "127.0.0.1"

    return config
```

### Claude Code Config Initialization

The system also creates/maintains `~/.claude.json` for Claude Code:

```python
async def initialize_claude_config() -> None:
    """
    Creates ~/.claude.json if it doesn't exist.
    This is required for Claude Code to function.
    """
    claude_config_path = Path.home() / ".claude.json"

    if claude_config_path.exists():
        return

    # Generate a random user ID
    import random
    user_id = ''.join(random.choices('0123456789abcdef', k=64))

    claude_config = {
        "numStartups": 184,
        "autoUpdaterStatus": "enabled",
        "userID": user_id,
        "hasCompletedOnboarding": True,
        "lastOnboardingVersion": "1.0.17",
        "projects": {}
    }

    claude_config_path.write_text(json.dumps(claude_config, indent=2))
```

## Python Implementation Recommendations

### Recommended Libraries

1. **Configuration parsing**:
   - `pydantic` - For data validation and settings management
   - `pyjson5` - For JSON5 parsing with comments
   - Or preprocess JSON5 to remove comments and use standard `json`

2. **Dataclasses**:
   - `dataclasses` (standard library) or `pydantic.BaseModel`

3. **File watching** (optional):
   - `watchdog` - To auto-reload config on file changes

### Implementation Structure

```python
# config.py
from dataclasses import dataclass, field
from typing import Optional, List, Dict, Any, Union
from pathlib import Path
import json
import os
import re
from datetime import datetime

# Or use Pydantic for validation:
from pydantic import BaseModel, Field, validator

class Provider(BaseModel):
    name: str
    api_base_url: str
    api_key: str
    models: List[str]
    transformer: Optional[Dict[str, Any]] = None

class RouterConfig(BaseModel):
    default: str
    background: Optional[str] = None
    think: Optional[str] = None
    longContext: Optional[str] = None
    longContextThreshold: int = 60000
    webSearch: Optional[str] = None
    image: Optional[str] = None

class Config(BaseModel):
    PORT: int = 3456
    HOST: str = "127.0.0.1"
    APIKEY: Optional[str] = None
    LOG: bool = True
    LOG_LEVEL: str = "debug"
    API_TIMEOUT_MS: int = 600000
    Providers: List[Provider]
    Router: RouterConfig
    # ... other fields

    @validator('HOST')
    def validate_host_with_apikey(cls, v, values):
        if v != "127.0.0.1" and not values.get('APIKEY'):
            return "127.0.0.1"
        return v

class ConfigManager:
    def __init__(self):
        self.config_path = Path.home() / ".claude-code-router" / "config.json"
        self.config: Optional[Config] = None

    async def load(self) -> Config:
        """Load and return configuration"""
        ...

    async def save(self, config: Config) -> None:
        """Save configuration to disk"""
        ...

    async def backup(self) -> Optional[Path]:
        """Create backup of current config"""
        ...
```

## Error Handling

### Config Not Found
- Create default minimal config
- Log warning
- Continue with defaults

### Parse Error
- Log error with line number if possible
- Exit with error code
- Suggest checking config syntax

### Validation Error
- Log specific validation failure
- Exit with error code
- Provide helpful error message

## Testing Considerations

The configuration system should have tests for:
- Loading valid configs
- Handling missing config (creates default)
- Environment variable interpolation
- Config validation (missing required fields)
- Backup creation and rotation
- Host/API key enforcement
- Transformer config parsing
