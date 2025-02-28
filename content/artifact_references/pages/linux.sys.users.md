---
title: Linux.Sys.Users
hidden: true
tags: [Client Artifact]
---

Get User specific information like homedir, group etc from /etc/passwd.

<pre><code class="language-yaml">
name: Linux.Sys.Users
description: Get User specific information like homedir, group etc from /etc/passwd.
parameters:
  - name: PasswordFile
    default: /etc/passwd
    description: The location of the password file.
sources:
  - precondition: |
      SELECT OS From info() where OS = 'linux'
    query: |
      SELECT User, Description, Uid, Gid, Homedir, Shell
      FROM split_records(
            filenames=PasswordFile,
            regex=":", record_regex="\n",
            columns=["User", "X", "Uid", "Gid", "Description", "Homedir", "Shell"])

</code></pre>

