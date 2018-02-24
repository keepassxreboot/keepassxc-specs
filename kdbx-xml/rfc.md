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

'Element' refers to an individual XML element inside the database.

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

A database MUST have exactly one *root group*, which acts as a common
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
alternative forms `true` and `false`, as well as `1` and `0`, are also
possible.

- An *OPT BOOLEAN* is the same as a *BOOLEAN* with the additional
allowed value `Null` (or `null`) which stands for no definite value.

- A *DATETIME* is an XML datetime in ISO 8601 format [@!RFC3339].

- A *COLOR* is a six-digit hexadecimal RGB color code (characters 0-9
and A-F or a-f), preceded by a \# character (e.g., \#FF00FF for magenta)

- A *BLOB* is a Base64-encoded [@!RFC4648] string representation of
arbitrary binary data.

- A *STRING MAP* is a complex XML type with exactly two child elements
`<Key>` and `<Value>` describing a mapping between a key string and a
value string.  The `<Value>` element MAY have the *BOOLEAN* attribute
`ProtectInMemory` to indicate that the value should be protected at
runtime by in-memory encryption.

### Protected Elements

All non-complex XML elements (including elements of the non-complex types
described in (#basic-data-types)) MAY have a *BOOLEAN* `Protected`
attribute.  If this attribute is set to `True`, the element's string data
MUST be a *BLOB* of encrypted data which must be decrypted before use.
The data MUST be encrypted with the KDBX container's inner random
stream cipher.

### Identifiers

Most resources (e.g., groups and entries) inside a database are
identified by a *UUID*, which is a globally unique 128-bit (16-byte)
identifier encoded as a *BLOB*.

An element of type *UUID* either directly defines the UUID of a resource
(indicated by *KEY*) or it is used to represent a reference to another
resource identified by this UUID (indicated by *REF*).  If the element is a
reference, but the target is undefined or does not exist, the zero UUID
MUST be used to indicate an invalid reference.  The zero UUID is a 16-tuple
of `0x0` bytes in Base64 encoding (i.e., `AAAAAAAAAAAAAAAAAAAAAA==`).
Where allowed, the referencing element MAY also be skipped instead of using
the zero UUID.

If a *UUID* element is a reference and the UUID is not the zero UUID,
the referenced resource MUST exist.

The zero UUID itself MUST NOT be used as a *KEY*.

### Ephemeral Changes

Most data in a database is generated and updated explicitly by the user.
There exists, however, data that is implicitly generated or updated by
certain (trivial) user actions (e.g. by selecting an entry or copying a
password).  Data that is implicitly updated in this way is classified as
*ephemeral data* and XML elements containing ephemeral data are called
*ephemeral elements* (indicated by *EPH*).

Changes to *EPH* elements SHOULD NOT mark a database as modified.
Accordingly, the user SHOULD NOT be asked to save changes before closing a
database which has only ephemeral changes.  If an implementation saves
changes automatically without explicit user interaction, ephemeral
changes SHOULD be silently discarded if no other non-ephemeral
changes exist.

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
contains the actual data section, i.e., the root group (see
(#general-database-structure)).

Besides the database's root group, `<Root>` MAY also contain historical
records of deleted objects, whose format is described in (#deleted-objects).

## Meta Data Section

The `<Meta>` element can contain any of the following elements at most once
to describe various database meta data.

`<HeaderHash>` (KDBX 3.1)
:   The SHA-256 [@RFC4634] hash of the KDBX header data as *BLOB*.  This
    element is only useful for KDBX 3.1 or lower, but MAY also be present
    in KDBX 4.0 or higher.

`<SettingsChanged>` (KDBX 4.0)
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
:   The username to use as a default when creating a new entry in the database.

`<DefaultUserNameChanged>`
:   *DATETIME* of the last change of `<DefaultUserName>`.

`<MaintenanceHistoryDays>`
:   *INTEGER* indicating the maximum age of the oldest history item
    to keep in days when performing database maintenance.

`<Color>`
:   A *COLOR* that a graphical implementation MAY use to show a colored
    application icon or database tab.

`<MasterKeyChanged>`
:   *DATETIME* of the last change of the database's master key.

`<MasterKeyChangeRec>`
:   *INTEGER* indicating the number of days after which an implementation
    SHOULD recommend changing the database's master key (-1 means 'never')

`<MasterKeyChangeForce>`
:   *INTEGER* indicating the number of days after which an implementation
    SHOULD force changing the database's master key (-1 means 'never')

`<MasterKeyChangeForceOnce>`
:   *BOOLEAN* indicating that an implementation SHOULD force changing the
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
    and (#entries)).  An icon is an `<Icon>` element, which has exactly two
    children `<UUID>` (a *UUID*) and `<Data>`.  `<Data>` contains the icon image
    data as *BLOB*.

`<RecycleBinEnabled>`
:   *BOOLEAN* indicating whether the database has an enable recycle bin.

`<RecycleBinUUID>` (*REF*)
:   The *UUID* of the recycle bin group inside the database.

`<RecycleBinChanged>`
:   *DATETIME* of the last change to the recycle bin.

`<EntryTemplatesGroup>` (*REF*)
:   The *UUID* of the group that contains templates for the creation of
    new entries.

`<EntryTemplatesGroupChanged>`
:   *DATETIME* of the last change to the templates group.

`<HistoryMaxItems>`
:   An *INTEGER* indicating the maximum number of items the history of
    an element may have (history items are described in (#entry-history)).
    When the number of items in an entry's history exceeds this number, an
    implementation SHOULD delete the oldest history items until the number
    of history items no longer exceeds this value.

`<HistoryMaxSize>`
:   An *INTEGER* indicating the maximum history size in bytes. If an entry's
    history exceeds this size, an implementation SHOULD delete the oldest
    history items, until the total history size no longer exceeds this value.

`<LastSelectedGroup>` (*REF*, *EPH*)
:   *UUID* of the group that was last selected by the user.

`<LastTopVisibleGroup>` (*REF*, *EPH*)
:   *UUID* of the top-most group that is still visible in the user's current
    scroll view.  Graphical implementations MAY use this element to save the
    scroll state of the groups tree.  If an implementation decides not to store
    scroll information, this SHOULD point to the root group.

`<Binaries>` (KDBX 3.1)
:   A sequence of zero or more `<Binary>` elements containing files attached
    to entries in the database.  Each `<Binary>` element is a *BLOB* containing
    the file's contents. `<Binary>` elements MUST have an `ID` attribute,
    which is a unique *INTEGER* identifying the *BLOB* data and a *BOOLEAN*
    attribute `Compressed`, indicating whether the *BLOB* data is
    Gzip-compressed [@RFC1952].
    \
    A database SHOULD NOT contain two `<Binary>` elements with the same
    *BLOB* data.
    \
    This element is only specified for KDBX 3.1 and lower.  KDBX 4.0 stores
    files in its encrypted binary inner header, so this element MUST NOT be
    present if the container format is version 4.0 or higher.

`<CustomData>`
:   A sequence of zero or more *STRING MAP* `<Item>` elements. `<Item>`
    elements can be used by plugins to store arbitrary string data at database
    level.

## Data Section

### Groups

Groups in a database are structured as a tree. There MUST be exactly one
root group inside the `<Root>` element.  This root group MAY contain an
arbitrary number of sub groups and entries.  A detailed description of
the database structure was given in (#general-database-structure).

A group is represented by a `<Group>` element, which MUST have a `<UUID>`
child element of type *UUID* defining the *KEY* of this group.

Any `<Group>` element inside another `<Group>` element is a sub group.
`<Entry>` elements inside a `<Group>` element are child entries of that group
(see (#entries)).

In addition, a group can have any of the following child elements at most once:

`<Name>`
:   A user-specified name for this group.

`<Notes>`
:   Additional notes which the user may save with this group.

`<IconID>`
:   The *INTEGER* identifier of a standard icon to display for this group.
    Standard icons are described in (#standard-icons)

`<CustomIconUUID>` (*REF*)
:   This element MAY be used to reference to a custom group icon from the
    `<CustomIcons>` element of the database's meta data section.
    If a non-zero UUID is specified, the referenced custom icon
    MUST be used instead of the default icon set in `<IconID>`.

`<Times>`
:   A complex XML type containing information about the group's last access,
    modification, or expiry date and time. A detailed description is given in
    (#times-and-expiry).

`<IsExpanded>` (*EPH*)
:   If a graphical implementation allows the user to expand and collapse sub trees,
    this element MAY contain a *BOOLEAN* indicating whether this group is
    expanded (`True`) or collapsed (`False`).  A collapsed group hides all
    it's child groups when displayed as a tree.

`<DefaultAutoTypeSequence>`
:   The default Auto-Type sequence to use for entries and sub groups in this
    group.  This overrides an implementation's default Auto-Type sequence.
    Sub groups or entries may again override this value.

`<EnableAutoType>`
:   *OPT BOOLEAN* for enabling or disabling Auto-Type for this group, its
    entries and sub groups.  `Null` inherits a definitive value from the
    parent group.  If this is the root group, `Null` is equivalent to `True`.
    Individual sub groups may override this value.  If a group enables
    Auto-Type, individual entries in this group may still choose to disable it.
    However, disabling Auto-Type at group level SHOULD disable it for all direct
    child entries, regardless of their setting.

`<EnableSearching>`
:   *OPT BOOLEAN* indicating whether entries in this group should be included
    in search results if the user searches the database.

`<LastTopVisibleEntry>` (*REF*, *EPH*)
:   Similarly to `<LastTopVisibleGroup>` in the database meta section, this
    saves a *UUID* reference to the top-most entry inside this group that
    is still visible inside the user's current scroll view.  GUI
    implementations MAY use this element to save the scroll state of
    individual groups.  If the group has no entries, this reference MUST point
    to the zero UUID.

`<CustomData>` (KDBX 4.0)
:   A sequence of zero or more *STRING MAP* `<Item>` elements. `<Item>`
    elements can be used by plugins to store arbitrary string data.  This is
    the same as the `<CustomData>` element from the database meta data section,
    but at group level.  This element MUST NOT be present if the container
    format is KDBX 3.1 or lower.

### Entries

An entry is where the user saves their actual secret data and credentials. 

An entry is represented by an `<Entry>` element, which MUST be a child
of a `<Group>` element.  Like `<Group>`, an `<Entry>` MUST have a `<UUID>`
child element of type *UUID* defining the *KEY* of this entry.

For storing the actual data, an entry can have an arbitrary number of
`<String>` elements of type *STRING MAP*.  Besides arbitrary user-defined
keys, the following keys SHOULD be present and displayed as dedicated
input fields and display columns:

`Title`
:   The entry title.

`UserName`
:   The username of this entry.

`Password`
:   The password of this entry.

`URL`
:   A URL to a remote location which this entry is used for.  An implementation
    SHOULD render this as a clickable link that launches a program associated
    with the URL's scheme.  The cmd:// scheme SHOULD execute arbitrary commands.

`Notes`
:   Additional notes the user may save with this entry.

In addition to `<String>` elements, an entry can also have an arbitrary number
of`<Binary>` elements.  This element is has a complex XML type defining a
reference to a file attachment.  The structure of this type is similar to
*STRING MAP* in that it MUST have an element `<Key>`, setting a key for this
attachment (usually the file name) and an element `<Value>`.

Though unlike *STRING MAP*, the `<Value>` element MUST *either* have an *INTEGER*
`Ref` attribute and no content *or* the raw attachment data as *BLOB* and no
attribute.  There are also no attributes for in-memory encryption.

For KDBX 3.1 or lower, the `Ref` attribute references a `<Binary>` from
the `<Binaries>` element inside the database's meta data section by its `ID`
attribute.

For KDBX 4.0 or higher, it references an item from the container's encrypted
binary inner header by its position.

If the container format is KDBX 4.0 or higher, an entry MAY also have a
`<CustomData>` element for saving arbitrary plugin data, identical to the one
that can be set at group level.  See (#groups) for a description of this element.
As for groups, the container format MUST be KDBX 4.0 or higher if this
element exists.

#### Additional Entry Data

Besides arbitrarily many `<String>` and `<Binary>` elements, the following
elements can occur at most once:

`<IconID>`
:   The *INTEGER* identifier of a standard icon to display for this group.
    Standard icons are described in (#standard-icons)

`<ForegroundColor>`
:   A *COLOR* to be used as custom entry text color.

`<BackgroundColor>`
:   A *COLOR* to be used as custom entry background color.

`<OverrideURL>`
:   An additional URL that overrides the triggered target when the user
    clicks the Link generated from the `URL` string field.

`<Tags>`
:   A list of used-defined tags for this entry separated by semicolons `;`.
    Tags allow to classify entries orthogonally to the group hierarchy.

`<Times>`
:   A complex XML type containing information about this entry's last access,
    modification, or expiry date and time. A detailed description is given in
    (#times-and-expiry).

`<AutoType>`
:   A complex XML type containing information about this entry's Auto-Type
    settings. A detailed description is given in (#autotype-settings).

`<History>`
:   A sequence of previous versions of this entry.  History items are
    described in (#entry-history).

#### Auto-Type Settings

An implementation SHOULD try to match window titles for Auto-Type automatically
with data from an entry and use `{USERNAME}{TAB}{PASSWORD}{ENTER}` as a default
Auto-Type sequence.  To lower the risk of accidental submission to wrong input
forms, it MAY also reduce the default sequence to `{USERNAME}{TAB}{PASSWORD}`.

An entry MAY specify additional Auto-Type settings, which override the default
Auto-Type settings provided by the implementation.  This is done in an
`<AutoType>` element inside the `<Entry>`.  The following child elements are
possible and can occur at most once:

`<Enabled>`
:   *BOOLEAN* indicating whether Auto-Type should be enabled for this entry.
    The default SHOULD be `True`.  If Auto-Type is enabled for the parent group,
    this can be used to override the setting.  On the other hand, if Auto-Type
    is disabled for the parent group, this setting SHOULD have no effect.

`<DataTransferObfuscation>`
:   *BOOLEAN* indicating whether an implementation MAY try to obfuscate Auto-Type
    key strokes to make it harder for key loggers to record the full sequence.

`<DefaultSequence>`
:   The default Auto-Type sequence to use for this entry (overrides the global
    or group default).

Additionally, the user SHOULD be able to add additional window associations to
an entry.  A window association is a specific Auto-Type setting for a specific
window.  An entry can hold an arbitrary number of `<Association>` elements,
one for every individual window association.

An `<Association>` MUST have the following two elements:

`<Window>`
:   The title of the window to match with this entry.  The string MUST support
    `*` as a wildcard character.

`<KeystrokeSequence>`
:   A custom Auto-Type sequence.  If the value is left empty, the sequence
    from `<DefaultSequence>` is used or, if that is empty as well, the group
    or global default.

#### Entry History

To improve robustness against user credential loss by accidental edits, all
changes to an entry SHOULD be recorded in an entry's history.

For this, an implementation SHOULD create a copy of an entry (excluding its
history) and add it to the original entry's `<History>` before applying any
modifications to the original entry.  This applies to all user-initiated,
non-ephemeral database changes which would result in an updated KDBX file
when saved.

A history item SHOULD be created immediately when the user commits a change
to the database, not only when the updated KDBX file is actually saved to
disk.

The newest history item MUST be last in the list.  Copying and adding an item
to the history MUST NOT change any of its elements. This also applies to all
ephemeral elements.  The history item MUST capture the exact state of the
original element before modification.

After adding a new history item, an implementation SHOULD ensure that the
history does not grow larger than the configured limits.

If the new number of history items exceeds the value configured in the
`<HistoryMaxItems>` setting from the databases meta data, the oldest history
items SHOULD be deleted until the number of actual history items does not
exceed this number anymore.

Similarly, if the total size of the history in bytes exceeds
`<HistoryMaxSize>`, the oldest items SHOULD be deleted, until the total size
does not exceed this number anymore.  The size of the history is calculated by
adding the sizes of all entries in the history.

The size of an entry SHOULD be calculated as the sum of its UTF-8 string data
in bytes, including all `<String>`, `<Binary>`, and `<CustomData>` keys and
values, Auto-Type `<Association>` window titles and sequences, `<Tags>`,
`<OverrideURL>`.  For simplicity, the size of the remaining entry data MAY be
estimated as `128`.

The size of `<Binary>` values SHOULD calculated *after* Base64 decoding.  

An implementation MAY also provide a manual or automatic database maintenance
feature, which deletes all history items older than `<MaintenanceHistoryDays>`.

### Times And Expiry

Groups and entries (hereafter called *items*) MAY have a `<Times>` element
recording and configuring various time- and usage-based data.  If present,
the `<Times>` element can have any the following child elements at most once:

`<CreationTime>`
:   *DATETIME* when the item was created.

`<LastModificationTime>`
:   *DATETIME* when the item was last modified.  If it was newly created,
    this SHOULD be the same as `<CreationTime>`.

`<LastAccessTime>` (*EPH*)
:   *DATETIME* when this item was last accessed.  Any kind of access MAY update
    this element, including data modification and item relocation, but also
    trivial tasks such as viewing an item's data, copying its data to the
    clipboard etc.  If an items was newly created, this SHOULD be the same as
    `<CreationTime>`.

`<ExpiryTime>`
:   User-configurable *DATETIME* when this item will expire.  An expired item
    SHOULD be marked in a special and distinguishable way (e.g., with a special
    icon or crossed out), so the user can easily spot expired items.  An
    implementation MAY also choose to hide expired items or display them in a
    special location.

`<Expires>`
:   *BOOLEAN* indicating whether this item will expire.  If this is set to `False`,
    `<ExpiryTime>` has no effect.  If this element is missing, a default value
    of `False` is assumed.

`<UsageCount>` (*EPH*)
:   *INTEGER* recording the number of times this item was used or accessed.
    Any kind of usage / access MAY update this element.  These kinds of usage
    SHOULD be the same that also update `<LastAccessTime>`.

`<LocationChanged>`
:   *DATETIME* when this item was last re-located inside the group hierarchy.
    This MAY be updated whenever the parent of an item changes.

### Recycle Bin

If the `<RecycleBinEnabled>` element of the database's meta data section is set
to `True`, an implementation MAY provide a recycle bin.  This is a regular group
into which groups and entries are moved before they are permanently deleted
from the database.

The group that is used as recycle bin is referenced by the `<RecycleBinUUID>`
meta data element.  Changes to the recycle bin SHOULD be reflected by
`<RecycleBinChanged>`. 

If a new group is created for the purpose of being used as a recycle bin, it
SHOULD have the standard icon `43` (see (#standard-icons)).

### Deleted Objects

When a group or an entry is deleted, an implementation MAY add a note about
this deletion in `<DeletedObjects>` of the data section.  This is to allow proper
two-way remote synchronization of databases.

Such a note is a `<DeletedObject>` element, which MUST have a `<UUID>` element
of type *UUID* (*KEY*) with the UUID of the deleted object and a `<DeletionTime>`
element of type *DATETIME* recording the date and time this item was deleted.

If an implementation keeps track of deleted groups and entries, it SHOULD also
provide an automatic or manual maintenance feature to clean up old deleted object
notes.

# Standard Icons

A graphical implementation SHOULD provide a set of standard icons, which groups and
entries can reference by an *INTEGER* identifier.

A total of 69 standard icons are specified.  Implementations MAY provide
a reduced set, but SHOULD at least support displaying all specified icons.

## List of Specified Standard Icons

00
:   An icon representing a generic password or key

01
:   An icon representing a network or networking

02
:   An icon representing a warning

03
:   Server

04
:   Clipboard or pinned notes

05
:   An icon representing language or communication

06
:   Set of blocks are packages

07
:   Text editor

08
:   An icon representing a network socket

09
:   An icon representing a user's identity

10
:   Address book

11
:   Camera or pictures

12
:   Wireless network

13
:   Key ring or set of keys

14
:   An icon representing electric power or energy

15
:   Scanner

16
:   An icon representing browser favorites or bookmarks

17
:   Optical storage medium

18
:   Monitor or display

19
:   E-Mail or letter

20
:   Gears or icon representing configurable settings

21
:   An icon representing a todo or check list

22
:   Empty text document

23
:   Computer desktop

24
:   An icon representing an established remote connection

25
:   E-Mail inbox

26
:   Floppy disk or save icon

27
:   An icon representing remote storage

28
:   An icon representing digital media files

29
:   An icon representing a secure shell

30
:   Console or terminal

31
:   Printer

32
:   An icon representing disk space utilization

33
:   An icon representing launching a program

34
:   Wrench or icon representing configurable settings

35
:   An icon representing a computer connected to the Internet

36
:   An icon representing file compression

37
:   An icon representing a percentage

38
:   An icon representing a Windows file share

39
:   An icon representing time

40
:   Magnifying glass or an icon representing search

41
:   Splines or an icon representing vector graphics

42
:   Memory hardware

43
:   Recycle bin

44
:   Post-it note

45
:   Red cross or icon representing canceling an action

46
:   An icon representing usage help

47
:   Software package

48
:   Closed folder

49
:   Open folder

50
:   TAR archive

51
:   An icon representing decryption

52
:   An icon representing encryption

53
:   Green tick or an icon representing OK

54
:   Pen or signature

55
:   Thumbnail image preview

56
:   Address book

57
:   An entry representing tabular data

58
:   An icon representing a cryptographic private key

59
:   An icon representing software or package development

60
:   An icon representing a user's home folder

61
:   A star or an icon representing favorites

62
:   Tux penguin

63
:   Feather or an icon representing the Apache web server

64
:   Apple or an icon representing macOS

65
:   An icon representing Wikipedia

66
:   An icon representing money or finances

67
:   An icon representing a digital certificate

68
:   A mobile device


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
