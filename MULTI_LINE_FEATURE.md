# Taskbook Multi-Line Input Feature Implementation

## Project Overview
Add a `--multi` flag to taskbook that allows creating multiple tasks through an interactive prompt, where each line becomes a separate task.

## Current vs Proposed Behavior

### Current (Single Task)
```bash
$ tb -t "Buy groceries"
# Creates 1 task
```

### Proposed (Multi-Line Input)
```bash
$ tb -t --multi
# Prompts user for input:
Enter tasks (one per line, empty line to finish):
> Buy groceries
> Call dentist
> Write blog post
> 
✓ Created 3 tasks
```

## Feature Requirements

### User Experience
1. User runs `tb -t --multi` or `tb --task --multi`
2. System prompts: "Enter tasks (one per line, press Enter twice to finish):"
3. User enters tasks, one per line
4. User presses Enter on empty line to finish
5. System creates all tasks and shows confirmation

### Technical Requirements
- **Backward Compatible**: Existing `tb -t "task"` syntax must still work
- **Interactive Input**: Use stdin to read multiple lines
- **Empty Line Detection**: Two consecutive newlines or single empty line signals end of input
- **Trim Whitespace**: Remove leading/trailing spaces from each task
- **Skip Empty Lines**: Don't create tasks for blank lines
- **Board/Priority Support**: Each task line can include `@board` and `p:X` syntax
- **Exit Gracefully**: Handle Ctrl+C gracefully

## Implementation Plan

### Files to Modify

#### 1. `cli.js` - Command Line Interface
**Location**: `/cli.js` (root of project)

**Changes Needed**:
- Add `--multi` flag to the options parser
- When `--multi` is detected with `-t` or `--task`, trigger interactive mode
- Pass control to the multi-task creation function

**Pseudocode**:
```javascript
// In cli.js, around where flags are parsed

if (flags.task && flags.multi) {
  // Call interactive multi-task creation
  taskbook.createTasksInteractive();
} else if (flags.task) {
  // Existing single task creation
  taskbook.createTask(input);
}
```

#### 2. `src/taskbook.js` - Core Logic
**Location**: `/src/taskbook.js` (likely location)

**New Function Needed**: `createTasksInteractive()`

**Pseudocode**:
```javascript
createTasksInteractive() {
  // 1. Display prompt
  console.log('Enter tasks (one per line, press Enter twice to finish):');
  
  // 2. Set up readline interface
  const readline = require('readline');
  const rl = readline.createInterface({
    input: process.stdin,
    output: process.stdout,
    prompt: '> '
  });
  
  const tasks = [];
  
  // 3. Listen for each line
  rl.on('line', (line) => {
    const trimmed = line.trim();
    
    if (trimmed === '') {
      // Empty line - finish input
      rl.close();
    } else {
      // Add task to list
      tasks.push(trimmed);
      rl.prompt(); // Show prompt for next task
    }
  });
  
  // 4. When input is complete, create all tasks
  rl.on('close', () => {
    if (tasks.length === 0) {
      console.log('No tasks entered.');
      return;
    }
    
    // Create each task using existing createTask method
    tasks.forEach(description => {
      this.createTask(description);
    });
    
    console.log(`✓ Created ${tasks.length} task${tasks.length > 1 ? 's' : ''}`);
  });
  
  // 5. Start prompting
  rl.prompt();
}
```

#### 3. Update Help Text
**Location**: Likely in `cli.js` or a separate help file

**Add to Usage**:
```
--multi            Use with -t to create multiple tasks interactively
```

**Add to Examples**:
```
$ tb -t --multi
$ tb --task --multi
```

### Dependencies Check
Check if `readline` is already available (it's built into Node.js, so should be fine).

## Testing Strategy

### Manual Tests

```bash
# Test 1: Basic multi-task creation
$ tb -t --multi
Enter tasks (one per line, press Enter twice to finish):
> Task 1
> Task 2
> Task 3
> 
✓ Created 3 tasks

# Verify tasks were created
$ tb

# Test 2: Tasks with boards
$ tb -t --multi
> @coding Fix bug #42
> @docs Update readme
> 
✓ Created 2 tasks

# Test 3: Tasks with priority
$ tb -t --multi
> Deploy to production p:3
> Review PR p:2
> 
✓ Created 2 tasks

# Test 4: Empty input (edge case)
$ tb -t --multi
> 
No tasks entered.

# Test 5: Tasks with whitespace (should trim)
$ tb -t --multi
>    Task with spaces   
> 
✓ Created 1 task

# Test 6: Mixed boards and priority
$ tb -t --multi
> @work @urgent Fix security issue p:3
> @personal Read documentation p:1
> 
✓ Created 2 tasks

# Test 7: Ctrl+C cancellation
$ tb -t --multi
> Task 1
> ^C
# Should exit gracefully without creating partial tasks

# Test 8: Backward compatibility (must still work)
$ tb -t "Single task without --multi flag"
✓ Created task 1
```

### Edge Cases to Handle

1. **Empty lines in middle of input**: Should skip them
   ```
   > Task 1
   > 
   > Task 2
   > 
   ```
   Should create 2 tasks, not fail

2. **Very long task descriptions**: Should handle gracefully

3. **Special characters**: Should preserve them
   ```
   > Task with "quotes" and @symbols
   ```

4. **Ctrl+C/Ctrl+D handling**: Should exit cleanly

## File Structure After Changes

```
taskbook/
├── cli.js                    # Modified: Add --multi flag parsing
├── src/
│   └── taskbook.js          # Modified: Add createTasksInteractive()
├── test/                    # Add new tests for multi-line feature
│   └── multi-task.test.js   # New test file
├── readme.md                # Modified: Document --multi flag
└── package.json             # No changes needed (readline is built-in)
```

## Implementation Steps

### Step 1: Setup Environment
```bash
# Ensure you're in the right directory
cd ~/dev/taskbook_xm

# Verify you're on your fork
git remote -v
# Should show YOUR-USERNAME, not klaudiosinani

# Create feature branch
git checkout -b feature/multi-line-input

# Install dependencies (if not already done)
npm install

# Test current version works
npm test
```

### Step 2: Understand Current Code
```bash
# Open and read these files to understand structure:
# - cli.js (how flags are parsed)
# - src/taskbook.js (how createTask currently works)
# - Look for existing interactive patterns

# Find where single task creation happens
grep -n "createTask" src/*.js
grep -n "flags.task" cli.js
```

### Step 3: Implement the Feature

1. **Modify `cli.js`**:
   - Add `multi` to the flags definition
   - Add conditional: if `flags.task && flags.multi` → call interactive method
   - Update help text

2. **Modify `src/taskbook.js`**:
   - Add `createTasksInteractive()` method
   - Use Node's built-in `readline` module
   - Collect tasks line by line
   - Call existing `createTask()` for each line

3. **Test locally**:
   ```bash
   # Link your local version for testing
   npm link
   
   # Now 'tb' command uses your local code
   tb -t --multi
   ```

### Step 4: Documentation
Update `readme.md`:

```markdown
### Create Multiple Tasks Interactively

To create multiple tasks through an interactive prompt, use the `--multi` flag:

\`\`\`bash
$ tb -t --multi
Enter tasks (one per line, press Enter twice to finish):
> Buy groceries
> Call dentist  
> Write blog post
> 
✓ Created 3 tasks
\`\`\`

Each task can include boards and priority:

\`\`\`bash
$ tb -t --multi
> @work Deploy app p:3
> @personal Plan vacation p:1
> 
✓ Created 2 tasks
\`\`\`
```

### Step 5: Commit and Push
```bash
# Stage changes
git add cli.js src/taskbook.js readme.md

# Commit with clear message
git commit -m "feat: Add --multi flag for interactive multi-task creation

- Adds --multi flag to create multiple tasks interactively
- Users enter tasks one per line, empty line to finish
- Each task supports @board and p:X syntax
- Backward compatible with existing single-task syntax
- Uses Node.js readline for interactive input"

# Push to your fork
git push origin feature/multi-line-input
```

## Code Example: Complete Implementation

### `cli.js` modification
```javascript
// Find the flags definition and add:
const flags = {
  // ... existing flags
  multi: {
    type: 'boolean',
    alias: 'm',
    default: false
  }
};

// Find where task creation happens and modify:
if (flags.task) {
  if (flags.multi) {
    // New interactive mode
    taskbook.createTasksInteractive();
  } else {
    // Existing single task mode
    const description = input.join(' ');
    taskbook.createTask(description);
  }
}
```

### `src/taskbook.js` new method
```javascript
createTasksInteractive() {
  const readline = require('readline');
  
  console.log('Enter tasks (one per line, press Enter twice to finish):');
  
  const rl = readline.createInterface({
    input: process.stdin,
    output: process.stdout,
    prompt: '> '
  });
  
  const tasks = [];
  let lastLineEmpty = false;
  
  rl.on('line', (line) => {
    const trimmed = line.trim();
    
    if (trimmed === '') {
      if (lastLineEmpty) {
        // Two empty lines in a row - finish
        rl.close();
      } else {
        lastLineEmpty = true;
        rl.prompt();
      }
    } else {
      lastLineEmpty = false;
      tasks.push(trimmed);
      rl.prompt();
    }
  });
  
  rl.on('close', () => {
    if (tasks.length === 0) {
      this._render.missingTasks();
      return;
    }
    
    // Create all tasks
    tasks.forEach(description => {
      this.createTask(description);
    });
    
    console.log(`\n✓ Created ${tasks.length} task${tasks.length === 1 ? '' : 's'}`);
  });
  
  rl.prompt();
}
```

## Verification Checklist

Before considering the feature complete:

- [ ] `tb -t --multi` prompts for input
- [ ] Empty line ends input collection
- [ ] Multiple tasks are created correctly
- [ ] Tasks appear when running `tb`
- [ ] Tasks can be checked off with `tb -c <id>`
- [ ] Board syntax works: `@boardname`
- [ ] Priority syntax works: `p:3`
- [ ] Combined syntax works: `@coding Fix bug p:3`
- [ ] Empty input shows appropriate message
- [ ] Ctrl+C exits gracefully
- [ ] Existing `tb -t "task"` still works (no regression)
- [ ] Help text updated
- [ ] README.md updated
- [ ] Tests pass: `npm test`

## Future Enhancements (Optional)

1. **File Input**: `tb -t --from tasks.txt`
2. **Confirmation Prompt**: "Create 5 tasks? (y/n)"
3. **Preview Mode**: Show what tasks will be created before confirming
4. **Undo Last**: Allow removing the last entered task before finishing

## Resources

- Node.js readline docs: https://nodejs.org/api/readline.html
- Taskbook contributing guide: https://github.com/klaudiosinani/taskbook/blob/master/contributing.md
- Your fork: https://github.com/YOUR-USERNAME/taskbook

## Notes for Claude Code

When implementing in Claude Code:

1. Start by exploring the existing codebase structure
2. Read `cli.js` to understand flag parsing
3. Read `src/taskbook.js` to understand task creation
4. Follow the existing code style and patterns
5. Test thoroughly before committing
6. Use `npm link` to test your local version

---

**Status**: Ready for implementation
**Priority**: Medium
**Complexity**: Low-Medium (mostly UI/UX work, core logic exists)
