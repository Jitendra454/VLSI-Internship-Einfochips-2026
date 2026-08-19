# Week 1: Linux, Shell Scripting & the ASIC Design Flow

> Part of my 8-week VLSI Summer Internship (eInfochips) documentation.
> Week 1 was structured as a 4-day module: ASIC flow overview → Linux file management → shell scripting & text processing → AWK/TCL. This guide follows that same structure so it matches how I actually learned it.

---

## Day 1: The ASIC Design Flow (Big Picture)

**Why start here:** Before touching a single Linux command, it helps to know *why* any of this matters. Every tool, script, and command in this internship exists to move a design through this pipeline. This is the map I kept coming back to.

### What is an ASIC?
An Application-Specific Integrated Circuit — custom silicon designed for one specific function (as opposed to a general-purpose CPU). Used in phones, IoT devices, automotive, data centers. Design is optimized around three competing goals, the **PPA** metrics:

- **Power** — minimize energy consumption
- **Performance** — meet timing/throughput targets
- **Area** — minimize chip size (= cost)

Improving one often costs you on another — this tradeoff shows up constantly through the flow.

### The Flow, Stage by Stage

**1. Specification & Architecture**
Define functional requirements, system-level block partitioning, and interfaces before writing any code.

**2. Front-End: RTL & Verification**
- Write the design's behavior in **Verilog/SystemVerilog** (RTL = Register Transfer Level)
- Verification is reportedly **~70% of total project time** — because catching a bug here is cheap; catching it after fabrication means an expensive silicon respin
- Flow: RTL Code → Testbench → Simulation → Coverage → Bug Detection
- **UVM** (Universal Verification Methodology) provides reusable verification components for this

**3. Logic Synthesis & DFT**
- Synthesis tools (e.g. Synopsys Design Compiler) convert behavioral RTL into a **gate-level netlist** — mapping to actual standard cells from a cell library
- Guided by **timing constraints (SDC files)**
- **DFT (Design for Testability)** gets inserted around here too — I go deeper on this in a later week

**4. Back-End: Physical Design**
- **Floorplanning & Power Planning** — define the core area, partition blocks, design the power grid
- **Placement** — position millions of standard cells on the die, minimizing wirelength/congestion
- **CTS & Routing** — Clock Tree Synthesis for synchronized clock delivery, then routing connects everything with metal layers

**5. Sign-Off & Tape-Out**
- **STA (Static Timing Analysis)** — mathematically verifies setup/hold timing at every flip-flop, no simulation needed
- **Physical Verification** — DRC (design rule check against foundry rules) and LVS (layout vs. schematic — confirms layout matches the netlist)
- **Tape-out** — final GDSII file sent to the foundry. No more changes after this point.

**Gotcha:** It's easy to think of this as a strict waterfall, but in practice there's a lot of iteration — e.g. STA violations send you back to placement/routing, not forward to fabrication.

---

## Day 2: Linux Fundamentals & File Management

### Why Linux matters here
Every EDA tool (Design Compiler, ICC2, simulators) runs on Linux and is scripted through the shell. This is the "plumbing" underneath the whole flow.

### Core navigation & file commands

| Command | Purpose | Example |
|---|---|---|
| `pwd` | Print current directory | `pwd` |
| `ls`, `ls -la` | List files (incl. hidden) | `ls -la` |
| `cd` | Change directory | `cd /home/user` |
| `mkdir` | Create directory | `mkdir project` |
| `touch` | Create an empty file / update timestamp | `touch notes.txt` |
| `cat` | Print file contents | `cat file.txt` |
| `rm` | Delete a file | `rm file.txt` |
| `mv` | Move/rename | `mv old.txt new.txt` |
| `cp` | Copy a file | `cp a.txt b.txt` |
| `clear` | Clear terminal | `clear` |
| `whoami` | Show current user | `whoami` |
| `chmod` | Change permissions | `chmod 755 script.sh` |
| `sudo` | Run as superuser | `sudo apt update` |
| `top` | Live process/resource monitor | `top` |
| `history` | Show command history | `history` |
| `man` | Manual page for a command | `man ls` |

**VLSI framing for `touch`:** used constantly for creating placeholder testbench files, generating multiple design-related files quickly, or (with `-t`) setting timestamps for version tracking.

### `cp` — copying files and directories

| Flag | What it does | Example |
|---|---|---|
| *(none)* | Copy a file to a new location | `cp file.txt backup/` |
| `-r` | Copy directories recursively | `cp -r src/ dst/` |
| `-i` | Ask before overwriting | `cp -i file.txt backup/` |
| `-v` | Show copy progress | `cp -v file.txt backup/` |
| `-r -i` | Recursive copy with overwrite confirmation | `cp -ri src/ dst/` |

### `find` — locating files across a directory tree

```bash
find [path] [options]

find . -name "*.v"                    # match by name/pattern
find . -type f                        # files only
find . -type d                        # directories only
find . -name "*.v" -and -type f       # combine conditions
find . -name "*.v" -or -name "*.sv"   # match either
find . -name "*.log" -and -type f     # VLSI example: find all log files in a project tree
```

**Why it matters for VLSI:** design directories get huge fast (RTL, testbenches, logs, reports scattered across sub-folders) — `find` is how you actually locate something in a project tree instead of navigating by hand.

---

## Day 3: Shell Scripting & Text Processing

### Shell scripting basics

A shell script starts with a **shebang** line, which tells Linux which interpreter to use:

```bash
#!/bin/bash
echo "Hello World"
echo "I am XYZ"
```

Key symbols/commands:

| Symbol/Command | Purpose |
|---|---|
| `#` | Comment |
| `echo` | Print data to terminal |
| `read` | Read data entered by user in terminal |
| `$var` | Call a variable in a script |
| `;` | End a statement so you can write another command on the same line |
| `$#` | Total number of arguments passed to the script |
| `echo $SHELL` | Check current working shell |
| `echo $0` | Current shell name |

**Making a script runnable:**
```bash
chmod +x script.sh
./script.sh
```

### Conditionals & loops (from practical scripts I wrote)

**Number comparison script** — reads two numbers and compares them:
```bash
#!/bin/bash
read -p "number a: " A
read -p "number b: " B

if [ $A -gt $B ]; then
    echo "a is greater than b"
elif [ $A -eq $B ]; then
    echo "a is equal to b"
else
    echo "b is greater than a"
fi
```

**Print-range script** — using `while`:
```bash
#!/bin/bash
num=10
count=5
while [ $count -le $num ]
do
    echo $count
    let count++
done
```

**Calculator script** — reads an operator symbol and branches:
```bash
if [ "$symbol" = "+" ]; then
    ans=$((a+b))
    echo "sum is : $ans"
elif [ "$symbol" = "-" ]; then
    ans=$((a-b))
    echo "sub is : $ans"
elif [ "$symbol" = "*" ]; then
    ans=$((a*b))
    echo "mul is : $ans"
else
    echo "error type correct symbol"
fi
```

**File organizer script** — a genuinely useful automation example: sorts a messy directory into subfolders by file type.
```bash
#!/bin/bash
mkdir logs/ tcl_scripts/ text_files/ scripts/ zip/

mv *.log logs/
mv *.tcl tcl_scripts/
mv *.txt text_files/
mv *.sh scripts/
mv *.zip zip/
```
This takes a folder full of mixed `.sh`, `.log`, `.tcl`, `.txt`, `.zip` files and organizes them into type-specific folders automatically — a small script, but a real demonstration of why automation matters once a project directory gets messy.

**Best practices I noted:**
- Always quote variables (`"$VAR"`) to avoid word-splitting issues, especially with filenames
- Use descriptive variable names (`DESIGN_DIR`, not `d`)
- Don't overwrite system variables like `$PATH` without knowing the implications
- `export` makes a variable available to child processes/sub-shells
- `local` prevents variable name conflicts inside functions

**Practical VLSI-flavored one-liners:**
```bash
for i in {1..10}; do mkdir test_$i; done              # create multiple test directories
if [ -f synth.log ]; then grep ERROR synth.log; fi     # check a synthesis log for errors
cp -ri design_v1 design_v1_backup                      # safely backup design files
```

### `grep` — pattern searching (this is the one I have the most detail on)

`grep` searches for a pattern inside a file or piped output.

```bash
grep "keyword" filename        # print all lines containing "keyword"
grep -c "keyword" filename     # count of matching lines
grep -h "keyword" filename     # suppress filename in output (useful across multiple files)
grep -l "keyword" *.txt        # list only the filenames that contain the keyword
grep -n "keyword" filename     # show matched lines WITH line numbers
grep -v "keyword" filename     # invert match — print lines that do NOT contain keyword
grep -w "keyword" filename     # match whole word only (not as a substring)
grep -o "keyword" filename     # print only the matched portion, not the whole line
grep -An "keyword" filename    # print matched line + n lines AFTER it
grep -Bn "keyword" filename    # print matched line + n lines BEFORE it
grep -Cn "keyword" filename    # print matched line + n lines before AND after
```

**Why it matters:** this is the command I'll use constantly for filtering tool logs — e.g. pulling every `ERROR` or `WARNING` line out of a huge synthesis log without opening the whole file.

### `sed` — stream editor (edit files through the terminal)

```bash
sed -n '2p' filename           # print only line 2
sed -n '$p' filename           # print the last line
sed -n '2,4p' filename         # print lines 2 through 4
sed -n '2p;4p' filename        # print line 2 AND line 4

sed 's/str1/str2/' filename    # replace str1 with str2 (prints to terminal, doesn't change file)
sed -i 's/str1/str2/' filename # -i makes the change IN the file itself
sed '5d' filename              # delete line 5
```

`sed` can also: delete a particular line, print a particular line, replace a word, add text before/after a line, and comment/uncomment a line.

**Gotcha that tripped me up initially:** `sed 's/.../.../'` without `-i` only shows you the result in the terminal — the file is untouched. You have to explicitly add `-i` if you actually want to modify the file. Easy to think it saved when it didn't.

### `cut` — extract columns/characters from structured text

```bash
cut -c1 filename          # first character of each line
cut -c5- filename          # 5th character to end of line
cut -c-40 filename         # first 40 characters

cut -d '|' -f3 filename        # field 3, where data is delimited by '|'
cut -d '|' -f1-4 filename      # range of fields 1 to 4
cut -d '|' -f1,3 filename      # only fields 1 and 3
```

---

## Day 4: AWK & Introduction to TCL

### `awk` — the most powerful of the text tools

`awk` scans a file line by line and splits each line into fields (space-separated by default).

```bash
awk '{print $1}' filename              # print field 1 (space-separated) of every line
awk -F'|' '{print $2}' filename        # print field 2, delimiter set to '|'
awk 'NR==2 {print $1}' filename        # print field 1 only on row 2 (NR = current row number)
awk 'NR!=2 {print $1}' filename        # print field 1 for every row EXCEPT row 2
awk 'NR>=3 && NR<=5' filename          # print rows 3 to 5
awk '{i=1; while (i<=3) {print $0; i++}}'   # print every line 3 times
awk 'if ($3 > 30) print $0' filename   # print every row where field 3 is greater than 30
```

More advanced patterns from the PPT:
```bash
awk '$2=="VIOLATION" {print $1, $3}' timing.rpt
awk '/ERROR/ {count[$2]++} END {for (type in count) print type, count[type]}'
awk 'NR>5 && NR<100 && $1 ~ /^[0-9]/ {print $0}' synth.log
```

**Why it matters:** `grep` finds lines; `awk` lets you pull out *specific fields* from structured report/log data — e.g. pulling just the violation type and value out of a timing report instead of the whole line.

### TCL — the "glue" of EDA tool automation

TCL (Tool Command Language) is the scripting language used to control EDA tools directly (Synopsys, Cadence, Mentor) — synthesis, place & route, and sign-off tools are all driven by TCL under the hood.

**Creating and running a TCL script:**
```bash
touch my_script.tcl
chmod +x my_script.tcl
tclsh my_script.tcl
```

**TCL vs Bash — key syntax differences:**

| Feature | Bash | TCL |
|---|---|---|
| Variables | `A=5` | `set A 5` |
| Printing | `echo "Hello"` | `puts "Hello"` |
| Variable expansion | `echo "Hello $name"` | `set name "VLSI"; puts "Hello $name"` |
| Math | `result=$((A+B))` | `set sum [expr $A + $B]` |

**The one rule that trips up every beginner:** in TCL, the opening curly brace `{` for control structures (`if`, `for`, `proc`, etc.) **must be on the same line** as the keyword. Unlike Bash, TCL is strict about this.

```tcl
# WRONG
if {$x > 5}
{
    puts "big"
}

# CORRECT
if {$x > 5} {
    puts "big"
}
```

**Why TCL is non-negotiable for VLSI:** every major EDA tool is scripted through it. Full design flows — loading a design, applying constraints, running synthesis/STA, saving outputs — get chained together in TCL scripts that can run unattended overnight on multi-billion-transistor designs.

---

## Gotchas / Things That Confused Me Initially

- `sed` without `-i` doesn't modify the file — only previews the change in the terminal
- TCL's brace-on-same-line rule, coming from Bash where it's more forgiving
- Remembering `chmod +x` before trying to run a script with `./script.sh`
- Distinguishing `grep -h` (suppress filename) from `grep -l` (show only filenames) — easy to mix these up since they sound like opposites of each other

---

## Hands-On Files

See the `scripts/` folder in this directory for the actual shell scripts referenced above (number comparison, print range, calculator, file organizer), plus the original presentation deck and handwritten notes for reference.
