# Natas 24 → 25 — OverTheWire Write-up

## Objective

Bypass the password verification mechanism on the Natas 24 challenge and retrieve the credentials for the next level (natas25).

## 1) Source code analysis

Relevant snippet (simplified):

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

The application uses `strcmp()` to compare the user-supplied `passwd` value with the expected password, and negates the return value using `!` so the condition is intended to pass only when `strcmp()` returns `0` (strings are equal).

## 2) Identifying the vulnerability

The check assumes `$_REQUEST['passwd']` is a string, but the application does not validate its type. PHP supports sending HTTP parameters as arrays using bracket syntax (for example `?passwd[]=test`). When that happens, PHP populates `$_REQUEST['passwd']` as an array instead of a string.

Calling `strcmp()` with an array where a string is expected leads to different behavior depending on the PHP runtime:

- On some older PHP versions used by the challenge, `strcmp(array, string)` emits a warning and returns `NULL`.
- The `!` operator converts `NULL` to `true`, so `!strcmp($_REQUEST['passwd'], $expected)` evaluates to `true` and the authentication check is bypassed.

In PHP 8+, passing an array into functions expecting strings will often throw a `TypeError`, so exploitability depends on the PHP version used by the target.

## 3) Exploitation

Trigger the vulnerability by sending `passwd` as an array. Example requests:

- Normal (fails):

```
GET /?passwd=test HTTP/1.1
```

- Exploit (bypasses check on vulnerable runtimes):

```
GET /?passwd[]=test HTTP/1.1
```

Or in the browser address bar:

http://natas24.natas.labs.overthewire.org/?passwd[]=test

When the server receives `passwd` as an array, the vulnerable `strcmp()` call returns `NULL` (on the challenge PHP version) and `!NULL` evaluates to `true`, causing the application to reveal the credentials for natas25.

## 4) Root cause

- Missing input validation: the application accepts user input without verifying it is of the expected type.
- Unsafe use of `strcmp()` combined with logical negation: negating the result (`!strcmp(...)`) assumes the function always returns an integer comparison result, but other return values (like `NULL`) can alter control flow.
- Reliance on PHP semantics that vary across versions — the behavior that enables the exploit is not guaranteed across all PHP versions.

## 5) Remediation

Validate input types and use strict checks. Example safe approach:

```php
if (
    isset($_POST['passwd']) &&
    is_string($_POST['passwd']) &&
    strcmp($_POST['passwd'], $expectedPassword) === 0
) {
    // grant access
}
```

Better yet, use secure password handling primitives (hashing + `password_verify()`), fail closed on malformed input, and avoid relying on coercing function return values with `!`.

## 6) Key takeaways

- HTTP parameters can be supplied as arrays using PHP's bracket syntax (e.g. `name[]=value`).
- Always validate user-controlled input before passing it to functions that expect a specific type.
- Negating function results (e.g. `!strcmp(...)`) can mask errors and lead to logic bypasses when functions return unexpected values.
- PHP behavior can change between versions — when designing security-sensitive checks, ensure behavior is well-defined and robust across runtimes.
- Authentication checks should fail securely (deny access) on malformed input.

## Vulnerability classification

Improper Input Validation / Type Confusion

## Tools used

- Browser
- (Optional) Burp Suite for inspecting and modifying requests
- PHP source-code analysis

## Lab

OverTheWire — Natas 24 → 25
