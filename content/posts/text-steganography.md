+++
date = "2014-01-26T21:44:54-05:00"
title = "Text Steganography"
tags = [ "steganography", "security", "encoding", "cryptography" ]
icon = "padlock15.svg"
+++

Steganography is the practice of hiding data within other data, such that a third party doesn't suspect the presence of the hidden data (the "payload") inside of the readily apparent data (the "carrier" or "canvas").  The type of the carrier is known as a "channel," which in this case is a Unicode text document.  Some parts can be adapted to plain ASCII.  All of these methods are easily detectable programatically, as they involve unusual character sequences, such as Cyrillic characters in an otherwise Latin word.

Steganography is useful in situations where cryptography is prohibited. Cryptography is very easy to detect as almost all decent encryption algorithms produce a uniformly random string of bytes for their ciphertext. The only other algorithms that do that are for compression. If a block of data is uniformly random and does not appear to be compressed using a standard algorithm (of which there are many: gzip, bz2, etc.), it is probably encrypted. Also, most protocols involving cryptography, e.g., TLS/SSL use standard headers for negotiating keys and cipher suites, which makes it trivial to detect even without statistical analysis.

Detection of all of these methods is very simple, but not as easy as detecting cryptography.

The following are various strategies for steganography of arbitrary data inside of Unicode text documents.

<!--more-->

## Here are methods to encode the payload in:

### Spaces

_Spaces: the final frontier_. Space characters are not limited to ASCII 0x20.

[Unicode 6.2](http://www.unicode.org/versions/Unicode6.3.0/ch06.pdf) lists spaces in table 6-2 (page 194):

| Unicode Code Point | Description |
|--------------------|-------------|
| U+0020             | Space
| U+00A0             | No-break Space
| U+1680             | Ogham Space Mark (most fonts display as a dash)
| U+180E             | Mongolian Vowel Separator
| U+2000             | En Quad
| U+2001             | Em Quad
| U+2002             | En Space
| U+2003             | Em Space
| U+2004             | Three-Per-Em Space
| U+2005             | Four-Per-Em Space
| U+2006             | Six-Per-Em Space
| U+2007             | Figure Space
| U+2008             | Punctuation Space
| U+2009             | Thin Space
| U+200a             | Hair Space
| U+202f             | Narrow No-Break Space
| U+205f             | Medium Mathematical Space
| U+3000             | Ideographic Space

Unicode also classifies these code points as spaces, even though they have no width (i.e., spaces that take up no space):

| Unicode Code Point | Description |
|--------------------|-------------|
| U+200B             | Zero-Width Space
| U+FEFF             | Zero-Width No-Break Space (BOM)

In ASCII, newlines were specified using either a CR/LF combination (Windows), LF (*NIX) or CR (old Mac).  To "simplify" this, Unicode has:

| Unicode Code Point | Description |
|--------------------|-------------|
| U+2028             | Line Separator
| U+2029             | Paragraph Separator

Typically, a space is typeset with 1/4 em width.  So, that should display identically to U+2005.

This simplified implementation only uses `U+200B` and `U+FEFF` to encode arbitrary data.  You could append data into a canvas using shell redirection, `cat` and this utility.

{% include_code 'Unicode Spaces Steganography' lang:ruby space-stego/unicode-space-steganography.rb %}

### Cyrillic Characters

This strategy is applicable more generally to alphabets containing Latin-alphabet-looking characters.  Unicode has quite a few characters (95,221 or so), some of which are indistinguishable from others in most fonts.  I found that Cyrillic characters are generally well-supported in common fonts.

| Latin | Cyrillic | Cyrillic Code Point |
|-------|----------|---------------------|
| A     | А        | U+0410
| a     | а        | U+0430
| B     | В        | U+0412
| C     | С        | U+0421
| c     | с        | U+0441
| E     | Е        | U+0415
| e     | е        | U+0435
| H     | Н        | U+041D
| I     | І        | U+0406
| i     | і        | U+0456
| K     | К        | U+041A
| M     | М        | U+041C
| m     | м        | U+043C
| O     | О        | U+041E
| o     | о        | U+043E
| P     | Р        | U+0420
| p     | р        | U+0440
| S     | Ѕ        | U+0405
| T     | Т        | U+0422
| y     | у        | U+0443
| X     | Х        | U+0425
| x     | х        | U+0445

So there is the _option_ of replacing a character in the Latin column with a character in the Cyrillic column. The byte-offset and bit-offset positions of the payload are tracked in the algorithm. Whenever a replaceable character is read from the canvas, a test is performed to see if the replacement should occur (in C):

```c
int perform_replacement = canvas[byte_offset] & bit_offset
```

Whenever a potential replacement is performed (i.e., a viable replacement character from the canvas is either replaced or not), the `bit_offset` is incremented. When `bit_offset == 8`, `bit_offset` is zeroed and `byte_offset` is incremented. Note that the range of `bit_offset` is 0-7.

### Text direction characters

Unicode has support for indicating that text should suddenly start flowing in a given direction (LTRM / RTLM) as well as reversing the direction

These can actually be nested up to 61 levels deep just in case you have a deeply nested quote from someone, e.g., an Arabic document quoting English that contains Arabic that contains English (etc., etc., ...)
U+200E, U+200F, U+202A, U+202B, U+202C, U+202D, U+202E

### Annotation characters
| Unicode Code Point | Description |
|--------------------|-------------|
| U+FFF9             | Interlinear Annotation Anchor
| U+FFFA             | Interlinear Annotation Separator
| U+FFFB             | Interlinear Annotation Terminator

These symbols are for general annotations on documents. The program rendering the document will probably show the annotations, but it should not display the annotation characters themselves.

### Language tag characters
[RFC 2482](http://tools.ietf.org/search/rfc2482) provides a method for embedding the language of text as metadata inside of a Unicode text document.  Nobody uses this anymore, and it was deprecated in 5.1, however deprecated doesn't mean removed. The Unicode spec says that these symbols should not be displayed, so I suppose you could hide data in fake language tags.

### The Unicode Confusables
The canonical reference on Unicode symbols that look like other symbols is the [Recommended confusable mapping for IDN](http://www.unicode.org/Public/security/revision-02/confusables.txt)

It would be very easy to write a script that uses all of these, however most are either not found in any common font or don't really look that similar.
