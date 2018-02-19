%%%

    title = "KeePass2 XML Payload Format"
    abbrev = "KeePass2 XML"
    category = "info"
    docname = "keepass2-xml-rfc"
    ipr= "trust200902"
    area = "Internet"
    keyword = ["keepass", "keepassxc"]

    date = 2018-02-19T00:00:00Z
    
    [pi]
    symrefs = "no"
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
4.0) files.  The payload is encoded as XML ("Extensible Markup Language" [@xml]).

KDBX file format versions 3.1 and 4.0 use the same XML structure with minor
differences in the tags supported.  A note is added where KDBX 3.1 and 4.0
differ.

The XML payload itself is mostly self-contained, but may reference data
from the outer KDBX container. A note is given where this is the case.


# Security Considerations

KDBX is an encrypted container format.  The security of the XML payload
described in the document depends on the strength of the cipher and
passphrase / key file used for encrypting the encapsulating KDBX container.

In addition, all security considerations for XML [@xml] apply as well,
including the use of recursive XML entities, which can consume an
arbitrarily large amount of memory on the user's system during parsing.


<reference anchor="xml" target="https://www.w3.org/TR/2008/REC-xml-20081126/">
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
