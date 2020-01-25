# riband
_jumblies protocol_

## Overview
This protocol should run over TLS for obvious security reasons. Both client and
server should check each other's certificate.

The protocol is mosty line based, with a `0x0a` byte representing the newline.

Whilst I generally abhor such protocols, most of the data is in the form of a
string, so it makes sense in this case.

Strings are encoded in utf-8.

'Special characters' refer to unicode control chars, or any of `"'>%`

### Structure
`<verb> [argument 1] [argument 2] ...`

Arguments can take one of  the following forms:
| Format       | Description                                                  |
| ------------ | ------------------------------------------------------------ |
| `argument`   | An argument with no special chars, spaces or newlines        |
| `"argument"` | A c-style string literal                                     |
| `%argument`  | An even number of hex bytes representing a byte sequence     |
| `>count`     | Specifies the length of the payload section for this argument|

### Quoted strings
The supported escape sequences (all beginning with a `\`):

| Sequence | Interpretation         |
| -------- | ---------------------- |
| `\r`     | `0x0d`: CR             |
| `\n`     | `0x0a`: LF             |
| `\"`     | `0x22`: `"`            |
| `\nnn`   | Exactly 3 octal digits |
| `\xnn`   | Exactly 2 hex digits   |

No unescaped quotes or unescaped unicode control chars are allowed in the 
quoted string.

### Hex string
Should be used sparingly (ideally never), as payload strings are far more 
efficient (and faster).

### Payload strings
Payload strings are the preferred format for most argument types, as they are
low in overhead. The arguments are read in order, and all arguments will be
parsed before the next line is read.

### Special verbs
`GOAWAY` means that the client should disconnect. Any work is now void, and 
should be stopped immediately.

`BYE` is the client equivalent: the server should disconnect from the client,
and remove it from any pending task queues.

## Initialisation
| Client              | Master              | 
| ------------------- | ------------------- |
| `HI <arch triplet>` | `GREET` or `GOAWAY` |
| `JOBS`              | `AVAIL (job id)...` |
| `ENROLL <job id>`   | `WORK (job DSO)...` |

The client loads the DSO on the WORK response, and is now ready to start
accepting tasks.

## Work loop
The client sends an `READY` request when it wants another task.

The server may respond with a `TASK` response, which must be forwarded directly
to the work DSO as a `jumblies::proto::arg_list`.

The server may respond with a `LAYOFF` command instead, at which point the 
client may `ENROLL` in another job, ask for the list of `JOBS` again, or just
say `BYE`.

When the task is complete, the client must send a `DONE` command, with 
arguments from the returned `jumblies::proto::arg_list`.
