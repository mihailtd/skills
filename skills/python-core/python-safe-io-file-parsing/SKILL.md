---
name: python-safe-io-file-parsing
description: Instructs the agent to strictly use the pathlib module for filesystem operations, parse files non-blockingly using generators, map CSVs directly into dataclasses, and use compiled regular expressions with named capture groups for log parsing.
---

# Python Safe I/O & File Parsing Guidelines

You are an expert Python developer specializing in robust file handling and data extraction. When asked to read, write, or parse files, you must adhere to the following rules:

## 1. Mandatory use of `pathlib`
Do not use raw strings and the older `os.path` module to manipulate paths, as dealing with edge cases like slashes in filenames or dots in directories makes string processing needlessly difficult.
*   Always use `pathlib.Path` objects for filesystem operations.
*   Use the `/` operator to assemble paths from components.
*   Use built-in properties and methods like `.parent`, `.stem`, and `.with_suffix()` to cleanly manipulate paths.

## 2. Safe I/O and Lazy Parsing (Generators)
Never leak file descriptors and avoid reading vast amounts of data into in-memory lists.
*   Always wrap file operations (`Path.open()`) in a `with` statement context manager to ensure OS resources are properly released.
*   Process files lazily. Write generator functions (using the `yield` statement) to read and yield one row at a time. This prevents out-of-memory errors when processing massive datasets.

## 3. Parsing CSVs into Dataclasses
When processing tabular CSV data, bridge the physical format and your application logic by utilizing dataclasses.
*   Use `csv.DictReader` to automatically map the header row to dictionary keys.
*   Define a `@dataclass` where the attribute names perfectly match the CSV column headers.
*   Inside a generator, instantiate the dataclass by unpacking the dictionary directly using `**row`.

## 4. Complex Parsing with Named Regex Groups
When parsing complex, non-standard files (like web server logs) where `csv` cannot be used, rely on compiled regular expressions.
*   Pre-compile your regex pattern using `re.compile()`.
*   Use the `re.X` (verbose) flag to break the pattern across multiple lines for readability.
*   Use named capture groups with the syntax `(?P<name>...)`, ensuring the names are valid Python identifiers.
*   Extract the parsed data using the `match.groupdict()` method, and unpack it directly (`**data`) into a strongly typed `NamedTuple` or `@dataclass`.

---

## Code Examples

Below are best-practice examples demonstrating how to safely parse files using `pathlib`, lazy generators, and strongly typed objects.

**1. CSV Parsing into a Dataclass (Lazy Generator)**
```python
import csv
from pathlib import Path
from dataclasses import dataclass
from collections.abc import Iterator

# The dataclass attributes precisely match the CSV column headers
@dataclass
class RawRow:
    date: str
    time: str
    lat: str
    lon: str

def waypoint_iter(data_path: Path) -> Iterator[RawRow]:
    # 1. Safely open the file using pathlib and a context manager
    with data_path.open("r", newline="") as data_file:
        # 2. Use DictReader to parse the physical format
        reader = csv.DictReader(data_file)
        
        # 3. Lazily yield dataclass instances by unpacking the dictionary
        for row in reader:
            yield RawRow(**row)
2. Complex Log Parsing with Named Capture Groups
import re
from pathlib import Path
from typing import NamedTuple
from collections.abc import Iterator

class LogLine(NamedTuple):
    date: str
    level: str
    module: str
    message: str

# Use re.X to allow whitespace and formatting in the pattern string
# Use (?P<name>...) to assign names to the capture groups
LOG_PATTERN = re.compile(
    r"\[(?P<date>.*?)]\s+"
    r"(?P<level>\w+)\s+"
    r"in\s+(?P<module>\S+?)"
    r":\s+(?P<message>.+)",
    re.X
)

def log_parser_iter(data_path: Path) -> Iterator[LogLine]:
    with data_path.open("r") as data_file:
        for source_line in data_file:
            if match := LOG_PATTERN.match(source_line):
                # match.groupdict() returns a dictionary of the named groups
                # Unpack the dictionary directly into the NamedTuple
                yield LogLine(**match.groupdict())
            else:
                raise ValueError(f"Unexpected input {source_line=}")
```