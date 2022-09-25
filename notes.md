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

- The `io.Writer` interface is for instance extended by the interface `net.Conn`, which is the return type of `net.Dial` used both in TCP, UDP and IP networks.

- The type `*bytes.Buffer` implements `io.Writer`. Notice that it is the *pointer type* and not `bytes.Buffer` that implements `io.Writer`. This is not an issue when calling a method on such a struct (since a method is bound to its receiver base type, in this case `bytes.Buffer`, and is visible inside selectors for both this type and the corresponding pointer type), but when given as an argument to a function (e.g. `fmt.Fprintln`) it must be of the correct type, namely `*bytes.Buffer`.

- [TODO: Buffered writers?]