---
title: Windows.Forensics.UEFI
hidden: true
tags: [Client Artifact]
---

Searches for an EFI partition in the current partition table and
enumerates all files in it.


<pre><code class="language-yaml">
name: Windows.Forensics.UEFI
description: |
  Searches for an EFI partition in the current partition table and
  enumerates all files in it.

parameters:
  - name: ImagePath
    default: "\\\\?\\GLOBALROOT\\Device\\Harddisk0\\DR0"
    description: Raw Device for main disk containing partition table to parse.
  - name: SectorSize
    type: int
    default: 512

sources:
- query: |
    SELECT * FROM foreach(row={
       SELECT *, Size AS PartitionSize
       FROM Artifact.Windows.Forensics.PartitionTable(
          ImagePath=ImagePath, SectorSize=SectorSize)
      WHERE name =~ "EFI"

    }, query={
       SELECT StartOffset, PartitionSize, name,
              OSPath.Path AS OSPath, Size, Mtime, Atime, Ctime, Data
       FROM glob(globs="/*",
          accessor="fat",
          root=pathspec(
            DelegateAccessor="offset",
            DelegatePath=pathspec(
               DelegateAccessor="raw_file",
               DelegatePath=ImagePath,
               Path=format(format="%d", args=StartOffset))))
    })

</code></pre>

