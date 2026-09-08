# Use `git commit -F <file>` for messages with shell-sensitive characters

Multi-line `git commit -m "$(cat <<'EOF' ... EOF)"` can fail in harness-executed
shells with "unexpected EOF while looking for matching quote" when the message
body contains apostrophes or brace/quote-heavy text (e.g. JSON snippets like
{"blocks": [...]}), because the outer command string is itself parsed before the
heredoc runs.

Reliable pattern:

1. Write the message to a file first (Create tool or `cat > /tmp/msg.txt`).
2. `git commit -F /tmp/msg.txt`

This avoids all nested quoting/interpolation issues and preserves the message
verbatim, including em dashes and JSON fragments.

Tags: git, commit, heredoc, quoting, factory, harness
Date: 2026-09-08
