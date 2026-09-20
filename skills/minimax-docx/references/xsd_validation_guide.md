# XSD Validation Guide

## Running Validation

Run from the skill directory.

```bash
# Structural validation against the WML subset schema
dotnet run --project scripts/dotnet/MiniMaxAIDocx.Cli -- validate --input input.docx --xsd assets/xsd/wml-subset.xsd

# Business rules (REQUIRED before Scenario C delivery)
dotnet run --project scripts/dotnet/MiniMaxAIDocx.Cli -- validate --input input.docx --business

# Gate-check the output against the template DOCX
dotnet run --project scripts/dotnet/MiniMaxAIDocx.Cli -- validate --input output.docx --gate-check template.docx
```

`--xsd` takes a single schema; repeating the flag is a CLI error, not a way to combine
schemas. The three switches are independent and combine in one call:

```bash
dotnet run --project scripts/dotnet/MiniMaxAIDocx.Cli -- validate \
  --input output.docx --xsd assets/xsd/wml-subset.xsd --business --gate-check template.docx
```

---

## What wml-subset.xsd Covers

The subset schema validates the most common WordprocessingML elements:

| Area | Elements Validated |
|------|--------------------|
| Document structure | `w:document`, `w:body`, `w:sectPr` |
| Paragraphs | `w:p`, `w:pPr`, `w:r`, `w:rPr`, `w:t` |
| Tables | `w:tbl`, `w:tblPr`, `w:tblGrid`, `w:tr`, `w:tc` |
| Styles | `w:styles`, `w:style`, `w:docDefaults` |
| Lists | `w:numbering`, `w:abstractNum`, `w:num` |
| Headers/Footers | `w:hdr`, `w:ftr` |
| Track Changes | `w:ins`, `w:del`, `w:rPrChange`, `w:pPrChange` |
| Comments | `w:comment`, `w:commentRangeStart`, `w:commentRangeEnd` |

### What It Does NOT Cover

- DrawingML elements (`a:`, `pic:`, `wp:`) — image/shape internals
- VML elements (`v:`, `o:`) — legacy shapes
- Math elements (`m:`) — equations
- Extended namespaces (`w14`, `w15`, `w16*`) — vendor extensions
- Custom XML data parts
- Relationship targets and content types — `wml-subset.xsd` accepts the `r:` attributes
  but does not check that the relationship they name actually exists

`wml-subset.xsd` imports two companion subsets, `relationships-subset.xsd` and
`xml-subset.xsd`, so that `r:id` and `xml:space` resolve. Both live beside it in
`assets/xsd/` and are resolved locally, so validation needs no network access. Keep the
three files together if you copy the schema elsewhere.

---

## Interpreting Errors

### Element Ordering Error

```
ERROR: Element 'w:jc' is not expected at this position.
Expected: w:spacing, w:ind, w:contextualSpacing, ...
Location: /word/document.xml, line 45
```

**Cause**: Child elements are in wrong order. See `references/openxml_element_order.md`.
**Fix**: Reorder children to match schema sequence.

### Missing Required Element

```
ERROR: Element 'w:tbl' missing required child 'w:tblPr'.
Location: /word/document.xml, line 102
```

**Cause**: A required child element is absent.
**Fix**: Add the missing element. Tables require both `w:tblPr` and `w:tblGrid`.

### Invalid Attribute Value

```
ERROR: Attribute 'w:val' has invalid value 'middle'.
Expected: 'left', 'center', 'right', 'both', 'distribute'
Location: /word/document.xml, line 78
```

**Cause**: An attribute value is not in the allowed enumeration.
**Fix**: Use one of the valid values listed in the error.

### Unexpected Element

```
ERROR: Element 'w:customTag' is not expected.
Location: /word/document.xml, line 200
```

**Cause**: An element the subset schema does not declare. This is *not* proof the
document is wrong — `wml-subset.xsd` is a curated subset, so a perfectly valid
WordprocessingML element it never declared reports the same way. Vendor extensions
(`w14`/`w15`/`w16*`) are the common case.
**Fix**: Confirm against `references/openxml_element_order.md` and the ISO 29500 element
list before treating it as a defect. If the element is legitimate and common, add it to
`wml-subset.xsd` rather than working around it.

---

## Business Rules

Business rules are enforced by the `--business` switch, implemented in
`MiniMaxAIDocx.Core/Validation/BusinessRuleValidator.cs` — no schema is loaded:

| Rule | What It Checks |
|------|---------------|
| Required styles | `Normal`, `Heading1`-`Heading3`, `TableGrid` must exist in `styles.xml` |
| Font consistency | `w:docDefaults` fonts match expected values |
| Margin ranges | Page margins within acceptable range (720-2160 DXA) |
| Page size | Must be A4 or Letter |
| Heading hierarchy | No gaps (e.g., H1 → H3 without H2) |
| Style chain | `w:basedOn` references must resolve to existing styles |

> **`assets/xsd/business-rules.xsd` and `assets/xsd/aesthetic-rules.xsd` are not
> `--xsd` targets.** Both declare constraints on individual elements (`w:pgSz`,
> `w:pgMar`, `w:sz`, `w:color`, ...) and have no `w:document` root, so passing either
> to `--xsd` fails with `The 'w:document' element is not declared`. They document the
> thresholds the `--business` / gate-check code applies; the code is what runs.

### Extending Business Rules

To add project-specific rules, extend `BusinessRuleValidator` and document the
threshold here. The fragment schemas use `xs:restriction` to record a rule, e.g.:

```xml
<!-- Require minimum 1-inch margins -->
<xs:element name="pgMar">
  <xs:complexType>
    <xs:attribute name="top" type="xs:integer">
      <xs:restriction>
        <xs:minInclusive value="1440" />
      </xs:restriction>
    </xs:attribute>
  </xs:complexType>
</xs:element>
```

---

## Gate-Check: Scenario C Hard Gate

In Scenario C (Apply Template), the output document **MUST** pass the gate-check before delivery:

```
1. Apply template  →  output.docx
2. Gate-check      →  dotnet run --project scripts/dotnet/MiniMaxAIDocx.Cli -- \
                         validate --input output.docx --gate-check template.docx
3. PASS?           →  Deliver to user
4. FAIL?           →  Fix issues, re-validate, repeat until PASS
```

The gate-check compares the output against the template's style set, page size and
margins, default font, and heading size hierarchy, so it needs the template DOCX.

**This is a hard gate.** A document that fails the gate-check is NOT deliverable, even
if it opens correctly in Word.

---

## False Positives

### Vendor Extensions

Elements from extended namespaces (`w14`, `w15`, `w16*`) are not in the subset schema and may trigger warnings:

```
WARNING: Element '{http://schemas.microsoft.com/office/word/2010/wordml}shadow' is not expected.
```

These are generally safe to ignore — they are Microsoft extensions for newer features (e.g., advanced text effects, comment extensions).

### Markup Compatibility

Documents may contain `mc:AlternateContent` blocks with fallback content. The subset schema may not recognize the `mc:` namespace processing. These are safe if the document opens correctly in Word.

### Recommended Approach

1. Run validation
2. Treat **errors** as must-fix
3. Review **warnings** — ignore known vendor extensions, investigate unknown elements
4. After fixing errors, re-validate to confirm
