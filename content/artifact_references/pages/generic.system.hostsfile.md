---
title: Generic.System.HostsFile
hidden: true
tags: [Client Artifact]
---

The system hosts file maps hostnames to IP addresses. In some cases,
entries in this file take precedence and overrides the results from
the system DNS service.

The file is a simple text file, with one line per IP address. Each
whitespace-separated word following the IP address is a hostname.
The Linux man page refers to the the first hostname as *canonical_hostname*,
and any following words as *aliases*. They are treated the same by this
artifact.

The hosts file is typically present on all Linux-based systems (including macOS),
with entries for localhost. The same file format is also supported on Windows.

The source *Hosts* returns each line in each hosts file that matches
the glob parameters for address and hostname. The hostname and aliases
are combined in a single column *Hostnames*. Columns returned:

- OSPath
- Hostnames
- Comment

Only comments that follows the hostname on the same line are captured in Comment.
Comments on their own lines are ignored.

A second source *HostsFlattened* provides a flattened result, with each row
containing an IP address and a single hostname.

This artifact also exports a function `parse_hostsfile()` that returns Hostname
and Aliases individually.


<pre><code class="language-yaml">
name: Generic.System.HostsFile
description: |
  The system hosts file maps hostnames to IP addresses. In some cases,
  entries in this file take precedence and overrides the results from
  the system DNS service.

  The file is a simple text file, with one line per IP address. Each
  whitespace-separated word following the IP address is a hostname.
  The Linux man page refers to the the first hostname as *canonical_hostname*,
  and any following words as *aliases*. They are treated the same by this
  artifact.

  The hosts file is typically present on all Linux-based systems (including macOS),
  with entries for localhost. The same file format is also supported on Windows.

  The source *Hosts* returns each line in each hosts file that matches
  the glob parameters for address and hostname. The hostname and aliases
  are combined in a single column *Hostnames*. Columns returned:

  - OSPath
  - Hostnames
  - Comment

  Only comments that follows the hostname on the same line are captured in Comment.
  Comments on their own lines are ignored.

  A second source *HostsFlattened* provides a flattened result, with each row
  containing an IP address and a single hostname.

  This artifact also exports a function `parse_hostsfile()` that returns Hostname
  and Aliases individually.

reference:
  - https://manpages.debian.org/bookworm/manpages/hosts.5.en.html

export: |
  LET _parse_hostsfile(OSPath) = SELECT parse_string_with_regex(
     string=Line,
     regex='''^[\t ]*(?P&lt;Address&gt;[^\s#]+)[\t ]+(?P&lt;Hostname&gt;[^\s#]+)(?P&lt;Aliases&gt;[^#\n\r]+)?(?:[\t ]*#(?P&lt;Comment&gt;.+))?''') AS Parsed
  FROM parse_lines(filename=OSPath)
  WHERE Parsed.Address

  LET parse_hostsfile(OSPath) = SELECT Parsed.Address AS Address,
     Parsed.Hostname AS Hostname,
     filter(list=split(sep='''\s+''', string=Parsed.Aliases), regex='.') AS Aliases,

     /* Remove any whitespace between comment character and comment: */
     regex_replace(re='''^\s+''', source=Parsed.Comment, replace='$1') AS Comment
  FROM _parse_hostsfile(OSPath=OSPath)

  LET Files = SELECT OSPath FROM glob(globs=hostsFileGlobs.HostsFileGlobs)

  LET HostsFiles = SELECT * FROM foreach(row=Files, query={
    SELECT OSPath, Address, Hostname, Aliases, Comment
    FROM parse_hostsfile(OSPath=OSPath)
  })

parameters:
  - name: hostsFileGlobs
    description: Globs to find hosts files
    type: csv
    default: |
        HostsFileGlobs
        C:\Windows\System32\drivers\etc\hosts
        /etc/hosts
  - name: HostnameRegex
    description: Hostname or aliases to match
    default: .
    type: regex
  - name: AddressRegex
    description: IP addresses to match
    default: .
    type: regex

sources:
  - name: Hosts
    query: |
      SELECT OSPath, Address,
        (Hostname, ) + Aliases AS Hostname,
        Comment
      FROM HostsFiles
      WHERE Hostname =~ HostnameRegex
        AND Address =~ AddressRegex

  - name: HostsFlattened
    query: |
      SELECT OSPath, Address, Hostname, Comment
      FROM flatten(query={
        SELECT OSPath, Address, (Hostname, ) + Aliases AS Hostname, Comment
        FROM HostsFiles
      })
      WHERE Address =~ AddressRegex
        AND Hostname =~ HostnameRegex

</code></pre>

