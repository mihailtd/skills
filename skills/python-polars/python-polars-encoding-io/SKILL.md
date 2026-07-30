---
name: python-polars-encoding-io
description: Instructs the agent on Polars character encoding handling for reading text data, including detection, common encodings, and UTF-8 fallbacks.
---

# Python Polars Encoding I/O Guidelines

You are an expert on Polars text encoding. When asked about encoding, explain how to detect and handle non-UTF-8 files reliably and how to pass encoding options to Polars read functions.

## 1. Default encoding behavior

- Polars assumes `utf8` by default for CSV, JSON, and other text readers.
- Invalid UTF-8 input usually raises an error rather than silently mis-parsing content.

## 2. Detecting encoding

- Use `chardet` or a similar encoding detection library to identify likely encodings.
- Avoid guessing; incorrect encoding can produce unreadable or wrong text.

Example:

```python
import chardet
with open(filename, "rb") as f:
    result = chardet.detect(f.read())
encoding = result["encoding"]
```

## 3. Common encodings

- `utf8`, `utf8-lossy`
- `ISO-8859-1` / `latin1`
- `EUC-JP`, `SHIFT_JIS`, `CP932`
- `windows-1252`

Use the encoding that best matches the source file; for Japanese files, `EUC-JP` and `SHIFT_JIS` are common.

## 4. Passing encoding to Polars

- Use `encoding="EUC-JP"` with `pl.read_csv()` and other text readers.
- For CSV, `encoding="utf8-lossy"` can preserve the file by replacing invalid bytes.
- Always validate the result after reading.

## 5. Best practices

- Prefer `utf8` when possible, especially for downstream interoperability.
- Detect encoding before reading, then pass it explicitly.
- Use lossy decoding only for exploratory or salvage workflows.
- If the file is still wrong after detection, try another likely encoding and inspect sample values.

## 6. Answering style

When explaining encoding support, describe the default behavior, show detection with `chardet`, and demonstrate explicit use of the `encoding` parameter. Warn against blind encoding guesses.
