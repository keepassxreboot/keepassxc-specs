%%%

    title = "KeePass2 XML Payload Format"
    abbrev = "KeePass2 XML"
    category = "info"
    docname = "keepass2-xml-rfc"
    ipr= "trust200902"
    area = "Security"
    keyword = ["keepass", "keepassxc"]

    date = 2018-02-19T00:00:00Z
    
    [pi]
    symrefs = "yes"
    private = "yes"
    
    [[author]]
    initials="J."
    surname="Bevendorff"
    fullname="Janek Bevendorff"
    organization = "KeePassXC Team"
    
%%%

.# Abstract

This document describes the XML payload format used in KDBX 3.1 and 4.0
containers.  KDBX is an encrypted, XML-based container format for secure
password storage used by KeePass 2.x, KeePassXC and various other password
managers.  This document only describes the actual XML payload and not
the KDBX container format itself.

.# Status of This Memo

This document is not an Internet Standards Track specification; it is
published for informational purposes with the goal to support development
of KDBX-based password management applications.

.# Copyright Notice

Copyright (C) 2018 KeePassXC Team

{mainmatter}


# Introduction

This document describes the payload format used in KDBX (version 3.1 and
4.0) files.  The payload is encoded as XML ("Extensible Markup Language" [@!XML]).

KDBX file format versions 3.1 and 4.0 use the same XML structure with minor
differences in the tags supported.  A note is added where KDBX 3.1 and 4.0
differ.

The XML payload itself is mostly self-contained, but may reference data
from the outer KDBX container. A note is given where this is the case.

# Conventions

In this document, the keywords 'MUST', 'MUST NOT', 'REQUIRED',
'SHALL', 'SHALL NOT', 'SHOULD', 'SHOULD NOT', 'RECOMMENDED', 'MAY',
and 'OPTIONAL' in capital letters are used to indicate requirements
 for the format specification. Their meaning is described in [@!RFC2119].
 
 If not explicitly referring to *this* RFC document, the term 'Document'
 refers to the XML payload document of the KDBX container.
 
 Furthermore, the term 'Database' is used to refer to the full KDBX file
 and its contained data structures.

# General Structure

A KeePass XML document SHOULD start with an XML declaration.  The encoding of
 the XML document MUST be UTF-8 [@!RFC3629].
 
 The XML document MUST have a root element named `KeePassFile`.  The
 `KeePassFile` root element MUST have exactly two child nodes `Meta`
 and `Root`:

~~~
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<KeePassFile>
    <Meta>
        <!-- Database meta data -->
    </Meta>
    <Root>
        <!-- Database group and entry tree -->
    </Root>
</KeePassFile>
~~~

`Meta` is the parent element for general database meta data, while `Root`
contains the actual database structure.

The database structure under `Root` is a tree of *groups*.

A *group* is an element with various attributes defined in (#database-groups),
containing zero or more *entries*.  A group can also contain zero or
more sub groups, which can again contain groups and entries.  

An *entry* is a set of user credentials with key-value attributes defined in
 (#database-entries).

The database MUST have exactly one *root group* which is a direct ancestor
of the `Root` element containing all other groups and entries.

## Database Meta Data

The `Meta` element MAY contain any of the following elements to describe
various database meta data:

`Generator`
:   The name of the program that generated the KDBX file.

`DatabaseName`
:   An optional name for the KDBX database.

`DatabaseNameChanged`
:   ISO 8601 datetime [@!RFC3339] of the last change of `DatabaseName`.

`DatabaseDescription`
:   An optional description of the KDBX database.

`DatabaseDescriptionChanged`
:   ISO 8601 datetime [@!RFC3339] of the last change of `DatabaseDescription`.

`DefaultUserName`
:   The username to use as a default when creating a new entry in the database.

`DefaultUserNameChanged`
:   ISO 8601 datetime [@!RFC3339] of the last change of `DefaultUserName`.

`MaintenanceHistoryDays`
:   The age of the oldest history item to keep in days.

All of the above-described child elements of `Meta` MAY also be empty.

## Database Groups

## Database Entries

# Security Considerations

KDBX is an encrypted container format.  The security of the XML payload
described in this document depends on the strength of the cipher and
passphrase / key file used for encrypting the encapsulating KDBX container.

In addition, all security considerations for [@!XML] apply as well,
including the use of recursive XML entities, which can consume an
arbitrarily large amount of memory on the user's system during parsing.


<reference anchor="XML" target="https://www.w3.org/TR/2008/REC-xml-20081126/">
    <front>
        <title>Extensible Markup Language (XML) 1.0 (Fifth Edition)</title>
        <author initials="T." surname="Bray" fullname="Tim Bray">
            <organization>Textuality and Netscape</organization>
            <address>
                <email>tbray@textuality.com</email>
            </address>
        </author>
        <author initials="J." surname="Paoli" fullname="Jean Paoli">
            <organization>Microsoft</organization>
            <address>
                <email>jeanpa@microsoft.com</email>
            </address>
        </author>
        <author initials="C. M." surname="Sperberg-McQueen" fullname="C. M. Sperberg-McQueen">
            <organization>W3C</organization>
            <address>
                <email>cmsmcq@w3.org</email>
            </address>
        </author>
        <author initials="E." surname="Eve" fullname="Eve Maler">
            <organization>Sun Microsystems, Inc.</organization>
            <address>
                <email>eve.maler@east.sun.com</email>
            </address>
        </author>
        <author initials="F." surname="Yergeau" fullname="François Yergeau"/>
        <date day="26" month="November" year="2008"/>
    </front>
</reference>

{backmatter}
