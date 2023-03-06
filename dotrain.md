# .rain files

For humans to write code they need a medium in which to write the code in.

Typically code is written in files/documents that are binary data representing
strings of printable characters and whitespace for humans to read and write.

There's the usual concerns such as

- Character encoding: ascii, utf8, utf16, et.
- Compression algorithms over binary data
- Handling whitespace such as end of line characters

Once we get past these and have higher level abstractions over the binary data
like

- Character
- End of line
- Indentation

Then we can meaningfully talk about the structure of a .rain file as a human
experiences it. Moving from binary data to a human legible file is outside the
scope of this spec, and typically should be considered an implementation detail.
For example a utf8 encoded .rain document on a file system and a utf16 javascript
string containing the same document should be considered equivalent and valid.

## Allowed characters

ONLY ascii printable and whitespace characters are allowed in .rain files.

As a regex this is `[\s -~]`.

utf8 is backwards compatible with ascii so the codepoints are the same in either
encoding.

## Self describing

