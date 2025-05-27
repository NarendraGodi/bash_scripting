## File Tests

---

### 1. **File exists (`-e`)**

```bash
test -e myfile.txt && echo "myfile.txt exists."
```

---

### 2. **Regular file (`-f`)**

```bash
test -f myfile.txt && echo "myfile.txt is a regular file."
```

---

### 3. **Directory (`-d`)**

```bash
test -d mydir && echo "mydir is a directory."
```

---

### 4. **Readable file (`-r`)**

```bash
test -r myfile.txt && echo "myfile.txt is readable."
```

---

### 5. **Writable file (`-w`)**

```bash
test -w myfile.txt && echo "myfile.txt is writable."
```

---

### 6. **Executable file (`-x`)**

```bash
test -x myscript.sh && echo "myscript.sh is executable."
```

---

### 7. **Non-empty file (`-s`)**

```bash
test -s myfile.txt && echo "myfile.txt is not empty."
```

---

### 8. **Newer than (`-nt`)**

```bash
test myfile1.txt -nt myfile2.txt && echo "myfile1.txt is newer than myfile2.txt."
```

---

### 9. **Older than (`-ot`)**

```bash
test myfile1.txt -ot myfile2.txt && echo "myfile1.txt is older than myfile2.txt."
```

---


