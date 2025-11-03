## VI Commands Cheat Sheet
- `vi filename` - Open or create a file named "filename".
- `i` - Enter insert mode to start editing the file.
- `Esc` - Exit insert mode.
- `:w` - Save the file.
- `:q` - Quit the editor.
- `:wq` - Save and quit the editor.
- `:q!` - Quit without saving changes.
- `dd` - Delete the current line.
- `yy` - Copy (yank) the current line.
- `p` - Paste the copied or deleted line below the current line.
- `/text` - Search for "text" in the file.
- `n` - Move to the next occurrence of the searched text.
- `u` - Undo the last action.
- `Ctrl + r` - Redo the last undone action.

## Making a Script Executable
By default, scripts are **not executable**.

Command:
```bash
chmod +x script_name.sh
```
OR
```bash
chmod 755 script_name.sh
```
- 7 = read, write, execute (owner)
- 5 = read, execute (group)
- 5 = read, execute (others)

Values:
- 1 = execute
- 0 = no permission
- 2 = write
- 4 = read

## Variables in Bash

#### Defining a variable
```bash
VARIABLE_NAME="value"
```
⚠️ No spaces around `=`.

#### Accessing a variable
```bash
echo $VARIABLE_NAME
```

#### Example
```bash
NAME="John"
echo "Hello, $NAME!"
```

### Variable with datatype: local
- Used inside functions to limit scope
```bash
my_function() {
    local LOCAL_VAR="I am local"
    echo $LOCAL_VAR
}

my_function
# echo $LOCAL_VAR  # This will give an error
```

## Command Substitution `$(...)`
This allows you to **store the output of a command** inside a variable.

#### Example
```bash
CURRENT_DATE=$(date)
echo "Today's date is: $CURRENT_DATE"
```

#### In the Assignment
```bash
timestamp=$(date +%Y%m%d-%H%M%S)
```

#### Understanding `date +FORMAT`
The `date` command formats time.

| Format | Meaning     |
| ------ | ----------- |
| `%Y`   | Year (2026) |
| `%m`   | Month       |
| `%d`   | Day         |
| `%H`   | Hour        |
| `%M`   | Minute      |
| `%S`   | Second      |

#### Example
```bash
date +%Y%m%d-%H%M%S
```
Output: `20240615-123456` (example output)

This is how we create **unique directory names**.

## Functions in Bash

#### Why functions?
- Reuse logic
- Cleaner scripts
- Library-style code (like `utils.sh`)

#### Syntax
```bash
function_name() {
    commands
}
# OR
function function_name {
    commands
}
```

#### Example
```bash
greet() {
    echo "Hello, $1!"
}

# Calling the function
greet "Alice"
```

### Returning Values from Bash Functions (Output Capture)

**Important Concept**
Bash functions cannot return strings or objects like functions in Python or Java.

- `return` in Bash is only for exit status codes (numbers from 0–255)
- To “return” data, Bash functions **print to standard output (stdout)** using echo
- The caller captures that output using command substitution `$(...)`

### Printing vs Returning in Bash
#### Printing Output
```bash
greet() {
    echo "Hello, $1!"
}

greet "Alice"
```

**What happens:**
- `Hello, Alice!` is printed to the terminal
- No value is stored for later use

#### Capturing Function Output (Bash-Style Return)
```bash
greet() {
    echo "Hello, $1!"
}

message=$(greet "Alice")
echo "Captured message: $message"
```

Output:
```
Captured message: Hello, Alice!
```

**What happens:**
- `echo` prints to stdout
- `$(greet "Alice")` captures that output into `message`
- This is the correct way to return values in Bash

## Function Arguments (`$1`, `$2`, ...)

Functions receive arguments just like scripts.

```bash
my_function() {
    echo "First argument: $1"
    echo "Second argument: $2"
}

# Calling the function with arguments
my_function "arg1" "arg2"
```

In the Assignment:
```bash
create_timestamped_dir "my-new-app"
# Inside the function, $1 -> "my-new-app"
```

## Printing Output (echo)
```bash
echo "Hello, World!"
```
Used to:
- Print logs
- Show results
- Return values visually

In the Assignment:
> Print the full path of the created directory

## Sourcing Another Script (`source`)
This is **CRITICAL** for this problem.

#### What `source` does
It loads functions and variables from another file into the current shell.

```bash
source utils.sh
# OR
. utils.sh
```

#### Why not `./utils.sh`?
Because:
- That runs it in a new shell
- Functions disappear after execution ❌

## Reliable Sourcing with `$(dirname "$0")`
This line:
```bash
source "$(dirname "$0")/utils.sh"
```

#### What it means
- `"$0"` - The path of the current script.
- `dirname` - Extracts the directory part of that path.
- `$(...)` - Command substitution to get the result.
- Ensures sourcing works no matter where script is run from
