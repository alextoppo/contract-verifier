# Contract Verifier — Hoare Logic Verification with Java PathFinder

A desktop tool for verifying **Hoare-logic contracts** on Java data structures using **Java PathFinder (JPF)** and **Symbolic PathFinder (SPF)**. Write a precondition and postcondition for any method, and the engine symbolically executes every possible path to prove or disprove the contract — no test cases or manual inputs needed.

```
{ !isFull() }   enqueue(p1)   { size() == oldSize + 1 }
      ↑                                ↑
 precondition                     postcondition
 (must hold before the call)      (must hold after, on every symbolic path)
```

The tool ships with a **Swing desktop UI** and also supports a **4-step CLI** for users who prefer the terminal.

---

## ⚠️ Platform & Java Version — Read Before Starting

| Platform | Support |
|---|---|
| Linux | ✅ Fully supported — developed and tested here |
| macOS | ✅ Works — same steps as Linux |
| Windows (native CMD / PowerShell) | ❌ JPF does not build or run reliably on Windows |
| Windows via WSL2 | ✅ Works — follow the Linux instructions inside a WSL2 Ubuntu terminal |

**Java version: you must use exactly Java 8.**
JPF and SPF will fail with bytecode errors on Java 11, 17, 21, or any newer version — even if the error message doesn't make it obvious. The most common symptom is:
```
error: Unsupported class file major version 61
```
This means your JPF JARs were compiled with or are being run by a JDK newer than 8. Fix: make `java -version` and `javac -version` both print `1.8.x` before doing anything else.

---

## Repository Structure

```
contract-verifier/
├── src/
│   ├── SPFVerifierUI.java              ← desktop GUI
│   ├── GenericContractsTest.java       ← JPF entry point (symbolic engine)
│   ├── DispatcherGenerator.java        ← generates method dispatcher via reflection
│   ├── BoundedQueue.java               ← example data structure (FIFO queue)
│   ├── BoundedQueue_contracts.txt      ← Hoare contracts for BoundedQueue
│   ├── BoundedList.java                ← example data structure (LIFO stack)
│   ├── BoundedList_contracts.txt       ← Hoare contracts for BoundedList
│   └── generic_verify.jpf              ← JPF config (auto-overwritten on each run)
└── README.md
```

> `GenericContractsTest.java`, `DispatcherGenerator.java`, and `generic_verify.jpf` are required support files. They must stay in the **same directory** as whatever `.java` class you want to verify.

---

## Step 1 — Install Java 8

**Ubuntu / Debian / WSL2:**
```bash
sudo apt update
sudo apt install openjdk-8-jdk
```

**macOS:**
```bash
brew install --cask temurin8
```

If you have multiple JDKs installed, set Java 8 as the active one:
```bash
# Linux / WSL2
sudo update-alternatives --config java
sudo update-alternatives --config javac
# → pick the entry containing "java-8"

# macOS
export JAVA_HOME=$(/usr/libexec/java_home -v 1.8)
export PATH=$JAVA_HOME/bin:$PATH
```

Verify before continuing:
```bash
java -version    # must print: openjdk version "1.8.x_xxx"
javac -version   # must print: javac 1.8.x_xxx
```

---

## Step 2 — Install Apache Ant

Ant is needed to build JPF from source.

```bash
# Linux / WSL2
sudo apt install ant

# macOS
brew install ant
```

```bash
ant -version   # Apache Ant(TM) version 1.x.x
```

---

## Step 3 — Build jpf-core

```bash
cd ~
git clone https://github.com/javapathfinder/jpf-core.git
cd jpf-core
git checkout java-8        # IMPORTANT: must be the java-8 branch, not main
ant build
```

Confirm it worked:
```bash
ls ~/jpf-core/build/
# Must contain: jpf.jar   jpf-classes.jar   RunJPF.jar
```

---

## Step 4 — Build jpf-symbc (Symbolic PathFinder)

```bash
cd ~
git clone https://github.com/SymbolicPathFinder/jpf-symbc.git
cd jpf-symbc
ant build
```

Confirm it worked:
```bash
ls ~/jpf-symbc/build/
# Must contain: jpf-symbc.jar   jpf-symbc-classes.jar

ls ~/jpf-symbc/lib/
# Must contain Choco/Z3 solver JARs — choco-solver-*.jar etc.
```

---

## Step 5 — Clone This Repo

```bash
git clone https://github.com/sandipghosal/contract-verifier.git
cd contract-verifier
```

---

## Running the UI

### Compile and launch

```bash
cd src/
javac SPFVerifierUI.java
java SPFVerifierUI
```

The UI window opens. **Before running any verification**, click `[ settings ]` in the top-right corner and fill in your JPF paths.

---

### Configuring Settings (required before first run)

Click **`[ settings ]`** → fill in all fields → click **`[ save ]`**.

| Settings field | What to enter |
|---|---|
| `jpf-symbc-classes` | `/home/YOUR_USERNAME/jpf-symbc/build/jpf-symbc-classes.jar` |
| `jpf-symbc.jar` | `/home/YOUR_USERNAME/jpf-symbc/build/jpf-symbc.jar` |
| `jpf-core.jar` | `/home/YOUR_USERNAME/jpf-core/build/jpf.jar` |
| `jpf-symbc/lib/*` | `/home/YOUR_USERNAME/jpf-symbc/lib/*` |
| `java8 home` | Leave **blank** if `java` in your PATH is already Java 8. Otherwise enter your Java 8 JDK root, e.g. `/usr/lib/jvm/java-8-openjdk-amd64` |
| `extra jars` | Leave **blank** for BoundedQueue / BoundedList. Only needed for classes that depend on external libraries. |

Replace `YOUR_USERNAME` with your actual Linux/macOS username (`echo $HOME` in a terminal will show you the full path).

> **Important:** These settings are not saved between sessions — you need to re-enter them each time you launch the UI. The default values in the fields (`../jpf-symbc/...`) are relative paths that only work if your `jpf-core` and `jpf-symbc` folders happen to be in the parent directory of wherever you ran `java SPFVerifierUI` from. Using the full absolute paths as shown in the table above avoids this problem entirely.

---

### Running a verification

1. Click **`> select file`** under `*.java` — navigate to `src/` and pick `BoundedQueue.java`
2. Click **`> select file`** under `*.txt` — pick `BoundedQueue_contracts.txt` (the file chooser opens in the same folder automatically)
3. Click **`[ run verification ]``
4. Results appear per-contract with a colour-coded status:
   - 🟢 **`[PASS]`** — JPF proved the contract holds on all symbolic paths
   - 🔴 **`[FAIL]`** — JPF found a path that violates the postcondition
   - 🟡 **`[UNKN]`** — contract not found in JPF output; expand to check the detail
5. Click **`▸`** on any row to expand the violation trace or detail message
6. Click **`// raw jpf output [toggle]`** at the bottom to see the full JPF stdout

---

## Running via CLI (no UI)

All four steps must be run from inside the `src/` directory. Run them in order every time you want to verify a class.

**Set up path variables first** (replace `~` with your actual home path if needed):
```bash
export JPF_SYMBC=~/jpf-symbc
export JPF_CORE=~/jpf-core
export JPF_CP="${JPF_SYMBC}/build/jpf-symbc-classes.jar:${JPF_SYMBC}/build/jpf-symbc.jar:${JPF_CORE}/build/jpf.jar:${JPF_SYMBC}/lib/*"
```

```bash
cd src/
```

**Step 1 — Compile the target class and engine:**
```bash
javac -cp ".:${JPF_CP}" BoundedQueue.java DispatcherGenerator.java GenericContractsTest.java
```

**Step 2 — Generate the method dispatcher:**
```bash
java -cp "." DispatcherGenerator BoundedQueue
# This produces GeneratedDispatcher.java in the current directory
```

**Step 3 — Compile the generated dispatcher:**
```bash
javac -cp ".:${JPF_CP}" GeneratedDispatcher.java
```

**Step 4 — Write the JPF config and run:**
```bash
cat > generic_verify.jpf << EOF
target=GenericContractsTest
target.args=BoundedQueue,BoundedQueue_contracts.txt
classpath=$(pwd)

vm.insn_factory.class=gov.nasa.jpf.symbc.SymbolicInstructionFactory
listener=gov.nasa.jpf.symbc.SymbolicListener

symbolic.min_int=-10
symbolic.max_int=10
symbolic.debug=true
symbolic.lazy=true
search.multiple_errors=true
EOF

java -cp ".:${JPF_CP}" gov.nasa.jpf.tool.RunJPF generic_verify.jpf
```

To verify `BoundedList` instead, replace `BoundedQueue` with `BoundedList` in Steps 1 and 4.

---

## Writing Your Own Contracts

Create a `.txt` file next to your `.java` file in `src/`. Each non-blank, non-comment line is one contract:

```
{ precondition }  methodName(arg1, arg2)  { postcondition }
```

Lines starting with `#` or `//` are ignored.

**Available tokens:**

| Token | Meaning |
|---|---|
| `p1`, `p2` | Symbolic integer arguments — mapped to method parameters in order |
| `b1`, `b2` | Additional symbolic integer variables for multi-variable constraints |
| `oldSize` | The value of `size()` captured before the method executes |
| `result` | The return value of the method |
| `size()` | Calls `size()` on the object |
| `isEmpty()` / `isempty()` | Calls `isEmpty()` |
| `isFull()` / `isfull()` | Calls `isFull()` |
| `contains(p1)` | Calls `contains(p1)` |
| `true` | Precondition always holds (unconditional contract) |

**Operators:** `==` `!=` `>` `<` `>=` `<=` `&&` `||` `!` `+` `-`

**Example:**
```
# Enqueue on non-full queue increases size by 1
{ !isFull() }  enqueue(p1)  { size() == oldSize + 1 }

# Dequeue on non-empty queue decreases size by 1
{ !isEmpty() }  dequeue()  { size() == oldSize - 1 }

# Intentionally wrong — JPF will detect the VIOLATION
{ isEmpty() }  enqueue(p1)  { size() == 0 }
```

> **Method names must exactly match** the method names in your Java class. The dispatcher is generated via reflection and routes calls by exact string match — a single character difference means the method is never called and the result will be UNKNOWN.

---

## Understanding the Output

```
[*] Verifying: {!isFull()} enqueue(p1) {size() == oldSize + 1}
[+] VALIDATED: {!isFull()} enqueue(p1) {size() == oldSize + 1}
```
✅ **VALIDATED** — JPF explored every symbolic execution path and the postcondition held on all of them.

```
[!!!] VIOLATION DETECTED!
Contract: {isEmpty()} enqueue(p1) {size() == 0}
```
❌ **VIOLATION** — JPF found at least one path where the postcondition failed. The output above the violation marker shows the symbolic variable values that triggered the failure.

---

## How It Works

```
YourClass.java  +  contracts.txt
       │                  │
       ▼                  ▼
 DispatcherGenerator   GenericContractsTest
 reads your class       reads contracts,
 via reflection,        creates symbolic
 writes a method        variables (p1, p2…),
 router               picks a contract
       │               symbolically
       ▼                  │
 GeneratedDispatcher ◄────┘
 routes method calls
 by name at runtime
       │
       ▼
 Java PathFinder + SPF
 explores ALL paths
 with symbolic integers
       │
  ┌────┴─────┐
PASS       FAIL
```

1. **DispatcherGenerator** inspects your class with Java reflection and writes `GeneratedDispatcher.java` — a router that maps method name strings to actual method calls on your object.
2. **GenericContractsTest** is the JPF target. It reads the contracts file, makes the method arguments symbolic, then for each contract: checks the precondition → calls the method via the dispatcher → checks the postcondition.
3. **JPF + SPF** runs `GenericContractsTest` symbolically, exploring every branch with symbolic constraint solving (Choco/Z3) instead of concrete values.

---

## Troubleshooting

**`Unsupported class file major version 61` (or 55, 65, etc.)**
Your JPF JARs were compiled with a newer JDK. Set Java 8 as default, delete `~/jpf-core/build/` and `~/jpf-symbc/build/`, and rebuild both with `ant build`.

**`ClassNotFoundException: gov.nasa.jpf.tool.RunJPF`**
`JPF_CP` is wrong or `ant build` didn't finish. Run `ls ~/jpf-core/build/` and confirm `RunJPF.jar` exists.

**`package gov.nasa.jpf.symbc does not exist` during compile**
The symbc JARs aren't on the classpath. Check that `JPF_CP` includes both `jpf-symbc.jar` and `jpf-symbc-classes.jar`.

**UI settings fields revert to `../jpf-symbc/...` after relaunch**
Settings are in-memory only — re-enter them each session, or set the environment variables listed below before launching so the fields pre-fill automatically:
```bash
export JPF_SYMBC_CLASSES=~/jpf-symbc/build/jpf-symbc-classes.jar
export JPF_SYMBC_JAR=~/jpf-symbc/build/jpf-symbc.jar
export JPF_CORE_JAR=~/jpf-core/build/jpf.jar
export JPF_SYMBC_LIB=~/jpf-symbc/lib/*
```
Add these lines to your `~/.bashrc` (Linux) or `~/.zshrc` (macOS) to make them permanent.

**`DispatcherGenerator: Error: Could not find or load main class`**
Step 2 needs `DispatcherGenerator.class` from Step 1. Make sure Step 1 compiled without errors.

**`GeneratedDispatcher.java not produced`**
The dispatcher generator uses reflection on the compiled class. If Step 1 failed silently, Step 2 produces nothing. Check the raw log.

**JPF hangs or runs out of memory**
Reduce the symbolic integer range in `generic_verify.jpf`:
```
symbolic.min_int=-3
symbolic.max_int=3
```

**Windows: `ant` errors / JPF won't build**
Use [WSL2](https://learn.microsoft.com/en-us/windows/wsl/install): run `wsl --install` in Windows PowerShell (admin), open the Ubuntu terminal it creates, and follow all steps in this README from inside that terminal.