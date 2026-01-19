# Render Step Summary Action

Composite action that renders a beautiful markdown summary to GitHub Step Summary.

## Inputs

| Input | Description | Required |
|-------|-------------|----------|
| `title` | Title of the summary section | Yes |
| `lines` | Multiline string with bullet points (markdown supported) | Yes |

## Features

- Writes formatted markdown to `$GITHUB_STEP_SUMMARY`
- Supports markdown formatting in lines
- Automatically adds section title

## Usage

```yaml
- name: Render Summary
  uses: ./actions/render-summary
  with:
    title: "Terraform Pipeline Summary"
    lines: |
      - **Format Check**: ✅ Passed
      - **Validate**: ✅ Passed
      - **Plan**: ✅ Generated
      - **Status**: ✅ All checks passed
```
