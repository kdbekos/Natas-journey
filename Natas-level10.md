# 🔓 Challenge: Natas Level 10 → 11

## 📋 Objective

Find the password for **Natas 11**, located at `/etc/natas_webpass/natas11`.

---

## 🔍 Analysis

The application provides a search interface that queries a dictionary file using the `grep` command. Unlike the previous level (Natas 9), this level implements a character blacklist via a regular expression to block command chaining.

### Vulnerable Code

```php
if($key != "") {
    if(preg_match('/[;|&]/',$key)) {
        print "Input contains an illegal character!";
    } else {
        passthru("grep -i $key dictionary.txt");
    }
}
```

**Blocked Characters:** `;`, `|`, `&`

The blacklist prevents command chaining, but **doesn't account for argument injection**.

---

## 💡 Exploitation Strategy

The vulnerability lies in the fact that the input `$key` is passed directly into a shell command **without being escaped or quoted**. While we cannot chain a new command using shell metacharacters, we can still **manipulate the arguments** of the grep command itself.

### Key Insight

The structure of the executed command is:
```bash
grep -i [USER_INPUT] dictionary.txt
```

By including a **space** in our input, we can trick `grep` into treating our input as:
1. A search pattern
2. An additional file path to search

This is known as **Argument Injection** (different from Command Injection).

---

## ✅ Solution

### Payload

```
. /etc/natas_webpass/natas11
```

### Execution Flow

**Our Input:**
```
. /etc/natas_webpass/natas11
```

**Resulting Command:**
```bash
grep -i . /etc/natas_webpass/natas11 dictionary.txt
```

### What Happens

- `grep` searches for the pattern `.` (matches any single character)
- It searches in **two files**:
  1. `/etc/natas_webpass/natas11` ← Our target file!
  2. `dictionary.txt`
- The output displays all matching lines from both files
- The natas11 password is revealed

---

## 🛡️ Prevention & Remediation

### Why Blacklisting Fails

This challenge demonstrates that **blacklisting is not an effective security measure**. It is often possible to achieve the same malicious goal using permitted characters (in this case, the space character).

### Secure Alternatives

1. **Best Practice:** Avoid shell-executing functions entirely
   ```php
   // Instead of passthru(), use native PHP functions:
   $results = array_filter(
       file('dictionary.txt'),
       fn($line) => stripos($line, $key) !== false
   );
   ```

2. **If passthru() is necessary:** Use `escapeshellarg()`
   ```php
   passthru("grep -i " . escapeshellarg($key) . " dictionary.txt");
   ```

3. **Use Whitelisting** instead of blacklisting
   ```php
   if(!preg_match('/^[a-zA-Z0-9]+$/', $key)) {
       print "Input contains invalid characters!";
   }
   ```

---

## 🎯 Key Takeaway

- **Argument Injection** ≠ **Command Injection**
- Blacklists are easily bypassed; **whitelists are more secure**
- **Never trust user input** in shell commands
- Use language-native APIs whenever possible
