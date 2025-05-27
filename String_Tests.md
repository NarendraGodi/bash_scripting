## String Tests

---

### 1. **Empty string (`-z STRING`)**

```bash
var=""
test -z "$var" && echo "The string is empty."
```

---

### 2. **Non-empty string (`-n STRING`)**

```bash
var="hello"
test -n "$var" && echo "The string is not empty."
```

---

### 3. **Strings are equal (`STRING1 = STRING2`)**

```bash
a="openai"
b="openai"
test "$a" = "$b" && echo "Strings are equal."
```

---

### 4. **Strings are not equal (`STRING1 != STRING2`)**

```bash
a="chatgpt"
b="gpt4"
test "$a" != "$b" && echo "Strings are not equal."
```

---


