## Logical Expression Tests

---

### 1. **Logical AND (`EXPR1 -a EXPR2`)**

```bash
test -f myfile.txt -a -r myfile.txt && echo "myfile.txt exists and is readable."
```

> ✅ Output appears only if `myfile.txt` exists **and** is readable.

---

### 2. **Logical OR (`EXPR1 -o EXPR2`)**

```bash
test -d mydir -o -f myfile.txt && echo "Either mydir is a directory or myfile.txt exists."
```

> ✅ Output appears if **either** `mydir` is a directory **or** `myfile.txt` exists.

---

### 3. **Logical NOT (`! EXPR`)**

```bash
test ! -f not_a_file.txt && echo "not_a_file.txt does not exist as a regular file."
```

> ✅ Output appears only if `not_a_file.txt` **does not** exist as a file.

---


