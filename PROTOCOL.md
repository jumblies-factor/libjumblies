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

## Initialisation
The client sends a `HI <arch triplet>`, and the server responds with `GREET`.

The client can now ask for a list of jobs with `JOBS`, probe their dependencies
with `PREREQ`, and check the type of files with `DESC`.

After finding one or more jobs they want, they `ENROLL` in those jobs, and send
a ready when they are done.

The first argument to the response to a `PREREQ` is the main work runtime,
which is responsible for communicating with the client controller.

## Work loop
The client sends an `READY` request when it wants another task.

The server's reponse must be forwarded directly to the work runtime as a 
`jumblies::proto::arg_list` (in the case of a shared object), or as a string
(in all other cases).

When the task is complete, the client must forward the `DONE` request, with 
arguments from the returned `jumblies::proto::arg_list`.

The runtime may also send `UPDATE` requests, which should also be forwarded to
the remote server.

## Push requests
Whilst almost all requests are sent from the client, the server may need to
interject with its own requests. These, and reponses thereof, should have an
exclaimation mark immediately before the verb (i.e. `!LAYOFF`).

## All verbs
### Client verbs
| Verb     | Arguments        | Description                                  |
| -------- | ---------------- | -------------------------------------------- |
| `HI`     | `<arch triplet>` | Used to start a connection                   |
| `JOBS`   |                  | Lists all the jobs available for this client |
| `ENROLL` | `<job id>`       | Applies for a job                            |
| `PREREQ` | `<job id>`       | Asks for a list of files required for a job  |
| `DESC`   | `<file hash>`    | Asks for the metadata of a file hash         |
| `GET`    | `<file hash>`    | Asks for the contents of a file hash         |
| `UPDATE` | `<job id> ...`   | Sends some update information to the server  |
| `DONE`   | `<job id> ...`   | Sends the result of a task to the server     |

## Special verbs
`!GOAWAY` means that the client should disconnect. Any work is now void, and 
should be stopped immediately.

`BYE` is the client equivalent: the server should disconnect from the client,
and remove it from any pending task queues.

`!LAYOFF (job id)...`  means that the client should stop working on the 
specified job(s).
