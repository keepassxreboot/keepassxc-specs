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

This document is not normative for the KDBX file format; it is
published for informative purposes with the goal to support development
of KDBX-based password management applications.

The information contained in this document is a work-in-progress reverse
engineering effort of the KDBX file format and likely to be incomplete or
even incorrect at this point in time. This memo is not recommended for
use as reference guide yet.

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
 
 The term 'Database' refers to the decrypted XML document payload of a
 KDBX container.
 
 Conversely, the term 'KDBX' refers to the full container, including
 its encrypted XML payload.

## General Database Structure

A database comprises two main parts: a *meta data section* and the actual
*data section*.

The meta data section contains general information about the database
itself, maintenance information (e.g., when and how to clean up deleted
entries), access and modification information (e.g., date and time of
the last database title change), and additional global resources (e.g.,
custom icons).

The data section contains the actual user data as a tree structure of
*groups* and *entries*.

A *group* is an element with various attributes containing zero or
more *entries*.  A group can also contain zero or more sub groups,
which can again contain groups and entries.  The format of a group
is described in (#groups).

An *entry* is a set of user credentials and other related data. An entry
MUST have a single parent group.  The attributes of an entry are
described in (#entries).

A database MUST have exactly one *root group*, which acts as common
ancestor of all other groups and entries.

# XML Format Description

## XML Schema

A full XML 1.0 schema [@!XMLSchema] is provided in:
[kdbx4.0-schema.xsd](kdbx4.0-schema.xsd)

The schema file is only intended for informative purposes.  In practice,
parsers SHALL NOT use this schema for strict validation.  Most importantly,
parsers SHALL NOT make assumptions about the order of elements.  `<xs:sequence>`
elements are used exclusively for Schema 1.0 compatibility, while in
reality, no particular element order is guaranteed.

Given the sensitive nature of the data contained in a passwords file,
a parser SHOULD be reasonably forgiving while trying to read the XML
data of a successfully decrypted KDBX file.

### Basic Data Types

All data stored in a KeePass XML file is UTF-8 string data.  Most XML
elements or attributes can contain arbitrary string values, but some elements
or attributes require special data types, which impose extra restrictions
on the allowed string values.

The following special data types exist:

- An *INTEGER* is a numeric decimal string consisting of one or more
characters in the range 0 to 9.  *INTEGERs* can have a negative sign `-`
in front.

- A *BOOLEAN* is represented by the strings `True` and `False`.  The
alternative forms `true` and `false` as well as `1` and `0` are also possible.

- A *DATETIME* is an XML datetime in ISO 8601 format [@!RFC3339].

- A *COLOR* is a six-digit hexadecimal RGB color code (characters 0-9 and A-F or
a-f), preceded by a \# character (e.g., \#FF00FF for yellow)

- A *BLOB* is a Base64-encoded [@!RFC4648] string representation of
arbitrary binary data.

- A *STRING MAP* is a complex XML type with exactly two child elements
`<Key>` and `<Value>` describing a mapping between a key string and a
value string.

### Identifiers

Unique resources inside the database are identified by a *UUID*, which is a globally
unique 128-bit (16-byte) identifier. *UUIDs* are encoded as *BLOBs*.

## Main Elements

A KeePass XML database SHOULD start with an XML declaration.  The encoding of
 the XML data MUST be UTF-8 [@!RFC3629].
 
 The XML document MUST have a root element named `<KeePassFile>`.  The
 `<KeePassFile>` root element MUST have exactly two child nodes `<Meta>`
 and `<Root>`:

~~~
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<KeePassFile>
    <Meta>
        <!-- Database meta data -->
    </Meta>
    <Root>
        <!-- Database root group -->
        <!-- Optional deleted object records -->
    </Root>
</KeePassFile>
~~~

`<Meta>` contains the database's meta data section, while `<Root>`
contains the actual data section, i.e., the root group (cf.
(#general-database-structure)).

Besides the database's root group, `<Root>` MAY also contain historical
records of deleted objects, whose format is described in (#deleted-objects).

## Meta Data Section

The `<Meta>` element MAY contain any of the following elements at most once
to describe various database meta data.

`<HeaderHash>`
:   The SHA-256 [@RFC4634] hash of the KDBX header data as *BLOB*.  This element is
only useful for KDBX 3.1 or lower, but MAY also be present in KDBX 4.0 or higher.

`<SettingsChanged>`
:   *DATETIME* of the last change of database meta data or KDBX container
settings.  This element is only specified for KDBX containers **version 4.0
or higher** and MUST NOT be present in KDBX 3.1 or lower.

`<Generator>`
:   The name of the program that generated the KDBX file.

`<DatabaseName>`
:   An optional name for the database.

`<DatabaseNameChanged>`
:   *DATETIME* of the last change of `<DatabaseName>`.

`<DatabaseDescription>`
:   An optional description of the database.

`<DatabaseDescriptionChanged>`
:   *DATETIME* of the last change of `<DatabaseDescription>`.

`<DefaultUserName>`
:   The username to use as a default when creating a new entry in the
database.

`<DefaultUserNameChanged>`
:   *DATETIME* of the last change of `<DefaultUserName>`.

`<MaintenanceHistoryDays>`
:   *INTEGER* indicating the maximum age of the oldest history item
to keep in days.

`<Color>`
:   A *COLOR* that can be used for coloring the program icon or database tab
inside the password manager application.

`<MasterKeyChanged>`
:   *DATETIME* of the last change of the database's master key.

`<MasterKeyChangeRec>`
:   *INTEGER* indicating the number of days after which the password manager
SHOULD recommend changing the database's master key (-1 means 'never')

`<MasterKeyChangeForce>`
:   *INTEGER* indicating the number of days after which the password manager
SHOULD force changing the database's master key (-1 means 'never')

`<MasterKeyChangeForceOnce>`
:   *BOOLEAN* indicating that the password manager SHOULD force changing the
database's master key next time the database is opened.  This element SHOULD
be removed or set to 'False' once the master key has been changed.

`<MemoryProtection>`
:   A complex XML type for configuring which fields SHOULD be protected by
in-memory encryption at runtime.  The element MAY have the *BOOLEAN* child
elements `<ProtectTitle>`, `<ProtectUserName>`, `<ProtectPassword>`,
`<ProtectURL>`, and `<ProtectNotes>` for protection of the title, username,
password, URL and notes fields.

`<CustomIcons>`
:   A complex XML type containing a sequence of zero or more custom icons, which
can be used for visually customizing groups and entries (described in (#groups)
and (#entries)).  An icon is an `<Icon>` element, which has exactly two children
`<UUID>` (a *UUID*) and `<Data>`.  `<Data>` contains the icon image data as *BLOB*.

`<RecycleBinEnabled>`
:   *BOOLEAN* indicating whether the database has an enable recycle bin.

`<RecycleBinUUID>`
:   The *UUID* of the recycle bin group inside the database.

`<RecycleBinChanged>`
:   *DATETIME* of the last change to the recycle bin.

`<EntryTemplatesGroup>`
:   The *UUID* of the group that contains templates for the creation of
new entries.

`<EntryTemplatesGroupChanged>`
:   *DATETIME* of the last change to the templates group.

`<HistoryMaxItems>`
:   An *INTEGER* indicating the maximum number of items the history of
an element may have (history items are described in (#entry-history)).
When the number of items in an entry's history exceeds this number, the
password manager SHOULD delete the oldest history items until the number
 of history items no longer exceeds this value.

`<HistoryMaxSize>`
:   An *INTEGER* indicating the maximum history size in bytes. If an entry's
history exceeds this size, the password manager SHOULD delete the oldest
history items, until the total history size no longer exceeds this value.

`<LastSelectedGroup>`
:   *UUID* of the group that was last selected by the user.

`<LastTopVisibleGroup>`
:   *UUID* of the last 'top' item. This seems to always be the root group
of the database.

`<Binaries>`
:   A sequence of zero or more `<Binary>` elements containing files attached
to entries in the database.  Each `<Binary>` element is a *BLOB* containing
the file's contents. `<Binary>` elements MUST have an `ID` attribute, which
is a unique *INTEGER* identifying the *BLOB* data and a *BOOLEAN* attribute
`Compressed`, indicating whether the *BLOB* data is Gzip-compressed [@RFC1952].
A database SHOULD NOT contain two `<Binary>` elements with the same *BLOB* data.

`<CustomData>`
:   A sequence of zero or more *STRING MAP* `<Item>` elements. `<Item>`
elements can be used by plugins to store arbitrary string data at database level.

## Data Section

### Groups

### Entries

#### Entry History

### Deleted Objects

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

<reference anchor="XMLSchema" target="https://www.w3.org/TR/xmlschema-1/">
    <front>
        <title>XML Schema Part 1: Structures Second Edition</title>
        <author initials="H. S." surname="Thompson" fullname="Henry S. Thompson">
            <organization>University of Edinburgh</organization>
            <address>
                <email>ht@cogsci.ed.ac.uk</email>
            </address>
        </author>
        <author initials="D." surname="Beech" fullname="David Beech">
            <organization>Oracle Corporation</organization>
            <address>
                <email>David.Beech@oracle.com</email>
            </address>
        </author>
        <author initials="M." surname="Maloney" fullname="Murray Maloney">
            <organization>Commerce One</organization>
            <address>
                <email>murray@muzmo.com</email>
            </address>
        </author>
        <author initials="N." surname="Mendelsohn" fullname="Noah Mendelsohn">
            <organization>Lotus Development Corporation</organization>
            <address>
                <email>Noah_Mendelsohn@lotus.com</email>
            </address>
        </author>
        <date day="28" month="October" year="2004"/>
    </front>
</reference>

{backmatter}
