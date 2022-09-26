# Golang notes

## I/O

### File descriptors and standard I/O

On a Unix system, a *file descriptor* is an integer identifying a file. It may be a file on disk, or the descriptor may refer to things we do not usually call files, like directores or input/output systems like keyboards or a display screen. Each process has a *file descriptor table* maintained by the kernel. To perform I/O, the process passes the relevant file descriptor to the kernel by a system call, and the kernel looks up the descriptor in the system-wide *file table* which contains all files open by any process on the system. The file table records the *mode* with which the file has been opened (e.g. read-only, read-write), and it also indexes into the *inode table* which describes the actual underlying files. Thus a process need not know the actual location of a file to access it, only its file descriptor.

File descriptors may be inherited by a process from its parent. Every process inherits at least three descriptors:

| Value | Name            |
|-------|-----------------|
| 0     | Standard input  |
| 1     | Standard output |
| 2     | Standard error  |

In Go the type `os.File` (a struct whose implementation is OS-specific) is used to represent open file descriptors. Looking at the definition of e.g. `os.Stdout` we see that

    var Stdout = NewFile(uintptr(syscall.Stdout), "/dev/stdout")

where `syscall.Stdout` is just `1` and `os.NewFile` has return type `*os.File`. (Of course the implementation of `os.NewFile` is also OS-specific since its return type is.)

We will return to file I/O below.


### Writers

The `io.Writer` type is defined as follows:

    type Writer interface {
        Write(p []byte) (n int, err error)
    }

Hence, a `Writer` is something that can be *written to*, not something that *itself writes*. It thus follows the [naming convention](https://go.dev/doc/effective_go#interface-names) that one-method interfaces are named by the method name plus the suffix '-er'.

This is the type accepted by functions such as `fmt.Fprintln` which is called by `fmt.Println`:

    func Fprintln(w io.Writer, a ...any) (n int, err error) {
        // ...
    }
    
    func Println(a ...any) (n int, err error) {
        return Fprintln(os.Stdout, a...)
    }

In particular, `os.Stdout` must implement the `io.Writer` interface, i.e. have a `Write` method (the implementation of this method is of course also OS-specific). And we may indeed use `Write` to print to standard output:

    os.Stdout.Write([]byte("Hello world!\n")) // prints "Hello world!" to terminal

Here we of course must first convert the string `"Hello world!\n"` to the type `[]byte`.

As far as I can tell the `io.Writer` interface is not implemented by the `io` package; instead the purpose of the `io` package is simply to provide interfaces to I/O primitives for other packages to implement.

We give some examples of structs that implement and interfaces that extend `io.Writer`:

- Standard output `os.Stdout` as described above.

- The `io.Writer` interface is for instance extended by the interface `net.Conn`, which is the return type of `net.Dial` used both in TCP, UDP and IP networks.

- The type `*bytes.Buffer` implements `io.Writer`. Notice that it is the *pointer type* and not `bytes.Buffer` that implements `io.Writer`. This is not an issue when calling a method on such a struct (since a method is bound to its receiver base type, in this case `bytes.Buffer`, and is visible inside selectors for both this type and the corresponding pointer type), but when given as an argument to a function (e.g. `fmt.Fprintln`) it must be of the correct type, namely `*bytes.Buffer`.

- [TODO: Buffered writers?]


### Readers

The `io.Reader` interface is defined

    type Reader interface {
        Read(p []byte) (n int, err error)
    }

The intention is that the `Read` method reads something into the slice `p`. Since a slice is in effect a pointer to an array, it can be modified in place without passing a pointer to `Read`. The following program demonstrates this behaviour using the implementation `strings.Reader`:

    package main

    import (
        "fmt"
        "strings"
    )

    func main() {
        reader := strings.NewReader("Hello world")
        p := make([]byte, 11)
        reader.Read(p)
        fmt.Println(p)
    }

This will output

    [72 101 108 108 111 32 119 111 114 108 100]

which is a slice containing the Unicode code points (in Go terminology the *runes*) of the characters in the string `"Hello world"`. To recover the string we may apply the function `string` to `p`, or we may use `fmt.Printf("%s\n", p)` to print the string instead.

The `io` package also provides the function

    func ReadAll(r Reader) ([]byte, error)

which reads until an error or EOF, taking care of resizing the slice `p` so that the entire contents of `r` is read. Notice that it is the `io` package which provides this function, so it is implemented only in terms of the `Read` method of `r`.

Examples of readers:

- Standard input, `os.Stdin`. Its `Read` method reads from standard input (usually the terminal) until the slice `p` is full, or until the end of the input. If the input (include the line feed at the end of the input) is strictly smaller than `len(p)`, then the remaining entries in `p` are filled with zeros (`0` being the Unicode code point for the null character). If the input is larger than `len(p)`, then the remaining input seems to be buffered by the terminal, and subsequent calls to `Read` will read from the buffer.
  
  If we use `io.ReadAll` to read from the terminal, then we must manually insert an EOF by typing CTRL+D.

- The package `bufio` implements `io.Reader` in the type `bufio.Reader`, which has the constructor
  
      func NewReader(rd io.Reader) *Reader
  
  The package provides several methods on `bufio.Reader`, including
  
      func (b *Reader) ReadString(delim byte) (string, error)
  
  which reads until the first occurrence of `delim` in the input, and returns a string containing the data up to and including `delim`. If called on `bufio.NewReader(os.Stdin)` and the input contains multiple occurrences of `delim` (e.g. if `delim` is a space, since the input is only sent to the method when pressing enter), then it again seems like the remaining input is buffered and read on subsequent calls to `ReadString`.

- The interface `net.Conn` also extends `io.Reader` (as it did `io.Writer`).


### Scanners

The `Scanner` type is provided by the `bufio` package:

    type Scanner struct {
        r            io.Reader // The reader provided by the client.
        split        SplitFunc // The function to split the tokens.
        maxTokenSize int       // Maximum size of a token; modified by tests.
        token        []byte    // Last token returned by split.
        buf          []byte    // Buffer used as argument to split.
        start        int       // First non-processed byte in buf.
        end          int       // End of data in buf.
        err          error     // Sticky error.
        empties      int       // Count of successive empty tokens.
        scanCalled   bool      // Scan has been called; buffer is in use.
        done         bool      // Scan has finished.
    }

We usually use the `bufio.NewScanner` function to create scanners:

    func NewScanner(r io.Reader) *Scanner {
        return &Scanner{
        r:            r,
        split:        ScanLines,
        maxTokenSize: MaxScanTokenSize,
        }
    }

The method `Scan` is used to scan the reader `r`, placing the next token in the `token` field of the `Scanner`, after which the `Text` method can be called to return `token`. The `Scan` method in turn calls the split function stored in the `split` field. Such split functions have the type

    type SplitFunc func(data []byte, atEOF bool) (advance int, token []byte, err error)

so a split function simply returns the next token, and `Scan` in turn saves it in the `token` field.

The default split function is `ScanLines`, which returns each line of text, stripped by trailing end-of-line markers (both `"\r\n"` and `"\n"`). An EOF also returns a line of text even if it doesn't contain a newline. Other split functions provided by `bufio` are `SplitBytes` (which returns individual bytes) and `SplitRunes` (which returns runes, i.e. Unicode code points). Another split function is `ScanWords`, which returns words separated by Unicode white space characters (e.g. `' '`, `'\n'`, `'\r'`).

The split function may be assigned to a scanner using the method `Split`.

[TODO: When to use scanners instead of readers. Scanners and `net.Conn`?]


### File I/O

The generalised way of opening a file is the `io` function

    func OpenFile(name string, flag int, perm FileMode) (*File, error)

where `name` is the file path, `flag` is a combination of flags, and `perm` is the mode with which to open the file (i.e. which permissions should be given). The following flags are possible:

| Flag       | Value    | Meaning                                     |
|------------|---------:|---------------------------------------------|
| `O_RDONLY` | 0x0      | Open the file read-only.                    |
| `O_WRONLY` | 0x1      | Open the file write-only.                   |
| `O_RDWR`   | 0x2      | Open the file read-write.                   |
| `O_APPEND` | 0x400    | Append data to the file when writing.       |
| `O_CREATE` | 0x40     | Create a new file if none exists.           |
| `O_EXCL`   | 0x80     | Used with `O_CREATE`, file must not exist.  |
| `O_SYNC`   | 0x101000 | Open for synchronous I/O.                   |
| `O_TRUNC`  | 0x200    | Truncate regular writable file when opened. |

Of these, exactly one of the first three must be specified, and the remaining flags may be or'ed in to control behaviour. Notice that apart from `O_RDONLY` and `O_WRONLY`, each flag have `1`s in different bits.

The `io` package provides functions `Open` (using the single flag `O_RDONLY`) and `Create` (with flags `O_RDWR|O_CREATE|O_TRUNC`) as user-friendly alternatives, both defined in terms of `OpenFile`. To append to a file using the functions provided by `io`, it seems like using `OpenFile` is necessary.

The `*os.File` type implements both `io.Reader` and `io.Writer`. (The methods `Read` and `Write` are defined in terms of OS-specific helper functions.) Hence we may use all the above methods to read from and write to files, given the right permissions. Notice that the behaviour of `Write` depends on the flags given to `OpenFile`: the flag `O_APPEND` being included or not determines whether the contents of the write should overwrite the existing content of the file (if any) or be appended to the file.