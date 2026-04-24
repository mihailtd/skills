---
name: python-architectural-fitness-functions
description: Instructs the agent to write automated tests (fitness functions) that verify structural integrity and enforce Clean Architecture's Dependency Rule using Python's ast module.
---

# Python Architectural Fitness Functions Guidelines

You are an expert Python developer specializing in Clean Architecture and automated architectural verification. When asked to write tests that ensure the structural integrity of a codebase or prevent architectural drift, you must create "fitness functions" and adhere to the following rules [2, 4]:

## 1. Verifying Layer Structure
Before checking dependencies, ensure the codebase maintains its explicit, well-defined layer organization [5, 6].
* Use `pathlib.Path` to inspect the root application directory [6].
* Assert that only the expected architectural layers (e.g., `domain`, `application`, `interfaces`, `infrastructure`) exist as top-level directories [6, 7]. 
* Fail the test if unexpected directories (like a rogue `notifications` folder) are found at the root level, enforcing that developers place new features into the correct architectural layer [7-9].

## 2. Enforcing the Dependency Rule with `ast`
The most fundamental rule of Clean Architecture is that dependencies must only point inward toward more central layers [10, 11]. You must write tests that verify this rule statically.
* Do not rely on complex third-party static analysis tools if simple built-in capabilities suffice. Use Python's built-in `ast` (Abstract Syntax Tree) module to parse source files [11, 12].
* Recursively find all `.py` files in an inner layer (like `domain`) using `Path.rglob()` [12].
* Parse each file using `ast.parse()` and walk the tree using `ast.walk()` to locate `ast.Import` and `ast.ImportFrom` nodes [12, 13].
* Extract the module names being imported and verify they do not reference outer layers (like `infrastructure`, `interfaces`, or `application`) [13].

## 3. Clear Violation Reporting
When a dependency rule violation is found, do not simply throw a generic error.
* Accumulate all violations in a list [12, 13].
* Use assertions (like `unittest.TestCase.assertEqual`) to compare the violations list against an empty list `[]` [14].
* Provide a highly detailed failure message that identifies the exact file containing the violation, the rule that was broken, and how to fix it [15].

---

## Code Examples

Below are best-practice examples demonstrating how to write architectural fitness functions in Python.

**1. Verifying Layer Structure**
```python
import unittest
from pathlib import Path

class ArchitectureConfig:
    # Ordered from innermost to outermost layer
    LAYER_HIERARCHY = [
        "domain",
        "application",
        "interfaces",
        "infrastructure"
    ]

class TestArchitectureStructure(unittest.TestCase):
    def test_source_folders(self):
        """Verify the app contains only Clean Architecture layer folders."""
        src_path = Path("my_app")
        folders = {f.name for f in src_path.iterdir() if f.is_dir() and not f.name.startswith("__")}
        
        # 1. All required layer folders must exist
        for layer in ArchitectureConfig.LAYER_HIERARCHY:
            self.assertIn(layer, folders, f"Missing {layer} layer folder")
            
        # 2. No unexpected folders should exist at the root
        unexpected = folders - set(ArchitectureConfig.LAYER_HIERARCHY)
        self.assertEqual(
            unexpected,
            set(),
            f"Source should only contain Clean Architecture layers.\n"
            f"Unexpected folders found: {unexpected}"
        )
2. Enforcing the Dependency Rule (ast module)
import ast
import unittest
from pathlib import Path

class TestDependencyRules(unittest.TestCase):
    def test_domain_layer_dependencies(self):
        """Verify domain layer has no outward dependencies."""
        domain_path = Path("my_app/domain")
        violations = []
        
        # 1. Recursively find all Python files in the Domain layer
        for py_file in domain_path.rglob("*.py"):
            with open(py_file) as f:
                # 2. Parse the file into an Abstract Syntax Tree
                tree = ast.parse(f.read())
                
            # 3. Walk the AST to find Import and ImportFrom nodes
            for node in ast.walk(tree):
                if isinstance(node, ast.Import) or isinstance(node, ast.ImportFrom):
                    # Safely handle the import name extraction
                    module = node.names.name if isinstance(node, ast.Import) else node.module
                    
                    if module and module.startswith("my_app."):
                        layer = module.split(".")[16]
                        
                        # 4. Check if the import references an outer layer
                        if layer in ["infrastructure", "interfaces", "application"]:
                            violations.append(
                                f"{py_file.relative_to(domain_path)}: "
                                f"Domain layer cannot import from {layer} layer"
                            )
                            
        # 5. Assert no violations exist, printing all gathered errors clearly
        self.assertEqual(
            violations,
            [],
            "\nDependency Rule Violations:\n" + "\n".join(violations)
        )
```