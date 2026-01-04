---
description: 'Markdown documentation standards for consistency and readability'
applyTo: '**/*.md'
---

# Markdown Documentation Standards

Guidelines for creating consistent, readable, and maintainable markdown documentation.

## Purpose and Scope

Apply markdown formatting standards to all documentation files to ensure consistent structure, accessibility, and maintainability across the project.

## Document Structure

### Start with H2, Not H1
Omit H1 headings (`#`) as document title is generated from filename or frontmatter.

**Rationale**: Automated tooling generates H1 from metadata, duplicate H1 breaks document hierarchy.

### Use Hierarchical Headings
Progress from H2 (`##`) to H3 (`###`) without skipping levels.
Restructure content if H4 or deeper levels are needed.

**Rationale**: Flat heading structure improves readability and screen reader navigation.

```markdown
<!-- Recommended -->
## Main Section
### Subsection
### Another Subsection

<!-- Not Recommended -->
## Main Section
#### Deep Subsection (skips H3)
```

### Separate Sections with Blank Lines
Place one blank line before and after headings, code blocks, lists, and tables.

```markdown
<!-- Recommended -->
Previous paragraph.

## New Section

First paragraph of section.

<!-- Not Recommended -->
Previous paragraph.
## New Section
First paragraph of section.
```

## Lists

### Use Consistent List Markers
Use `-` for unordered lists and `1.` for ordered lists.
Indent nested lists with two spaces.

```markdown
<!-- Recommended -->
- First item
- Second item
  - Nested item
  - Another nested item
- Third item

1. First step
2. Second step
   1. Sub-step
3. Third step

<!-- Not Recommended -->
* Mixed markers
- In same list
  * Creates confusion
```

### Break Long List Items
Limit list item line length to 80 characters.
Continue on next line with proper indentation.

```markdown
<!-- Recommended -->
- This is a long list item that exceeds 80 characters
  and continues on the next line with proper indentation

<!-- Not Recommended -->
- This is a very long list item that goes on and on without breaking which makes it hard to read in source form
```

## Formatting and Structure

Follow these guidelines for formatting and structuring your markdown content:

- **Headings**: Use `##` for H2 and `###` for H3. Ensure that headings are used in a hierarchical manner. Recommend restructuring if content includes H4, and more strongly recommend for H5.
- **Lists**: Use `-` for bullet points and `1.` for numbered lists. Indent nested lists with two spaces.
- **Code Blocks**: Use triple backticks (`) to create fenced code blocks. Specify the language after the opening backticks for syntax highlighting (e.g., `csharp).
- **Links**: Use `[link text](URL)` for links. Ensure that the link text is descriptive and the URL is valid.
- **Images**: Use `![alt text](image URL)` for images. Include a brief description of the image in the alt text.
- **Tables**: Use `|` to create tables. Ensure that columns are properly aligned and headers are included.
- **Line Length**: Break lines at 80 characters to improve readability. Use soft line breaks for long paragraphs.
- **Whitespace**: Use blank lines to separate sections and improve readability. Avoid excessive whitespace.

## Validation Requirements

Ensure compliance with the following validation requirements:

- **Front Matter**: Include the following fields in the YAML front matter:

  - `post_title`: The title of the post.
  - `author1`: The primary author of the post.
  - `post_slug`: The URL slug for the post.
  - `microsoft_alias`: The Microsoft alias of the author.
  - `featured_image`: The URL of the featured image.
  - `tags`: The tags for the post.
  - `ai_note`: Indicate if AI was used in the creation of the post.
  - `summary`: A brief summary of the post. Recommend a summary based on the content when possible.
  - `post_date`: The publication date of the post.

- **Content Rules**: Ensure that the content follows the markdown content rules specified above.
- **Formatting**: Ensure that the content is properly formatted and structured according to the guidelines.
- **Validation**: Run the validation tools to check for compliance with the rules and guidelines.
