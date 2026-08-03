---
name: python-architectural-fitness-functions
description: Instructs the agent to write automated tests (fitness functions) that verify structural integrity and enforce Clean Architecture's Dependency Rule using Python's ast module.
---

# Python Architectural Fitness Functions Guidelines

You are an expert Python developer specializing in Clean Architecture and automated architectural verification. When asked to write tests that ensure the structural integrity of a codebase or prevent architectural drift, you must create "fitness functions" and adhere to the following rules:

"Fitness function" is Neal Ford, Rebecca Parsons, and Patrick Kua's term (from *Building Evolutionary Architectures*) for an automated test that verifies an architectural characteristic rather than a functional behavior — the same way a unit test verifies code behavior. Run these as part of the normal test suite (`pytest tests/architecture`) so a violation is caught within seconds on a developer's own machine — a "shift left" catch, long before code review or CI, and certainly before the violation becomes an established pattern others copy.

For how to *design* the layers this skill verifies — functional-lite Functional Core/Imperative Shell structure, the Dependency Rule, dependency inversion without ABCs, and Screaming Architecture naming — see the `python-clean-architecture` package (python-clean-architecture-functional-core-imperative-shell, python-clean-architecture-dependency-rule, python-clean-architecture-dependency-inversion, python-clean-architecture-screaming-architecture). This skill assumes that layering already exists and checks it stays intact.

## 1. Verifying Layer Structure
Before checking dependencies, ensure the codebase maintains its explicit, well-defined layer organization.
* Use `pathlib.Path` to inspect the root application directory.
* Assert that only the expected architectural layers (e.g., `domain`, `application`, `interfaces`, `infrastructure`) exist as top-level directories. 
* Fail the test if unexpected directories (like a rogue `notifications` folder) are found at the root level, enforcing that developers place new features into the correct architectural layer.

## 2. Enforcing the Dependency Rule with `ast`
The most fundamental rule of Clean Architecture is that dependencies must only point inward toward more central layers. You must write tests that verify this rule statically.
* Do not rely on complex third-party static analysis tools if simple built-in capabilities suffice. Use Python's built-in `ast` (Abstract Syntax Tree) module to parse source files.
* Recursively find all `.py` files in an inner layer (like `domain`) using `Path.rglob()`.
* Parse each file using `ast.parse()` and walk the tree using `ast.walk()` to locate `ast.Import` and `ast.ImportFrom` nodes.
* Extract the module names being imported and verify they do not reference outer layers (like `infrastructure`, `interfaces`, or `application`).

## 3. Clear Violation Reporting
When a dependency rule violation is found, do not simply throw a generic error.
* Accumulate all violations in a list.
* Use assertions (like `unittest.TestCase.assertEqual`) to compare the violations list against an empty list `[]`.
* Provide a highly detailed failure message that identifies the exact file containing the violation, the rule that was broken, and how to fix it.

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
                    # ast.Import.names is a list of aliases (e.g. `import a.b, c.d`);
                    # take the first one. ast.ImportFrom has a single `module` string.
                    module = node.names[0].name if isinstance(node, ast.Import) else node.module
                    
                    if module and module.startswith("my_app."):
                        layer = module.split(".")[1]
                        
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

## 4. Extending Beyond the Two Core Checks

Structure and Dependency Rule checks are the high-impact starting point, not the ceiling. Once these are in place, consider extending fitness functions to:
* **Interface conformance:** verify a `Callable`-typed dependency's concrete implementations actually match the expected signature (see the `python-clean-architecture` package's dependency-inversion pattern — there's no ABC to check `issubclass` against, so this means signature inspection via `inspect.signature`, not an `isinstance`/`issubclass` check).
* **Naming/molds consistency:** verify a project's established name molds are followed (see the `code-review` package's code-review-naming-consistency for the concept these checks would automate).
* **Layer-specific rules:** add custom checks for constraints specific to one layer (e.g., no `async def` inside `domain/`, since domain functions should be pure and synchronous — see python-clean-architecture-functional-core-imperative-shell's async/await-as-shell-signal test).

Start with the two checks above; add more only once they've proven their value in practice, evolving fitness functions alongside the architecture rather than trying to anticipate every violation up front.