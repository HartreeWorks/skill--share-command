---
name: share-command
description: This skill should be used when the user asks to "share a command", "make a command public", "publish a command", "share my slash command", or mentions making a Claude Code command available publicly. Publishes user-defined commands from ~/.claude/commands/ to the HartreeWorks/claude-commands repository.
---

# Share Command

This skill publishes a Claude Code user-defined command to the public HartreeWorks/claude-commands repository.

## What it does

1. Validates the command file exists in `~/.claude/commands/`
2. **CRITICAL: Security & privacy review** - checks for credentials and private information
3. Creates the public repo if it doesn't exist (first time only)
4. Copies the command to the repo
5. Updates the README index
6. Commits and pushes to GitHub

## Prerequisites

- GitHub CLI (`gh`) must be installed and authenticated
- The command must exist as a `.md` file in `~/.claude/commands/`

## Workflow

When the user asks to share a command, follow these steps:

### Step 1: Validate the command

```bash
COMMAND_NAME="{command-name}"  # Without .md extension
COMMANDS_DIR=~/.claude/commands
COMMAND_PATH="$COMMANDS_DIR/$COMMAND_NAME.md"

# Check command file exists
ls "$COMMAND_PATH"
```

If the command doesn't exist, list available commands:

```bash
ls "$COMMANDS_DIR"/*.md
```

### Step 2: Security & privacy review (CRITICAL)

**This step is mandatory. Do NOT proceed to publishing without completing this review.**

#### 2a: Read and review the command file

```bash
cat "$COMMAND_PATH"
```

**Look for:**

| Type | Examples | Replacement |
|------|----------|-------------|
| Client company names | Acme Corp, 80,000 Hours | HartreeWorks LTD |
| Real people's names | John Smith | Alice, Bob |
| Email addresses | john@client.com | alice@example.com |
| API keys or tokens | sk-xxx, token-xxx | {YOUR_API_KEY} |
| Private paths | /Users/john/secret/ | /path/to/ |
| Phone numbers | Real numbers | +44 20 1234 5678 |

**Check the allowed-tools field** - ensure no private script paths are exposed that shouldn't be.

#### 2b: Report findings and get approval

Present findings clearly and use AskUserQuestion:
```
question: "I've completed the security review. Should I make any suggested changes and proceed?"
header: "Review"
options:
  - label: "Apply changes & proceed"
    description: "Make all suggested replacements and continue publishing"
  - label: "Show me the changes first"
    description: "Display the exact edits before applying"
  - label: "Stop - I'll review manually"
    description: "Abort so you can review and edit files yourself"
```

**Only proceed after user explicitly approves.**

### Step 3: Ensure the public repo exists

```bash
REPO_DIR="$COMMANDS_DIR/../skills/share-command/claude-commands"

# Check if we have a local clone
if [ -d "$REPO_DIR" ]; then
    cd "$REPO_DIR"
    git pull origin main
else
    # Check if remote repo exists
    if gh repo view HartreeWorks/claude-commands --json url 2>/dev/null; then
        # Clone existing repo
        gh repo clone HartreeWorks/claude-commands "$REPO_DIR"
    else
        # Create the repo for the first time
        mkdir -p "$REPO_DIR"
        cd "$REPO_DIR"
        git init

        # Create initial README
        cat > README.md << 'EOF'
# Claude Commands

A collection of user-defined slash commands for [Claude Code](https://claude.com/claude-code).

## Installation

Copy any `.md` file to your `~/.claude/commands/` directory. The command becomes available as `/{filename}`.

## Available commands

| Command | Description |
|---------|-------------|

## About

Created by [Peter Hartree](https://x.com/peterhartree). For updates, follow [AI Wow](https://wow.pjh.is), my AI uplift newsletter.

Find more Claude Code resources at [HartreeWorks](https://github.com/HartreeWorks).
EOF

        git add README.md
        git commit -m "Initial commit"

        # Create public repo
        gh repo create HartreeWorks/claude-commands --public --source=. --remote=origin --push
    fi
fi
```

### Step 4: Copy the command to the repo

```bash
cd "$REPO_DIR"
cp "$COMMAND_PATH" ./
```

### Step 5: Update the README index

Read the command's description from its YAML frontmatter:

```bash
# Extract description from frontmatter
DESCRIPTION=$(sed -n '/^---$/,/^---$/p' "$COMMAND_PATH" | grep "^description:" | sed 's/description: *//')
```

Edit `README.md` to add a row to the "Available commands" table in alphabetical order:

```markdown
| [{command-name}](./{command-name}.md) | {description} |
```

Keep the table sorted alphabetically by command name.

### Step 6: Commit and push

```bash
cd "$REPO_DIR"
git add .
git commit -m "Add $COMMAND_NAME command

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
git push origin main
```

### Step 7: Confirm success

Output:

```
✓ Command "/{command-name}" is now public!

Repository: https://github.com/HartreeWorks/claude-commands

Others can install it by copying the .md file to their ~/.claude/commands/ directory.
```

## Error handling

| Error | Cause | Solution |
|-------|-------|----------|
| Command not found | Typo in name | List available commands with `ls ~/.claude/commands/` |
| No description | Missing frontmatter | Add `description:` to the command's YAML frontmatter |
| Privacy review failed | User chose to stop | User reviews manually and runs share again |
| gh auth error | Not logged in | Run `gh auth login` |
| Push failed | Network issue | Retry with `git push` |

## Example usage

User: "Share the screenshot command"

1. Validate: `~/.claude/commands/screenshot.md` exists ✓
2. Security review:
   - Read command file
   - Check for private paths, API keys, personal info
   - Report findings and get approval
3. User approves
4. Ensure repo exists (create if first time)
5. Copy `screenshot.md` to repo
6. Update README table with command description
7. Commit and push
8. Report success with repo URL

## Notes

- Commands are single `.md` files in `~/.claude/commands/`
- Target repository: `HartreeWorks/claude-commands`
- Local clone kept at `~/.claude/skills/share-command/claude-commands/`
- **Always complete security review before publishing**
- Command names derive from the filename (without `.md` extension)

## Update check

This skill is managed by [skills.sh](https://skills.sh). To check for updates, run `npx skills update`.
