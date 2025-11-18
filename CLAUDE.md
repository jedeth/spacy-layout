# CLAUDE.md - AI Assistant Guide for spacy-layout

## Project Overview

**spacy-layout** is a Python library that integrates with [Docling](https://ds4sd.github.io/docling/) to bring structured processing of PDFs, Word documents, and other formats to [spaCy](https://spacy.io) pipelines. It converts documents into clean, structured data and creates spaCy `Doc` objects with labeled text spans (sections, headings, tables) and layout information.

- **Current Version**: 0.0.12 (Beta)
- **Repository**: https://github.com/explosion/spacy-layout
- **License**: MIT
- **Maintainer**: Explosion AI
- **Python Requirements**: >=3.10

### Core Purpose

Bridge document processing (PDFs, DOCX) with spaCy's NLP capabilities by:
1. Using Docling to parse and extract content from documents
2. Converting extracted content to spaCy `Doc` objects with layout annotations
3. Enabling NLP workflows (NER, classification, etc.) on document content
4. Providing access to tables as pandas DataFrames
5. Supporting RAG chunking and structured data extraction

---

## Codebase Structure

```
spacy-layout/
├── .github/workflows/      # CI/CD automation
│   ├── test.yml           # pytest on push/PR
│   └── publish.yml        # PyPI publishing on releases
├── spacy_layout/          # Main source package (~355 LOC)
│   ├── __init__.py       # Package entry point (exports spaCyLayout)
│   ├── layout.py         # Core implementation (236 lines) ⭐ MAIN MODULE
│   ├── types.py          # Dataclass definitions (63 lines)
│   └── util.py           # Serialization utilities (54 lines)
├── tests/                # Test suite (~215 LOC)
│   ├── data/            # Test PDFs and DOCX files
│   │   ├── simple.pdf
│   │   ├── simple.docx
│   │   ├── starcraft.pdf
│   │   ├── table.pdf
│   │   └── table_document_index.pdf
│   └── test_general.py  # Main test module (pytest)
├── .gitignore
├── LICENSE              # MIT License
├── README.md            # Comprehensive documentation (285 lines)
├── requirements.txt     # Dependencies (runtime + dev)
├── setup.cfg           # Package metadata and configuration
└── setup.py            # Minimal build script
```

### Key Characteristics

- **Small, focused codebase**: ~355 lines of source code
- **Clean separation**: types, utilities, and business logic in separate modules
- **Well-tested**: 12 test functions covering major functionality
- **Minimal dependencies**: Only 4 runtime dependencies (spacy, docling, pandas, srsly)
- **Type-annotated**: Full type hints throughout with TYPE_CHECKING guards

---

## Key Components and Architecture

### 1. Main Class: `spaCyLayout` (layout.py:43-236)

The primary API for document processing.

**Initialization Parameters:**
- `nlp`: spaCy Language object (required) - used for tokenization
- `separator`: Text separator between sections (default: `"\n\n"`)
- `attrs`: Override custom attribute names
- `headings`: Label types to consider as headings (default: section_header, page_header, title)
- `display_table`: Function or string for table text representation (default: `"TABLE"`)
- `docling_options`: Format options passed to Docling's DocumentConverter

**Main Methods:**
- `__call__(source)`: Process single document → returns spaCy Doc
- `pipe(sources, as_tuples=False)`: Batch process documents → yields Docs
- `_result_to_doc()`: Convert Docling document to spaCy Doc (internal)
- `_texts_to_doc()`: Tokenize and create layout spans (internal)
- `get_pages()`: Extension getter for `Doc._.pages`
- `get_tables()`: Extension getter for `Doc._.tables`
- `get_heading()`: Extension getter for `Span._.heading`

### 2. Data Types (types.py)

All dataclasses have `from_dict()` factory methods for deserialization.

```python
@dataclass
class PageLayout:
    page_no: int    # 1-indexed page number
    width: float    # Page width in pixels
    height: float   # Page height in pixels

@dataclass
class DocLayout:
    pages: list[PageLayout]

@dataclass
class SpanLayout:
    x: float        # Horizontal offset (bounding box)
    y: float        # Vertical offset (bounding box)
    width: float    # Bounding box width
    height: float   # Bounding box height
    page_no: int    # Page number

@dataclass
class Attrs:
    # Configurable attribute names for spaCy extensions
    doc_layout: str = "layout"
    doc_pages: str = "pages"
    doc_tables: str = "tables"
    doc_markdown: str = "markdown"
    span_layout: str = "layout"
    span_data: str = "data"
    span_heading: str = "heading"
    span_group: str = "layout"
```

### 3. Utilities (util.py)

**Serialization Functions:**
- `encode_obj()` / `decode_obj()`: Serialize/deserialize dataclasses
- `encode_df()` / `decode_df()`: Serialize/deserialize pandas DataFrames
- `get_bounding_box()`: Convert Docling bounding boxes to SpanLayout

**Important**: Handles coordinate origin transformations (BOTTOMLEFT ↔ TOPLEFT)

### 4. Extension Attributes System

spaCy's extension system is heavily used. Extensions are registered in `spaCyLayout.__init__()`.

**On Doc objects:**
- `Doc._.layout` → `DocLayout` (document-level layout features)
- `Doc._.pages` → `list[tuple[PageLayout, list[Span]]]` (pages and spans)
- `Doc._.tables` → `list[Span]` (all table spans)
- `Doc._.markdown` → `str` (markdown representation)

**On Span objects:**
- `Span._.layout` → `SpanLayout | None` (bounding box data)
- `Span._.data` → `DataFrame | None` (table data)
- `Span._.heading` → `Span | None` (closest heading)
- `Span.label_` → `str` (section type: "text", "title", "section_header", etc.)
- `Span.id` → `int` (running index of layout span)

**SpanGroup:**
- `Doc.spans["layout"]` → All layout spans

---

## Development Workflow

### Setup

```bash
# Clone repository
git clone https://github.com/explosion/spacy-layout.git
cd spacy-layout

# Install in editable mode with dev dependencies
pip install -e .
pip install pytest

# Or install all from requirements.txt
pip install -r requirements.txt
```

### Running Tests

```bash
# Run all tests
python -m pytest tests

# Run specific test
python -m pytest tests/test_general.py::test_general

# Run with verbose output
python -m pytest tests -v

# Run tests matching pattern
python -m pytest tests -k "table"
```

### Testing Conventions

**Framework**: pytest

**Fixtures** (tests/test_general.py:25-32):
- `nlp`: Returns `spacy.blank("en")`
- `span_labels`: Returns all valid `DocItemLabel` values

**Test Data**: Located in `tests/data/`
- Use `PDF_SIMPLE`, `PDF_STARCRAFT`, `PDF_TABLE`, etc. (defined at top of test file)
- Test both file paths and bytes input

**Parametrized Tests**: Used extensively for testing multiple inputs
```python
@pytest.mark.parametrize("path", [PDF_STARCRAFT, PDF_SIMPLE, PDF_SIMPLE_BYTES])
def test_general(path, nlp, span_labels):
    # ...
```

**Assertion Patterns**:
- Check types: `isinstance(doc._.layout, DocLayout)`
- Check DataFrame equality: `assert_frame_equal(df1, df2)`
- Access extension attributes: `doc._.get(layout.attrs.doc_layout)`

### Building and Publishing

```bash
# Build distribution packages
python -m build

# Publish to PyPI (automated via GitHub Actions on release)
# Manual: python -m twine upload dist/*
```

**CI/CD Pipeline**:
- **Test workflow**: Runs on push to main and PRs (ignores .md files)
  - Python 3.10 on Ubuntu
  - Installs in editable mode
  - Runs pytest
- **Publish workflow**: Triggers on GitHub releases
  - Uses trusted publishing (OIDC)
  - Automatically publishes to PyPI

---

## Code Style and Conventions

### Python Style

**No formal linters/formatters configured**, but code follows these patterns:

- **Type hints**: Used throughout with `TYPE_CHECKING` guards for imports
- **Private methods**: Prefixed with `_` (e.g., `_result_to_doc()`)
- **String types**: Use modern union syntax (`str | None` not `Optional[str]`)
- **Imports**: Organized but no isort configuration
  - Standard library
  - Third-party (spacy, docling, pandas)
  - Local (relative imports from package)

### Naming Conventions

- **Classes**: PascalCase (`spaCyLayout`, `DocLayout`)
- **Functions/methods**: snake_case (`get_bounding_box`, `encode_obj`)
- **Constants**: UPPER_SNAKE_CASE (`TABLE_PLACEHOLDER`, `TABLE_ITEM_LABELS`)
- **Type variables**: _PrefixedPascalCase (`_AnyContext`)

### Docstrings

Present for main public methods but not comprehensive. Follow this pattern when adding:
```python
def method(self, arg: type) -> return_type:
    """Brief description.

    arg: Description of argument.
    return_type: Description of return value.
    """
```

### Type Annotations

Always include type hints:
```python
def pipe(
    self,
    sources: Iterable[str | Path | bytes] | Iterable[tuple[str | Path | bytes, _AnyContext]],
    as_tuples: Literal[False] = False,
) -> Iterator[Doc]:
```

Use `overload` decorator for methods with different return types based on parameters.

---

## Common Tasks and Commands

### Adding New Features

1. **Identify the appropriate module**:
   - Document-level logic → `layout.py`
   - New data structures → `types.py`
   - Serialization helpers → `util.py`

2. **Add type hints**: Always include comprehensive type annotations

3. **Register extensions** (if needed):
   ```python
   # In spaCyLayout.__init__()
   if not Doc.has_extension(self.attrs.new_attr):
       Doc.set_extension(self.attrs.new_attr, default=None, getter=self.get_new_attr)
   ```

4. **Write tests**: Add test function to `test_general.py`

5. **Update README.md**: Document new features in API section

### Working with Serialization

**Critical**: Custom dataclasses and DataFrames use msgpack with custom encoders/decoders.

**Registration** (layout.py:36-40):
```python
srsly.msgpack_encoders.register("spacy-layout.dataclass", func=encode_obj)
srsly.msgpack_decoders.register("spacy-layout.dataclass", func=decode_obj)
srsly.msgpack_encoders.register("spacy-layout.dataframe", func=encode_df)
srsly.msgpack_decoders.register("spacy-layout.dataframe", func=decode_df)
```

**Usage**:
```python
from spacy.tokens import DocBin

# Serialize
docs = layout.pipe(["one.pdf", "two.pdf"])
doc_bin = DocBin(docs=docs, store_user_data=True)  # ⚠️ store_user_data=True required!
doc_bin.to_disk("./file.spacy")

# Deserialize
layout = spaCyLayout(nlp)  # ⚠️ Initialize first to register extensions!
doc_bin = DocBin(store_user_data=True).from_disk("./file.spacy")
docs = list(doc_bin.get_docs(nlp.vocab))
```

### Debugging Document Processing

**Common issues**:
1. **Empty text spans**: Check separator logic
2. **Missing layout data**: Ensure Docling parsed bounding boxes
3. **Table extraction fails**: Verify table labels in `TABLE_ITEM_LABELS`
4. **Coordinate issues**: Check `CoordOrigin` transformation in `get_bounding_box()`

**Debug pattern**:
```python
doc = layout(path)
print(f"Total tokens: {len(doc)}")
print(f"Layout spans: {len(doc.spans['layout'])}")
for span in doc.spans["layout"]:
    print(f"{span.label_:20} {span.start:4}-{span.end:4} {span.text[:50]}")
    if span._.layout:
        print(f"  BBox: ({span._.layout.x}, {span._.layout.y})")
```

---

## Important Constraints and Requirements

### Python Version

**Minimum: Python 3.10**

Reason: Uses modern type syntax (`str | None`, `list[Type]`) and Docling requires 3.10+.

### Dependencies

**Runtime** (requirements.txt):
```
spacy>=3.7.5          # NLP framework and data structures
docling>=2.5.2        # Document parsing engine
pandas                # Table data representation
srsly                 # Efficient serialization
```

**Development**:
```
pytest                # Testing framework
```

### Platform Support

Officially supports:
- Linux (POSIX)
- macOS
- Windows

### Critical Design Constraints

1. **Tokenization dependency**: Requires a spaCy `nlp` object for tokenization
   - Must be initialized before creating `spaCyLayout`
   - Can be `spacy.blank("en")` or a full model like `en_core_web_trf`

2. **Extension registration**: Must initialize `spaCyLayout` before deserializing Docs
   - Extensions are registered during `__init__()`
   - Required for `Doc._.layout`, `Span._.data`, etc.

3. **Separator handling**: Default `"\n\n"` separator not included in spans
   - Set to `None` to disable separators
   - Custom separators must be single strings

4. **Table representation**: Default `"TABLE"` placeholder in Doc.text
   - Customize with `display_table` parameter
   - Can be function or string

---

## Integration Points

### Docling Integration

**Entry point**: `DocumentConverter` from docling

**Key types imported**:
- `DoclingDocument`: Main document representation
- `DocumentStream`: For bytes input
- `DocItemLabel`: Label types (text, title, section_header, table, etc.)
- `BoundingBox`, `CoordOrigin`: Layout geometry

**Conversion flow**:
```
Input (PDF/DOCX) → DocumentConverter → DoclingDocument → _result_to_doc() → spaCy Doc
```

### spaCy Integration

**Requirements**:
- `Language` object (nlp) for tokenization
- `Doc`, `Span`, `SpanGroup` for data structures
- Extension attribute system for custom data

**Usage patterns**:
```python
# Basic tokenization
nlp = spacy.blank("en")
layout = spaCyLayout(nlp)
doc = layout("file.pdf")

# With NLP pipeline
nlp = spacy.load("en_core_web_trf")
layout = spaCyLayout(nlp)
doc = layout("file.pdf")
doc = nlp(doc)  # Apply pipeline for POS, NER, etc.
```

### pandas Integration

**Table data**: Extracted tables converted to DataFrames

**Access pattern**:
```python
for table in doc._.tables:
    df = table._.data  # pandas DataFrame
    print(df.columns.tolist())
    print(df.head())
```

---

## Tips for AI Assistants

### When Making Changes

1. **Read first**: Always read files before editing (tool requirement)
2. **Test data**: Use existing test PDFs in `tests/data/` for testing
3. **Type safety**: Maintain type hints and fix any type errors
4. **Extension attributes**: Remember to register new extensions in `__init__()`
5. **Serialization**: Update encoders/decoders if adding serializable types
6. **Tests**: Add/update tests in `test_general.py` for any changes
7. **README**: Update API documentation in README.md if changing public API

### Understanding the Data Flow

```
1. Input: PDF/DOCX file path or bytes
   ↓
2. Docling DocumentConverter.convert()
   ↓
3. DoclingDocument (hierarchical document tree)
   ↓
4. Iterate tree: extract text items and tables
   ↓
5. _texts_to_doc(): Tokenize with nlp object
   ↓
6. Create Doc with words, spaces, layout spans
   ↓
7. Add extension attributes (layout, tables, pages, markdown)
   ↓
8. Return enriched spaCy Doc object
```

### Common Patterns

**Processing single document**:
```python
layout = spaCyLayout(nlp)
doc = layout("document.pdf")
```

**Batch processing**:
```python
paths = ["one.pdf", "two.pdf", "three.pdf"]
for doc in layout.pipe(paths):
    process(doc)
```

**With context (as_tuples)**:
```python
sources = [("one.pdf", {"id": 1}), ("two.pdf", {"id": 2})]
for doc, context in layout.pipe(sources, as_tuples=True):
    print(f"Doc {context['id']}: {len(doc)} tokens")
```

**Accessing layout information**:
```python
# Document level
print(doc._.layout.pages)

# Span level
for span in doc.spans["layout"]:
    if span.label_ == "section_header":
        print(f"Heading: {span.text}")
        print(f"Position: {span._.layout.x}, {span._.layout.y}")
```

**Working with tables**:
```python
for table_span in doc._.tables:
    df = table_span._.data
    print(f"Table at tokens {table_span.start}-{table_span.end}")
    print(df.to_markdown())
```

### Testing Guidelines

**Always**:
- Use fixtures (`nlp`, `span_labels`)
- Test with existing test data files
- Use parametrized tests for multiple inputs
- Check both positive and edge cases

**Test structure**:
```python
def test_feature(nlp):
    layout = spaCyLayout(nlp)
    doc = layout(PDF_SIMPLE)
    # Assertions
    assert expected_behavior
```

### Performance Considerations

- Use `.pipe()` for batch processing (more efficient than loop + `__call__()`)
- Docling conversion is CPU-intensive (PDF parsing)
- Consider caching serialized Docs for reuse
- Large PDFs can be memory-intensive

### Debugging Checklist

If something isn't working:

1. **Check Python version**: Must be 3.10+
2. **Check nlp object**: Must be initialized spaCy Language object
3. **Check file format**: Verify Docling supports the input format
4. **Check extensions**: Ensure `spaCyLayout` was initialized before deserializing
5. **Check separator**: Verify separator logic if spans look wrong
6. **Check coordinates**: Verify bounding box origin if layout seems flipped
7. **Check labels**: Verify span labels match expected DocItemLabel values

---

## Version History and Compatibility

**Current Version**: 0.0.12 (Beta)

**Status**: Active development, breaking changes possible

**Recent changes**:
- Added `as_tuples` support to `pipe()` method
- Added `Doc._.tables` shortcut
- Fixed document index table handling

**spaCy compatibility**: Requires spaCy >=3.7.5
**Docling compatibility**: Requires Docling >=2.5.2

---

## Resources

- **GitHub**: https://github.com/explosion/spacy-layout
- **PyPI**: https://pypi.org/project/spacy-layout/
- **spaCy docs**: https://spacy.io
- **Docling docs**: https://ds4sd.github.io/docling/
- **Blog post**: ["From PDFs to AI-ready structured data"](https://explosion.ai/blog/pdfs-nlp-structured-data)

---

## Contact and Contributions

**Maintainer**: Explosion AI (contact@explosion.ai)

**Contributors**: 5 developers including Ines Montani, svlandeg, magdaaniol, mkessy, William Mattingly

**Contributing**: Submit pull requests to https://github.com/explosion/spacy-layout/pulls

**Issues**: Report at https://github.com/explosion/spacy-layout/issues

---

*Last updated: 2025-11-18*
*Document version: 1.0*
