# Natas 24 → 25 — OverTheWire Write-up

---

## Objective
Retrieve the credentials for the next level (natas25) by bypassing the password verification mechanism on the Natas 24 challenge.

---

## 1) Source code (relevant snippet)

```php
if (array_key_exists("passwd", $_REQUEST)) {
    if (!strcmp($_REQUEST["passwd"], "<censored>")) {
        echo "<br>The credentials for the next level are:<br>";
        echo "<pre>Username: natas25 Password: <censored></pre>";
    } else {
        echo "<br>Wrong!<br>";
    }
}
```

The application uses `strcmp()` to compare the user-supplied `passwd` value with the expected password, and negates the return value with `!`. The intent is to run the protected branch only when `strcmp()` returns `0` (strings are equal).

---

## 2) Why this is vulnerable
- The code assumes `$_REQUEST['passwd']` is a string but never validates its type.
- PHP allows parameters to be sent as arrays using bracket syntax (e.g. `?passwd[]=value`). When that happens, `$_REQUEST['passwd']` becomes an array.
- Passing an array to `strcmp()` (which expects strings) produces runtime-dependent behavior:
  - On older PHP runtimes used by the challenge, `strcmp(array, string)` emits a warning and returns `NULL`.
  - `!NULL` evaluates to `true`, so the authentication check is bypassed.
  - On modern PHP (PHP 8+), passing an array where a string is expected typically throws a `TypeError`, so exploitability depends on the server's PHP version.

This is a type-handling / input-validation bug combined with unsafe negation of a function return value.

---

## 3) Exploitation (reproduction)
Trigger the vulnerability by sending `passwd` as an array.

Examples:

Normal (expected, fails):

```
GET /?passwd=test HTTP/1.1
```

Exploit (bypasses the check on vulnerable runtimes):

```
GET /?passwd[]=test HTTP/1.1
```

Or paste in the browser address bar:

```
http://natas24.natas.labs.overthewire.org/?passwd[]=test
```

What happens: the server receives `passwd` as an array; `strcmp()` returns `NULL` on the vulnerable PHP version; `!NULL` is `true`, and the page prints the credentials for `natas25`.

---

## 4) Root cause
- Missing input validation: the code accepts user input without verifying it is the expected type (string).
- Unsafe idiom: negating the return of `strcmp()` (`!strcmp(...)`) masks non-integer return values and can turn warnings or errors into truthy values.
- Reliance on PHP semantics that vary across versions — behavior that allows the bypass is not guaranteed across all PHP releases.

---

## 5) Remediation
Always validate and sanitize input prior to use. Prefer explicit, strict checks instead of relying on coercion.

Example secure check:

```php
if (
    isset($_POST['passwd']) &&
    is_string($_POST['passwd']) &&
    strcmp($_POST['passwd'], $expectedPassword) === 0
) {
    // grant access
}
```

Better practices:
- Use `password_hash()` / `password_verify()` for password handling.
- Fail closed: deny access on malformed input rather than attempting to coerce or ignore it.
- Avoid negating function return values (e.g. `!strcmp(...)`); use explicit comparison (`=== 0`).

---

## 6) Key takeaways
- PHP HTTP parameters may be supplied as arrays using bracket syntax (e.g. `name[]=value`).
- Validate user-controlled input types before passing them to functions that expect specific types.
- Negating function results (like `!strcmp(...)`) can mask errors and lead to logic bypasses when function returns are unexpected.
- PHP behavior changes between versions; security checks should be robust across runtimes.
- Authentication checks should fail securely on malformed input.

---

## Vulnerability classification
Improper Input Validation / Type Confusion

---

## Tools used
- Browser
- (Optional) Burp Suite for request inspection / modification
- PHP source-code analysis

---

## Lab
OverTheWire — Natas 24 → 25
