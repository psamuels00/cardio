# Cardio Application
Blood Pressure & Heart Rate Monitoring

## Document purpose and status

This document is under construction.  It will eventually include commands
and other details useful for setting up a project like this, or for
ongoing maintenance.

## Format Markdown documentation

    brew install prettier

    format_markdown_file() {
        local file="$1"

        printf "%10s: %s\n" prettier 'make it look nice'
        prettier -w "$file" > /dev/null

        printf "%10s: %s\n" sed 'add blank line before sections'
        sed -i '' -E $'/^#{2,3}[[:space:]]/i\\\n' "$file"

        printf "%10s: %s\n" doctoc 'update TOC'
        npx doctoc "$file" > /dev/null
    }

    reformat_markdown_files() {
        format_markdown_file docs/requirements.md
    }

    reformat_markdown_files

