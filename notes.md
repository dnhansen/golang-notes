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

This is the type accepted by functions such as `fmt.Fprintln` which is called by `fmt.Println`:

    func Println(a ...any) (n int, err error) {
        return Fprintln(os.Stdout, a...)
    }

In particular, `os.Stdout` must implement the `Writer` interface. [TODO: How does it acquire this method?]