# Taskbook Multi-Line Task Creation with Editor

## Feature Overview
Add a `--multi` flag that opens the user's default editor (vim, nano, emacs, etc.) to create multiple tasks at once.

## User Experience

### Command
```bash
$ tb -t --multi
```

### Flow
1. User runs `tb -t --multi`
2. Taskbook creates a temporary file with a template
3. Opens the file in user's `$EDITOR` (vim, nano, VS Code, etc.)
4. User edits the file (add/remove/reorder tasks)
5. User saves and closes the editor (`:wq` in vim, `Ctrl+X` in nano)
6. Taskbook reads the file, parses tasks, creates them
7. Shows confirmation message

### Template File
```
# Taskbook Multi-Task Creation
# 
# Enter one task per line below
# Lines starting with # are comments (ignored)
# Empty lines are ignored
# You can use @board and p:X syntax on each line
#
# Examples:
#   Buy groceries
#   @coding Fix bug #42 p:3
#   @personal @health Schedule dentist appointment
#
# Save and close this file to create tasks
# -------------------------------------------------

```

### After Editing
```
# User adds:
Buy groceries
@coding Fix bug #42 p:3
@personal Call dentist
Schedule team meeting p:2

# Saves and closes
```

### Output
```
✓ Created 4 tasks:
  1. Buy groceries
  2. @coding Fix bug #42 p:3
  3. @personal Call dentist
  4. Schedule team meeting p:2
```

## Why This Approach is Better

### Advantages over readline-based input:
1. **Full editing power** - Delete, reorder, copy/paste tasks
2. **Visual overview** - See all tasks at once
3. **Familiar workflow** - Like `git commit`, `crontab -e`, `visudo`
4. **Works with ANY editor** - vim, nano, emacs, VS Code, Sublime
5. **Comments/instructions** - Template can guide users
6. **No learning curve** - Users already know their editor

### Comparison to Other Options

| Feature | System Editor | readline (Option 3) | Pipe Separator (Option 1) |
|---------|--------------|---------------------|---------------------------|
| Multi-line editing | ✅ Full power | ❌ Line by line | ❌ Single line |
| Reorder tasks | ✅ Easy | ❌ Can't | ❌ Can't |
| Delete/edit tasks | ✅ Before creation | ❌ Hard | ❌ Can't |
| Visual overview | ✅ See all tasks | ❌ One at a time | ✅ In command |
| Familiar to devs | ✅ Like git | ⚠️ Different | ⚠️ New syntax |
| Works offline | ✅ Yes | ✅ Yes | ✅ Yes |

## Implementation

### Dependencies
```javascript
// Built-in Node.js modules (no npm install needed)
const fs = require('fs');
const os = require('os');
const path = require('path');
const { execSync } = require('child_process');
```

### Core Function

```javascript
// In src/taskbook.js

createTasksWithEditor() {
  // 1. Create temporary file
  const tmpDir = os.tmpdir();
  const tmpFile = path.join(tmpDir, `taskbook-${Date.now()}.txt`);
  
  // 2. Write template to file
  const template = `# Taskbook Multi-Task Creation
# 
# Enter one task per line below
# Lines starting with # are comments (ignored)
# Empty lines are ignored
# You can use @board and p:X syntax on each line
#
# Examples:
#   Buy groceries
#   @coding Fix bug #42 p:3
#   @personal @health Schedule dentist appointment
#
# Save and close this file to create tasks
# -------------------------------------------------

`;
  
  fs.writeFileSync(tmpFile, template, 'utf8');
  
  // 3. Detect user's preferred editor
  const editor = process.env.VISUAL || 
                 process.env.EDITOR || 
                 (process.platform === 'win32' ? 'notepad' : 'vim');
  
  try {
    // 4. Open editor (blocks until user closes it)
    execSync(`${editor} ${tmpFile}`, { 
      stdio: 'inherit',
      shell: true 
    });
    
    // 5. Read the file after editing
    const content = fs.readFileSync(tmpFile, 'utf8');
    
    // 6. Parse tasks (ignore comments and empty lines)
    const tasks = content
      .split('\n')
      .map(line => line.trim())
      .filter(line => line && !line.startsWith('#'));
    
    // 7. Delete temporary file
    fs.unlinkSync(tmpFile);
    
    // 8. Create tasks
    if (tasks.length === 0) {
      this._render.missingTasks();
      return;
    }
    
    console.log(`\n✓ Created ${tasks.length} task${tasks.length === 1 ? '' : 's'}:`);
    tasks.forEach((description, index) => {
      this.createTask(description);
      console.log(`  ${index + 1}. ${description}`);
    });
    
  } catch (error) {
    // User cancelled or editor failed
    if (fs.existsSync(tmpFile)) {
      fs.unlinkSync(tmpFile);
    }
    
    if (error.signal === 'SIGINT') {
      console.log('\nTask creation cancelled.');
    } else {
      console.error('Error opening editor:', error.message);
    }
  }
}
```

### CLI Integration

```javascript
// In cli.js

// Add flag definition
const flags = {
  // ... existing flags
  multi: {
    type: 'boolean',
    default: false
  }
};

// In the task creation section
if (flags.task) {
  if (flags.multi) {
    // Use editor for multi-task creation
    taskbook.createTasksWithEditor();
  } else {
    // Single task creation (existing behavior)
    const description = input.join(' ');
    taskbook.createTask(description);
  }
}
```

## Features

### 1. Editor Detection
Respects user's editor preference in order:
1. `$VISUAL` environment variable
2. `$EDITOR` environment variable  
3. Platform default (`vim` on Unix, `notepad` on Windows)

### 2. Comment Support
Lines starting with `#` are ignored - great for:
- Instructions
- Organizing tasks into sections
- Temporarily disabling tasks

Example:
```
# Work tasks
@work Review PR #42 p:3
@work Update documentation

# Personal tasks  
@personal Buy groceries
# @personal Call dentist  ← commented out, won't be created
```

### 3. Empty Line Handling
Blank lines are automatically filtered out - users can organize for readability

### 4. Syntax Highlighting (Future)
Could add `.taskbook.txt` file type for syntax highlighting in editors

## Usage Examples

### Basic Usage
```bash
$ tb -t --multi
# Opens editor
# Add tasks
# Save and close
✓ Created 3 tasks
```

### With Custom Editor
```bash
# Use VS Code
EDITOR="code --wait" tb -t --multi

# Use nano
EDITOR=nano tb -t --multi

# Set permanently in ~/.bashrc or ~/.zshrc
export EDITOR=vim
```

### Advanced Organization
```
# === URGENT ===
@work @urgent Fix production bug p:3
@work @urgent Contact client about outage p:3

# === This Week ===
@coding Review teammate's PR p:2
@coding Update documentation p:1

# === Personal ===
@personal Buy birthday gift
@health Schedule dentist appointment

# === Ideas (commented out for now) ===
# @learning Read about Rust async
# @side-project Start blog post about X
```

## Testing Checklist

### Basic Functionality
- [ ] `tb -t --multi` opens editor
- [ ] After saving, tasks are created
- [ ] Cancelling editor (Ctrl+C) doesn't create tasks
- [ ] Empty file creates no tasks
- [ ] Comment lines are ignored

### Task Parsing
- [ ] Single task works
- [ ] Multiple tasks work
- [ ] Tasks with `@board` syntax work
- [ ] Tasks with `p:X` priority work
- [ ] Tasks with both board and priority work
- [ ] Empty lines between tasks are ignored
- [ ] Leading/trailing whitespace is trimmed

### Editor Compatibility
- [ ] Works with vim
- [ ] Works with nano
- [ ] Works with emacs
- [ ] Works with VS Code (`code --wait`)
- [ ] Works with Sublime (`subl --wait`)
- [ ] Respects `$EDITOR` variable
- [ ] Falls back to platform default

### Edge Cases
- [ ] Very long task descriptions
- [ ] Special characters in tasks
- [ ] Tasks with # in the middle (not at start)
- [ ] Unicode characters
- [ ] 100+ tasks at once
- [ ] File permissions issues handled gracefully

## Error Handling

### Scenarios to Handle
1. **Editor not found**: Show helpful message
   ```
   Error: Editor 'xyz' not found
   Please set EDITOR environment variable or use a default editor
   ```

2. **Permission denied**: Handle temp file creation issues
   ```
   Error: Cannot create temporary file
   ```

3. **Editor crashes**: Clean up temp file
   ```
   Editor exited with error. No tasks created.
   ```

4. **User cancels (Ctrl+C)**: Clean exit
   ```
   Task creation cancelled.
   ```

## Documentation Updates

### README.md

Add new section:

```markdown
### Create Multiple Tasks with Editor

To create multiple tasks using your preferred editor:

\`\`\`bash
$ tb -t --multi
\`\`\`

This opens a file in your default editor where you can:
- Add one task per line
- Use `@board` and `p:X` syntax on each line
- Add `#` comments to organize or disable tasks
- Reorder tasks by moving lines
- Delete tasks before creating them

Example file:
\`\`\`
# Work tasks
@work Fix production bug p:3
@work Update documentation

# Personal
@personal Buy groceries
@personal Call dentist
\`\`\`

Save and close the file to create all tasks at once.

#### Setting Your Editor

Taskbook respects the `EDITOR` environment variable:

\`\`\`bash
# Use VS Code
export EDITOR="code --wait"

# Use nano
export EDITOR=nano

# Use vim (default on Unix)
export EDITOR=vim
\`\`\`

Add to `~/.bashrc` or `~/.zshrc` to make permanent.
\`\`\`

### Help Text (cli.js)

Update:
```
--multi            Create multiple tasks using your editor
```

Examples:
```
$ tb -t --multi
$ tb --task --multi
```

## Advantages Over Original Options

### vs Option 1 (Pipe Separator)
✅ Can edit/reorder before creating
✅ Visual overview of all tasks
✅ Can use comments to organize
✅ Multi-line descriptions possible (future feature)

### vs Option 3 (readline/Interactive)
✅ Full editing power (vim/emacs users love this)
✅ Can see all tasks at once
✅ Can copy/paste from other sources
✅ Familiar workflow (like git commit)

## Future Enhancements

1. **Syntax highlighting** - Create `.taskbook` file type
2. **Multi-line tasks** - Use `---` separator for task bodies
3. **Batch operations** - Edit existing tasks in editor
4. **Templates** - Pre-fill with common task lists
5. **Import from file** - `tb -t --from tasks.txt`

## Implementation Checklist

- [ ] Add `createTasksWithEditor()` to `src/taskbook.js`
- [ ] Add `--multi` flag to `cli.js`
- [ ] Add editor detection logic
- [ ] Add template generation
- [ ] Add comment filtering
- [ ] Add error handling
- [ ] Update README.md
- [ ] Update help text
- [ ] Test with vim
- [ ] Test with nano
- [ ] Test with VS Code
- [ ] Test edge cases
- [ ] Write tests

## Estimated Complexity

**Low-Medium**
- Core logic: ~50 lines of code
- Uses only built-in Node.js modules
- Most complexity is error handling
- Leverages existing `createTask()` method

## Related Git Commands (Similar Pattern)

This pattern is well-established in CLI tools:
- `git commit` (opens editor for commit message)
- `git rebase -i` (interactive rebase in editor)
- `crontab -e` (edit cron jobs)
- `visudo` (edit sudoers file)
- `vipw` (edit password file)

Users already know this workflow! 🎉
