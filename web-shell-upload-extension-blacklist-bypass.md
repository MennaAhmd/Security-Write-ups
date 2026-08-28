# Web Shell Upload via Extension Blacklist Bypass | PortSwigger

**Lab:** Web Shell Upload via Extension Blacklist Bypass
**Category:** File Upload Vulnerabilities
**Difficulty:** Practitioner
**Platform:** PortSwigger Web Security Academy

Lab link: https://portswigger.net/web-security/file-upload/lab-file-upload-web-shell-upload-via-extension-blacklist-bypass

## Introduction

File upload features are everywhere avatars, documents, attachments and they're one of the fastest ways for an application to hand an attacker code execution if the server-side validation is weak.

This lab blacklists the obvious extension (`.php`), but a blacklist can only block what its author thought of. In this write-up I'll walk through how I found an extension that wasn't on that list one that Apache still happily executed as PHP and used it to read `/home/carlos/secret`.

There's more than one way to solve this lab. The "intended" PortSwigger solution abuses an `.htaccess` upload to remap a brand-new extension to the PHP handler. I went a different route: fuzzing the extension itself. Both are worth knowing, and I'll touch on why mine worked toward the end.

**Credentials provided:** `wiener:peter`

---

## Step 1: Mapping the Upload Feature

After logging in as `wiener`, the account page exposes an avatar upload field.

<img width="1502" height="472" alt="image" src="https://github.com/user-attachments/assets/f80cf81c-26e1-47cc-8f89-4bb9d2a7b1b8" />


Before trying to break anything, I wanted to understand how a normal upload behaves: where the file ends up, what the server checks, and how it's served back. The account page shows a file input under **Avatar**, tied to an **Upload** button that posts to `/my-account/avatar`.

<img width="1012" height="637" alt="image" src="https://github.com/user-attachments/assets/858aab8d-05aa-4386-b32b-22f42ad2e659" />


## Step 2: Confirming the Blacklist

The first, obvious test is uploading a plain PHP web shell and seeing what happens.

**`shell.php`:**
```php
<?php
echo file_get_contents('/home/carlos/secret');
?>
```

The server rejected it outright:

<img width="912" height="261" alt="image" src="https://github.com/user-attachments/assets/8d66238b-c646-4a2a-9ff3-0949643a9d26" />


So `.php` is explicitly blacklisted. That's not a surprise it's the first extension anyone would think to block. The interesting question is what *else* the server would execute as PHP that the blacklist author didn't think to list.

## Step 3: A Near Miss `.pHp5`

Blacklists that key off a fixed extension string are usually easy to sidestep with case changes or alternate PHP extensions (`.php3`, `.php4`, `.php5`, `.phtml`, `.pht`…). I tried mixing both approaches at once and renamed the shell to `shell.pHp5`.

<img width="1009" height="584" alt="image" src="https://github.com/user-attachments/assets/a17ac850-5ef6-49a3-9cdf-f1645ccdd95f" />


This time the upload was accepted no rejection message, no error. The blacklist clearly wasn't matching this extension.

But accepted isn't the same as executed. Opening the uploaded file directly:

<img width="926" height="279" alt="image" src="https://github.com/user-attachments/assets/40cf6b12-eaf5-470a-a01b-34efab7f1bea" />


The browser returned the **raw PHP source**, not its output. The `<?php ... ?>` code was displayed as plain text instead of running. This is a useful (and easy to miss) distinction: bypassing the filename filter only gets you a file on disk the server still has to be configured to hand that specific extension to the PHP interpreter, or all you've uploaded is a text file with a confusing name. In this case, `.pHp5` wasn't mapped to any handler Apache would execute, so it was served as static content.

## Step 4: `.phar` Accepted *and* Executed

Rather than trying to modify the server's configuration (the `.htaccess` route), I looked for an extension that PHP itself already treats as executable out of the box. `.phar` PHP's own archive format fit the bill, since many PHP/Apache setups map it to the PHP handler alongside `.php`.

I renamed the same payload to `shell.phar` and uploaded it:

<img width="872" height="580" alt="image" src="https://github.com/user-attachments/assets/8a280657-184d-42b0-a9ab-a2f00cf0b7fb" />



The server accepted it without complaint:

<img width="904" height="194" alt="image" src="https://github.com/user-attachments/assets/7ea744d3-37de-4449-82f9-a4dff46453c3" />


This time, opening the file actually executed it:

<img width="904" height="194" alt="image" src="https://github.com/user-attachments/assets/c232ae11-a066-438d-8e99-5701f57675e6" />


The response contained the contents of `/home/carlos/secret` instead of the source code confirmation that the PHP interpreter had run the file rather than just serving it.

## Step 5: Submitting the Solution

I copied the value returned by the shell and submitted it through the lab's solution dialog.

<img width="1470" height="461" alt="image" src="https://github.com/user-attachments/assets/e98f38cc-ee64-498a-9e13-9a5deffa6ab9" />


<img width="1575" height="935" alt="image" src="https://github.com/user-attachments/assets/e1133dfd-c570-4314-8503-24531d85620d" />


**Lab solved.**

---

## The Attack Chain

```
Normal image upload
        │
Discover /my-account/avatar and /files/avatars/
        │
Upload shell.php  →  blocked ("php files are not allowed")
        │
Upload shell.pHp5 →  accepted, but served as plain text (not executed)
        │
Upload shell.phar →  accepted AND executed as PHP
        │
GET /files/avatars/shell.phar
        │
Read /home/carlos/secret
```

## Why This Worked

The blacklist only accounted for the `.php` extension. It didn't account for:

- **Case variation** (`.pHp5` slipped past the filter, even though it didn't get executed in this case)
- **Alternate PHP-family extensions** that a properly configured server may still hand to the PHP interpreter `.phar` being a good example, since it's PHP's own packaging format and is frequently left mapped to the same handler as `.php`

The core lesson from the `.pHp5` attempt is important on its own: getting a file *past the filter* and getting a file *executed* are two separate problems. A blacklist bypass is only dangerous if the resulting extension is also wired up to run server-side code. Testing each candidate extension by actually requesting it not just checking whether the upload was "accepted" is what separates a real finding from a false lead.

It's also worth noting this isn't the only bypass. PortSwigger's own intended solution uploads a malicious `.htaccess` file (`AddType application/x-httpd-php .l33t`) to make Apache treat a brand-new, attacker-chosen extension as PHP. That route works regardless of what extensions the server happens to already support mine relied on the server already recognizing `.phar`. Both land on the same root cause: the application trusted a blacklist to anticipate every extension the underlying server could ever be persuaded to execute.

## Impact

An attacker able to upload and execute arbitrary server-side code can typically:

- Achieve Remote Code Execution (RCE)
- Read application source code and configuration files
- Access environment variables, credentials, and secrets
- Pivot further into the underlying host or connected systems

Here, the shell executed with the privileges of the web application user and was sufficient to read another user's private file.

## Remediation

Blacklisting dangerous extensions is fragile because it requires the defender to predict every extension the server might ever execute including ones introduced by case changes, legacy PHP handler mappings, or (as in the `.htaccess` variant) configuration files the application shouldn't accept at all. A more robust upload implementation should:

1. **Use an allowlist**, not a blacklist only permit known-good extensions/MIME types (e.g. `.jpg`, `.png`), and reject everything else by default.
2. **Validate actual file content**, not just the filename or `Content-Type` header (e.g. verify image files are genuinely images).
3. **Generate the stored filename server-side** rather than trusting user input, removing any control over the extension entirely.
4. **Store uploads outside the web root**, or in a location with script execution disabled, and serve them through a handler that never interprets them as code.
5. **Reject configuration files** such as `.htaccess`, `.htpasswd`, or `web.config` outright, since these can redefine how a directory is handled.

## Key Takeaways

- A blacklist is only as good as the list of things its author remembered to block.
- "Upload accepted" and "file executes" are different milestones always confirm execution, not just acceptance.
- Case variations and legacy/alternate extensions (`.php5`, `.phtml`, `.phar`, etc.) are cheap to test and can reveal gaps a blacklist author didn't anticipate.
- The same vulnerability class can have more than one valid exploitation path knowing the "textbook" solution doesn't mean it's the only one worth understanding.

## References

- PortSwigger — [File Upload Vulnerabilities](https://portswigger.net/web-security/file-upload)
- PortSwigger — [Lab: Web Shell Upload via Extension Blacklist Bypass](https://portswigger.net/web-security/file-upload/lab-file-upload-web-shell-upload-via-extension-blacklist-bypass)
- OWASP — [Unrestricted File Upload](https://owasp.org/www-community/vulnerabilities/Unrestricted_File_Upload)
