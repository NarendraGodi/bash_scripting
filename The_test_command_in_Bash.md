## Test Command in Bash
The `test` command in Bash is used to evaluate conditional expressions. It returns true (exit status 0) or false (exit status 1) depending on the evaluation of the expression. Here are examples for each category of test cases in Bash:

---

### 1. **File Tests**

Check properties of files and directories.

| Test              | Description            | Example                            |
| ----------------- | ---------------------- | ---------------------------------- |
| `-e FILE`         | File exists            | `test -e myfile.txt`               |
| `-f FILE`         | Regular file           | `test -f myfile.txt`               |
| `-d FILE`         | Directory              | `test -d mydir`                    |
| `-r FILE`         | Readable               | `test -r myfile.txt`               |
| `-w FILE`         | Writable               | `test -w myfile.txt`               |
| `-x FILE`         | Executable             | `test -x myscript.sh`              |
| `-s FILE`         | File size > 0          | `test -s myfile.txt`               |
| `FILE1 -nt FILE2` | FILE1 newer than FILE2 | `test myfile1.txt -nt myfile2.txt` |
| `FILE1 -ot FILE2` | FILE1 older than FILE2 | `test myfile1.txt -ot myfile2.txt` |

---

### 2. **String Tests**

Evaluate string-related conditions.

| Test                 | Description           | Example             |
| -------------------- | --------------------- | ------------------- |
| `-z STRING`          | Empty string          | `test -z "$var"`    |
| `-n STRING`          | Non-empty string      | `test -n "$var"`    |
| `STRING1 = STRING2`  | Strings are equal     | `test "$a" = "$b"`  |
| `STRING1 != STRING2` | Strings are not equal | `test "$a" != "$b"` |

---

### 3. **Integer Tests**

Compare numeric values.

| Test            | Description           | Example        |
| --------------- | --------------------- | -------------- |
| `INT1 -eq INT2` | Equal                 | `test 5 -eq 5` |
| `INT1 -ne INT2` | Not equal             | `test 5 -ne 3` |
| `INT1 -gt INT2` | Greater than          | `test 5 -gt 3` |
| `INT1 -lt INT2` | Less than             | `test 3 -lt 5` |
| `INT1 -ge INT2` | Greater than or equal | `test 5 -ge 5` |
| `INT1 -le INT2` | Less than or equal    | `test 3 -le 5` |

---

### 4. **Logical Tests**

Combine expressions.

| Test             | Description | Example                               |
| ---------------- | ----------- | ------------------------------------- |
| `EXPR1 -a EXPR2` | Logical AND | `test -f myfile.txt -a -r myfile.txt` |
| `EXPR1 -o EXPR2` | Logical OR  | `test -d mydir -o -f myfile.txt`      |
| `! EXPR`         | Logical NOT | `test ! -f not_a_file.txt`            |

---

### Notes:

* You can also use `[ ... ]` as a synonym for `test` (e.g., `[ -f file ]` is equivalent to `test -f file`).
* For complex expressions, use `[[ ... ]]` for more flexible conditional checks (e.g., pattern matching, `==` operator with globbing).


