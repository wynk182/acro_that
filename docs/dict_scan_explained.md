# DictScan Explained: Text Traversal in Action

## The Big Picture

`DictScan` is a module that appears complicated at first glance, but it's fundamentally just **text traversal**—walking through PDF files character by character to find and extract dictionary structures.

This document explains how each function in `DictScan` works and why the text-traversal approach is both powerful and straightforward.

## Core Principle

**PDF dictionaries are text patterns.** They use `<<` and `>>` as delimiters, just like how programming languages use `{` and `}` or `[` and `]`. Once you recognize this, parsing becomes a matter of tracking depth and matching delimiters.

## Function-by-Function Guide

### `strip_stream_bodies(pdf)`

**Purpose:** Remove binary stream data that would confuse text parsing.

**How it works:**
- Finds all `stream...endstream` blocks using regex
- Replaces the binary content with a placeholder
- Preserves the stream structure markers

**Why:** Streams can contain arbitrary binary data (compressed images, fonts, etc.) that would break our text-based parsing. We strip them out since we're only interested in the dictionary structure.

```ruby
pdf.gsub(/stream\r?\n.*?endstream/mi) { "stream\nENDSTREAM_STRIPPED\nendstream" }
```

This is regex-based, but it's necessary preprocessing before we can safely do text traversal.

---

### `each_dictionary(str)`

**Purpose:** Iterate through all dictionaries in a string.

**Algorithm:**
1. Find the first `<<` at position `i`
2. Initialize depth counter to 0
3. Scan forward:
   - If we see `<<`, increment depth
   - If we see `>>`, decrement depth
   - If depth reaches 0, we've found a complete dictionary
4. Yield the dictionary substring
5. Continue from where we left off

**Example:**
```
Input: "<< /A 1 >> << /B 2 >>"
  i=0: find "<<"
  depth=1, scan forward
  see ">>", depth=0 → found "<< /A 1 >>"
  yield and continue from i=11
  i=11: find "<<"
  depth=1, scan forward
  see ">>", depth=0 → found "<< /B 2 >>"
  yield and continue
```

**Why it works:** This is classic **bracket matching**. No PDF-specific knowledge needed—just counting delimiters.

---

### `unescape_literal(s)`

**Purpose:** Decode PDF escape sequences in literal strings.

**PDF escapes:**
- `\n` → newline
- `\r` → carriage return
- `\t` → tab
- `\b` → backspace
- `\f` → form feed
- `\\(` → literal `(`
- `\\)` → literal `)`
- `\123` → octal character (up to 3 digits)

**Algorithm:** Character-by-character scan:
1. If we see `\`, look ahead one character
2. Map escape sequences to actual characters
3. Handle octal sequences (1-3 digits)
4. Otherwise, copy character as-is

**Why it works:** This is standard escape sequence handling, identical to how many programming languages handle string literals.

---

### `decode_pdf_string(token)`

**Purpose:** Decode a PDF string token into a Ruby string.

**PDF string types:**
1. **Literal:** `(Hello World)` or `(Hello\nWorld)`
2. **Hex:** `<48656C6C6F>` or `<FEFF00480065006C006C006F>`

**Algorithm:**
1. Check if token starts with `(` → literal string
   - Extract content between parentheses
   - Unescape using `unescape_literal`
   - Check for UTF-16BE BOM (`FE FF`)
   - Decode accordingly
2. Check if token starts with `<` → hex string
   - Remove spaces, pad if odd length
   - Convert hex to bytes
   - Check for UTF-16BE BOM
   - Decode accordingly
3. Otherwise, return as-is (name, number, reference, etc.)

**Why it works:** PDF strings have well-defined formats. We just pattern-match on the delimiters and decode accordingly.

---

### `encode_pdf_string(val)`

**Purpose:** Encode a Ruby value into a PDF string token.

**Handles:**
- `true` → `"true"`
- `false` → `"false"`
- `Symbol` → `"/symbol_name"`
- `String`:
  - ASCII-only → literal string `(value)`
  - Non-ASCII → hex string with UTF-16BE encoding

**Why it works:** Reverse of `decode_pdf_string`—we know the target format and encode accordingly.

---

### `value_token_after(key, dict_src)`

**Purpose:** Extract the value token that follows a key in a dictionary.

**This is the heart of text traversal.** Here's how it works:

1. **Find the key:**
   ```ruby
   match = dict_src.match(%r{#{Regexp.escape(key)}(?=[\s(<\[/])})
   ```
   Use regex to ensure the key is followed by a delimiter (whitespace, `(`, `<`, `[`, or `/`). This prevents partial matches.

2. **Skip whitespace:**
   ```ruby
   i += 1 while i < dict_src.length && dict_src[i] =~ /\s/
   ```

3. **Switch on the next character:**
   - **`(` → Literal string:**
     - Track depth of parentheses
     - Handle escaped characters (skip `\` and next char)
     - Match closing `)` when depth returns to 0
   - **`<` → Hex string or dictionary:**
     - If `<<` → return `"<<"` (nested dictionary marker)
     - Otherwise, find matching `>`
   - **`[` → Array:**
     - Track depth of brackets
     - Match closing `]` when depth returns to 0
   - **`/` → PDF name:**
     - Extract until whitespace or delimiter
   - **Otherwise → Atom:**
     - Extract until whitespace or delimiter (number, reference, boolean, etc.)

**Why it works:** PDF has well-defined token syntax. Each value type has distinct delimiters, so we can pattern-match on the first character and extract accordingly.

**Example:**
```
Dict: "<< /V (Hello) /R [1 2 3] >>"
value_token_after("/V", dict):
  → Finds "/V" at position 3
  → Skips space
  → Next char is "("
  → Extracts "(Hello)" using paren matching
  → Returns "(Hello)"
```

---

### `replace_key_value(dict_src, key, new_token)`

**Purpose:** Replace a key's value in a dictionary string.

**Algorithm:**
1. Find the key using pattern matching
2. Extract the existing value token using `value_token_after`
3. Find exact byte positions:
   - Key start/end
   - Value start/end
4. Replace using string slicing: `before + new_token + after`
5. Verify dictionary is still valid (contains `<<` and `>>`)

**Why it works:** We use **precise byte positions** rather than regex replacement. This:
- Preserves exact formatting (whitespace, etc.)
- Avoids regex edge cases
- Is deterministic and safe

**Example:**
```
Input:  "<< /V (Old) /X 1 >>"
        before: "<< /V "
        value:  "(Old)"
        after:  " /X 1 >>"
Output: "<< /V (New) /X 1 >>"
```

---

### `upsert_key_value(dict_src, key, token)`

**Purpose:** Insert a key-value pair if the key doesn't exist.

**Algorithm:**
- If key not found, insert after the opening `<<`
- Uses simple string substitution: `"<<#{key} #{token}"`

**Why it works:** Simple string manipulation when we know the key doesn't exist.

---

### `remove_ref_from_array(array_body, ref)` and `add_ref_to_array(array_body, ref)`

**Purpose:** Manipulate object references in PDF arrays.

**Algorithm:**
- Use regex/gsub to find and replace reference patterns: `"5 0 R"`
- Handle edge cases: empty arrays, spacing

**Why it works:** Object references have a fixed format (`num gen R`), so we can pattern-match and replace.

---

## Why This Approach Works

### 1. PDF Structure is Text-Based

PDF dictionaries, arrays, strings, and references are all defined using text syntax. No binary parsing needed for structure.

### 2. Delimiters Are Unique

Each PDF value type has distinct delimiters:
- Dictionaries: `<<` `>>`
- Arrays: `[` `]`
- Literal strings: `(` `)`
- Hex strings: `<` `>`
- Names: `/`
- References: `R`

We can pattern-match on these to extract values.

### 3. Depth Tracking is Simple

Nested structures (dictionaries, arrays, strings) can be parsed by tracking depth—increment on open, decrement on close. Standard algorithm from compiler theory.

### 4. Position-Based Replacement is Safe

When modifying dictionaries, we use exact byte positions rather than regex replacement. This:
- Preserves formatting
- Avoids edge cases
- Is predictable

### 5. No Full Parser Needed

We don't need to:
- Build an AST
- Validate the entire PDF structure
- Handle all PDF features

We only need to:
- Find dictionaries
- Extract values
- Replace values
- Preserve structure

This is a **minimal parser** that does exactly what we need.

## Common Patterns

### Pattern 1: Find and Extract

```ruby
# Find all dictionaries
each_dictionary(pdf_text) do |dict|
  # Extract a value
  value_token = value_token_after("/V", dict)
  value = decode_pdf_string(value_token)
  puts value
end
```

### Pattern 2: Find and Replace

```ruby
# Get dictionary
dict = "<< /V (Old) >>"

# Replace value
new_dict = replace_key_value(dict, "/V", "(New)")

# Result: "<< /V (New) >>"
```

### Pattern 3: Encode and Insert

```ruby
# Prepare new value
token = encode_pdf_string("Hello")

# Insert into dictionary
dict = upsert_key_value(dict, "/V", token)
```

## Performance Considerations

**Why this is fast:**
- No AST construction
- No full PDF parsing
- Direct string manipulation
- Minimal memory allocation

**Trade-offs:**
- Doesn't validate entire PDF structure
- Assumes dictionaries are well-formed
- Stream stripping is regex-based (could be optimized)

## Conclusion

`DictScan` appears complicated because it handles many edge cases and value types, but the **core approach is elegantly simple**:

1. PDF dictionaries are text patterns
2. Parse them with character-by-character traversal
3. Track depth for nested structures
4. Use precise positions for replacement

No magic, no complex parsers—just careful text traversal with attention to PDF syntax rules.

The complexity you see is:
- **Edge case handling** (escaping, nesting, encoding)
- **Safety checks** (verification, error handling)
- **Support for multiple value types** (strings, arrays, dictionaries, references)

But the **fundamental algorithm** is straightforward: find delimiters, track depth, extract substrings, replace substrings.

