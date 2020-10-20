# Programs and Processes
**fork/exec/exit/wait, C programs (basic pointers, flags, …)**

**getter/setter, identities, crashes, zombie/orphans**

## fork()
pid > 0 -> success, current execution within parent

`pid = 0 -> success, current execution within child`

pid = −1 -> failure, no child created, errno set

```
#include <unistd.h> int pid;
int status = 0;
pid = fork ();
// Parent: PID IS NON-ZERO
if (pid > 0) {
    printf (“Parent: child has pid=%d”, pid); 
    pid = waitpid(pid, &status, 0); // Parent typically blocks or waits until the child terminates by using wait or waitpid
} else if (pid == 0) {
    printf (“child here”);
    exit(status);
} else {
    perror (“fork problem”); 
    exit (-1);
}
```
## wait(NULL)
`pid_t wait (int *stat_loc); // any child`

`pid_t waitpid (pid_t pid, int *stat_loc, int options); // spec. child`

wait(): Returns process ID of terminated child on success, else returns -1 and errors

waitpid(): Returns process ID of child on state change, else return -1 and errors

stat_loc: why did child exit? (don’t care, NULL or 0) 
return parameter, if not NULL, must allocate it!

`wait (NULL); // OR`

`int stat_loc;`

`wait (&stat_loc); // “output” param`


```
// Why we need fork? Multi-tab browser.
const int MaxURL = 100; // #define MaxURL 100 int i, num_urls;
char *URLs [MaxURL];
pid_t child_pids [MaxURL];
// assume num_urls and URLs have been set
for (i=0; i<num_urls; i++) {
    if ((child_pids[i] = fork()) == 0) {
        render_page (URLs[i]);
        break; 
    }
}
for (i=0; i<num_urls; i++) wait (NULL);
```

## exec()
`int execl (const char *path, const char *arg0,  const char *arg1, ... (char *) 0);`

arg0 is just the name of the executable “prog” remainder are optional, with 0 or NULL terminating

exec() overwrites current process image with a new one

Executes program named by 1st argument Pathname+executable name (e.g. “/usr/jon/prog”)

exec() will not return on success, else they return -1 and set the errno.

```
#include <unistd.h>
int pid;
pid = fork ();
if (pid > 0) {
    pid = wait (&status);
} else if(pid==0){ //child // does not return if successful!
    execl (“/bin/prog”, “prog”, (char*)0);
    perror (“exec failed”); 
    exit(-1);
} else {
    perror (“fork problem”);
    exit (-1);
}
```
## Other
If child exits when parent is not waiting -> Child becomes a “zombie”

Parent exits while child is running -> Child becomes an orphan

`fork () returns pid of the child to the parent`

`getpid () returns the pid of the calling process`

`getppid () returns my parents pid`

# I/O
**low-level, high-level, redirection, semantics, binary, random, buffering, errors, control, formatting**

`int fflush (FILE *stream);`

`fprintf (F, “your output is %d”);`

`fflush (F); // force it`

`stderr; // always causes buffer to be flushed s`

`tdout; // with ‘\n’ as well`

Sometime we want to “force” buffer to spill to the OS –> Because see data immediately (OS auto-flushes K sec)

## Open/Close
`FILE* fopen(const char* fileName, const char* mode)`

filename can be absolute / relative path of the file

mode can be “r”, “w” (create a file if not present), “a” etc.

returns a pointer to the file

`int fclose ( FILE* stream)`

stream is the pointer to file

returns 0 on success, -1 otherwise

## Character I/O
`int fgetc(FILE *stream);`

reads a char from the stream

returns an int (ASCII) value of char; returns EOF at the end of file **eg: fgetc(fptr)**

`int fputc(int char, FILE *stream);`

writes a char into the stream; depending upon the mode of stream,

it will overwrite (“r+”), append (“a”) or create a new file and write (“w), etc.
**eg: fputc(100, fptr) where 100 is ascii value of ‘d’**

## String I/O
`char *fgets(char *buf, int nsize, FILE *inf);`

reads nsize-1 characters or up to a newline ‘\n’ or EOF into buf
appends ‘\0’ at the end of buf

returns buf or NULL if at EOF or an error occurs respectively
` eg: fgets(buf, 60, fptr)` where buf is a char array of size 60

`int fputs(const char *buf, FILE *outf);`

writes the string pointed by buf to stream pointed by outf
without the null character

returns a non negative integer on success or EOF if an error occurs
**eg: fputs(buf, fptr) where buf is a char array or char stream pointed by buf**

## Formatted I/O
`int fprintf(FILE *outf, const char *fmt, args)`

writes formatted data to given outf stream

**eg: int x = 3; fprintf(fptr, “%d”, x);**

`int sprintf(char *string, const char *fmt, args)`

writes formatted output to given char stream

**eg: sprintf(str, “%d %d %s”, 12, x, “hello”);**

## Formatted I/O contd

`fscanf(FILE *in, const char *fmt, <ptr args...allocated>);`

reads inputs from the stream pointer in

**eg: fscanf(fptr, “%s”, buff), where buff is char array**

`sscanf(const char *in, const char *fmt, <ptr args...allocated>);`

reads input from the char string pointed by in

**eg: sscanf("11 12 34.07 keithben", "%d %d %f %s %s", &i1, &i2, &flt, str1, str2);**

## Binary I/O

`fwrite(void *buffer, size_t size, size_t nitems, FILE *f);`
`fread(void *buffer, size_t size, size_t nitems, FILE *f);`

```
typedef struct S{
    int ss; // 8 digits
    int phone; // 10 digits 
} info_t;
info_t mine = {12345678, 5384937474}; 
fprintf (F, “%d %d”, mine.ss, mine.phone);
fwrite ((void *)&mine, sizeof (info_t), 1, F);
// ascii file: 12345678 538493747 = (8+10)*1 bytes
// binary file: ^a93e&^%8 = (4+4)*1
fread ((void *)&mine, sizeof (info_t), 1, F); 
printf (“%d %d\n”, mine.ss, mine.phone);
```


## Low Level File Operations
`int open (char *pname, int flags, mode_t mode);`

Attempts to open file Returns descriptor.

`int close (int fd);` 

Closes file.

`ssize_t read (int fd, char *buffer, size_t n);`

Reads n bytes from fd to buffer.`

Returns # of bytes acutally read up to EOF Returns 0 if at EOF, -1 if an error

`ssize_t write (int fd, char *buffer, size_t n);`

Writes n bytes from buffer to fd. Will overwrite any data in the file if it exists (unless O_APPEND)

Returns number of bytes actually written If this is < n, usually a problem As always, -1 is an error

## File Flags
`O_CREAT // Create file if it doesn’t exist.`

`O_RDONLY // Read-only flag.`

`O_WRONLY // Write-only flag.`

`O_RDWR // Read-Write flag.`

```
#define BUFSIZE 512
#define PERM 0644 // user can read/write, group/other can only read 
void copyfile(const char *name1, const char *name2) {
    int infile, outfile; ssize_t nread; char buffer [BUFSIZE];
    infile= open (name1, O_RDONLY);
    outfile= open (name2, O_WRONLY | O_TRUNC | O_CREAT, PERM); 
    while ((nread = read(infile, buffer, BUFSIZE)) > 0) 
        write (outfile, buffer, nread); 
    close (infile); close (outfile);
}
```

## Rerouting with dup2
`dup2(int old_fd, int new_fd);`

Reroutes all reads/writes from new_fd to old_fd. <<<<<< IMPORTANT

`cat old_fd > new_fd`

# File systems
**files and directories**
**links, i-nodes, masks, permissions**

`int stat (const char* filename, struct stat *buf)`

Stat system call is used to get the file attributes

A stat structure pointed by buf is used to store the attributes

Files must be in directories you have execute permissions for or the struct will be blank

Other variations: fstat() and lstat()

`st_uid`: user id of owner 

`st_size`: total size in bytes 

`st_mode`: protection mode

`st_ino`: inode numbe
