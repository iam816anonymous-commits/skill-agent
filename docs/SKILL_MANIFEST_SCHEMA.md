# Skill Manifest Schema Specification

## 1. Executive Summary

Every Skill in SkillAgent MUST be defined by an immutable, signed declarative JSON manifest file named `manifest.json`. The manifest acts as the definitive contract governing the skill's identity, versioning, capability requirements, configuration parameters, and execution entrypoint.

## 2. Complete Manifest JSON Schema (`manifest.json`)

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "SkillManifest",
  "type": "object",
  "required": [
    "schema_version",
    "id",
    "name",
    "version",
    "min_runtime_version",
    "entrypoint",
    "capabilities"
  ],
  "properties": {
    "schema_version": {
      "type": "string",
      "enum": ["1.0.0"]
    },
    "id": {
      "type": "string",
      "pattern": "^[a-z0-9_]{3,64}$",
      "description": "Unique reverse-domain or snake_case identifier (e.g., calculator_skill, sys_settings_v1)"
    },
    "name": {
      "type": "string",
      "maxLength": 100
    },
    "version": {
      "type": "string",
      "pattern": "^(0|[1-9]\\d*)\\.(0|[1-9]\\d*)\\.(0|[1-9]\\d*)$",
      "description": "Semantic versioning strings (e.g. 1.0.0)"
    },
    "min_runtime_version": {
      "type": "string",
      "pattern": "^(0|[1-9]\\d*)\\.(0|[1-9]\\d*)\\.(0|[1-9]\\d*)$"
    },
    "description": {
      "type": "string",
      "maxLength": 500
    },
    "author": {
      "type": "string",
      "maxLength": 100
    },
    "entrypoint": {
      "type": "object",
      "required": ["type", "class_name"],
      "properties": {
        "type": {
          "type": "string",
          "enum": ["BUILTIN_KOTLIN", "DYNAMIC_JVM_CLASS", "WORKER_EXECUTABLE"]
        },
        "class_name": {
          "type": "string",
          "pattern": "^([a-zA-Z_$][a-zA-Z\\d_$]*\\.)*[a-zA-Z_$][a-zA-Z\\d_$]*$"
        }
      }
    },
    "capabilities": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["category", "access_level"],
        "properties": {
          "category": {
            "type": "string",
            "enum": ["HARDWARE", "ACCESSIBILITY", "BROWSER", "STORAGE", "SYSTEM_SETTINGS"]
          },
          "access_level": {
            "type": "string",
            "enum": ["READ", "WRITE", "EXECUTE"]
          },
          "reason": {
            "type": "string",
            "maxLength": 250
          }
        }
      }
    },
    "input_schema": {
      "type": "object",
      "description": "JSON Schema validating SkillInput parameters"
    },
    "output_schema": {
      "type": "object",
      "description": "JSON Schema validating SkillOutput payload"
    },
    "resource_requirements": {
      "type": "object",
      "properties": {
        "max_heap_mb": {
          "type": "integer",
          "minimum": 5,
          "maximum": 512,
          "default": 35
        },
        "execution_timeout_sec": {
          "type": "integer",
          "minimum": 1,
          "maximum": 3600,
          "default": 60
        }
      }
    }
  }
}
```

## 3. Manifest Validation & Schema Rules

During skill import or registration, `SkillManifestValidator` executes four mandatory validation gates:

1. **Schema Syntax Verification**: Validates the JSON structure against the official Draft-07 JSON Schema above. Rejects malformed JSON, unknown schema versions, or missing required fields.
2. **Identifier Pinning**: Rejects duplicate skill IDs. If `calculator_skill` version `1.0.0` is installed, importing `calculator_skill` `1.0.0` again is rejected unless explicitly upgrading with a higher version number.
3. **Capability Sanitization**: Checks all declared capabilities against known capability categories. Rejects invalid or undefined categories.
4. **Runtime Compatibility Gate**: Verifies that `min_runtime_version` is less than or equal to `SkillAgent.RUNTIME_VERSION`. Rejects incompatible future manifests.

## 4. Manifest Location in Packages

In dynamic `.skill` packages (Phase 10 Marketplace), `manifest.json` MUST be positioned at the exact root of the `.skill` ZIP archive. Archives where `manifest.json` is located in subdirectories or missing are rejected immediately during import validation.
