### Introduction

It has often been said that the art of computer programming is part
managing complexity and part naming things. I contend that this is
largely true with the addition of "and sometimes it requires drawing
boxes". 

In this article I'll name some things and manage some complexity while
writing a small C program that is loosely based on the program
structure I [discussed earlier][1], but different. This one will do
something. Grab your favorite beverage, your favorite editor and
compiler, crank up some tunes and let's write a mildly interesting C
program together.

### Philosophy of A Good UNIX C Program

The first thing to know about this C program is that it's a [UNIX][2]
command-line tool. This means that it runs on, or can be ported to,
operating systems that provide a UNIX C run-time environment. When
UNIX was first invented at Bell Labs, it was imbued from the beginning
with a design philosophy: programs do one thing, do it well and act on
files.  While it makes sense to do one thing and do it well, the part
about acting on files seems a little out of place. 

It turns out that the UNIX abstraction of a "file" is very powerful. A
UNIX file is a stream of bytes that ends with an End Of File (EOF)
marker. That's it. Any other structure in a file is imposed by the
application and not the operating system. The operating system
provides system calls which allow a program to perform a set of
standard operations on files: open, read, write, seek, and close
(there are others but those are the biggies). Standardizing access to
files allows different programs to share a common abstraction and work
together even when they were implemented by different people in
possibly different languages.

Having a shared file interface makes it possible to build programs
that are **composable**. The output of one program can be the input of
another program. The UNIX family of operating systems provides
three files by default whenever a program is executed:
standard in (```stdin```), standard out (```stdout```) and standard
error (```stderr```). Two of these files are opened in write-only
mode; ```stdout``` and ```stderr```, while ```stdin``` is opened
read-only. We see this in action whenever we use file redirection
in a command shell like ```bash```:

```bash
   $ ls | grep foo | sed -e 's/bar/baz/g' > ack
```

This construction can be described briefly as; the output of ```ls```
is written to stdout which is redirected to the stdin of ```grep```
whose stdout is redirected to ```sed``` whose stdout is redirected to
write to a file called 'ack' in the current directory.

We want our program to play well in this ecosystem of equally flexible
and awesome programs, so lets write a program that reads and writes files.

### Concept: MeowMeow - A Stream Encoder/Decoder

When I was a dewy-eyed kid studying computer science in the &lt;mumbles&gt;'s,
there were an actual plethora of encoding schemes. Some of them were
for compressing files, some were for packaging files together and
others had no purpose but to be excrutiatingly silly. An example of
the last is the [MooMoo encoding scheme][3].

For our example, I'll update this concept for the [2000s][4] and
implement a MeowMeow encoding since the Internet loves cats. The basic
idea here is to take files and encode each nibble (half of a byte)
with the text "meow". A lower-case letter indicates a zero and
upper-case indicates a one. Yes, it will balloon the size of a file
since we are trading four bits for thirty two bits. Yes, it's
pointless. But imagine the surpise on someone's face when this
happens:

```
   $ cat /home/your_sibling/.super_secret_journal_of_my_innermost_thoughts
   MeOWmeOWmeowMEoW...
```

This is going to be awesome.

### Implementation, Finally

The full source for this can be found on [GitHub][5] but I'm going to
talk through my thought process while writing it. The object here is
to illustrate how to structure a C program composed of multiple files.

Having already established that I want to write a program that encodes
and decodes files in MeowMeow format, I fired up a shell and issued the
following commands:

```bash
   $ mkdir meowmeow
   $ cd meowmeow
   $ git init
   $ touch Makefile     # recipes for compiling the program
   $ touch main.c       # handles command-line options
   $ touch main.h       # "global" constants and definitions
   $ touch mmencode.c   # implements encoding a MeowMeow file
   $ touch mmencode.h   # describes the encoding API
   $ touch mmdecode.c   # implements decoding a MeowMeow file
   $ touch mmdecode.h   # describes the decoding API
   $ touch README.md    # this awesome file
   $ touch .gitignore   # names in this file are ignored by git
   $ git add .
   $ git commit -m "initial commit of empty files"
```

In short, I created a directory full of empty files and committed them to git. 

Even though the files are empty, you can infer the purpose of each
from it's name. Just in case you can't, I annotated each ```touch```
with a brief description.

Normally a program starts as a single ```main.c``` file that is
simple, with only two or three functions that solve the problem. And
then the programmer rashly shows that program to a friend or her boss
and suddenly the number of functions in the file balloons to support
all the new "features" and "requirements" that pop up. First rule of
"Program Club" is don't talk about "Program Club".  The second rule is
minimize the number of functions in one file.

To be honest, the C compiler does not care one little bit if every
function in your program is in one file. But we don't write programs
for computers or compilers, we write them for other people (who are
sometimes us). I know that is probably a surprise, but it's true. A
program embodies a set of algorithms that solve a problem with a
computer, and it's important that people understand it when the
parameters of the problem change in unanticipated ways. People will
have to modify the program and they will curse your name if you have
all 2049 functions in one file.

So we good and true programmers break functions out, grouping like
functions into seperate files. Here I've got files **```main.c```**,
**```mmencode.c```**, and **```mmdecode.c```**. For small programs
like this, it may seem like overkill. But small programs rarely stay
small, so planning for expansion is a "Good Idea".

But what about those ```.h``` files? I'll explain them in general terms
later, but in brief those are called _header_ files and they can
contain C language type definitions and C preprocessor
directives. Header files should **not** have any functions in them.

### What The Heck is a Makefile?

I know all you cool kids are using the "Ultra CodeShredder 3000"
integrated development environment to write the next blockbuster app
and building your project consists of mashing on
"Ctrl-Meta-Shift-Alt-Super-B". But back in my day (and also today),
lots of useful work got done by C programs built with Makefiles. A
Makefile is a text file that contains recipes for working with files
and programmers use it to automate building their program binaries
from source (and other stuff too!).

For instance, this little gem:

```Makefile
   00 # Makefile
   01 TARGET= my_sweet_program
   02 $(TARGET): main.c
   03    cc -o my_sweet_program main.c
```

Text after an octothorpe/pound/hash is a comment, like line 00.

Line 01 is a variable assignment where the variable TARGET takes on
the string value ```my_sweet_program```. By convention, ok my
preference, all Makefile variables are capitalized and use underscores
to seperate words.

Line 02 consists of the name of the file that the recipe creates
and the files it depends on. In this case the target is ```my_sweet_program```
and the dependency is ```main.c```.

The final line, 03, is indented with an honest-to-Crom tab, not four
spaces. This is the command that will be executed to create the
target. In this case, we call ```cc``` the C compiler front-end to
compile and link ```my_sweet_program```.

Using a Makefile is simple:

```bash
   $ make
   cc -o my_sweet_program main.c
   $ ls 
   Makefile  main.c  my_sweet_program
```

The [Makefile][makefile] that will build our MeowMeow encoder/decoder is
considerably more sophisticated than this example, but the basic
structure is the same. Some targets create real files, some targets
are 'phony' since they don't create a specific file but act as a sort
of alias. I smell another article here about writing Makefiles. Before
I forget, the GNU Make command has excellent [documentation][6].

### Form Follows Function

The idea here is to write a program that reads a file, transforms
it and then writes it to another file. The following fabricated
command-line interaction is how I imagine using the program:

```bash
	$ meow < clear_text > meow_text
	$ unmeow < meow_text > declawed_text
	$ diff clear_text declawed_text
	$
```

We need to write code to handle command-line parsing and managing the
input and output streams. We need a function to encode a stream and
write it to another stream. And finally we need a function to decode a
stream and write it to another stream. Wait a second, I've only been
talking about writing one program but in the example above I invoke
two commands; ```meow``` and ```unmeow```? I know you are probably
thinking that this is getting complex as heck.

### Minor Side Track - ```argv[0]``` and the ```ln``` command

If you recall, the signature of a C main function is:

```C
int main(int argc, char *argv[])
```

where ```argc``` is the number of command line arguments
and ```argv``` is a list of character pointers (strings).
The value of ```argv[0]``` is the path of the file containing
the program being executed. Many UNIX utility programs with
complementary functions (e.g. compress and uncompress ) look
like two programs, but in fact they are one program with two names
in the filesystem. The two-name trick is accomplished by creating
a filesystem "link" using the ```ln``` command.

An example from ```/usr/bin``` on my laptop is:
```bash
   $ ls -li /usr/bin/git*
3376 -rwxr-xr-x. 113 root root     1.5M Aug 30  2018 /usr/bin/git
3376 -rwxr-xr-x. 113 root root     1.5M Aug 30  2018 /usr/bin/git-receive-pack
...
```

Here ```git``` and ```git-receive-pack``` are the same file with
different names. We can tell it's the same file since they have the
same i-node number (the first column). An i-node is a feature of the
UNIX filesystem and is super outside the scope of this article. Also
smells like another article.

Good and/or lazy programmers can use this feature of the UNIX
filesystem to write less code but double the number of programs they
deliver. First we write a program that changes it's behavior based on
the value of ```argv[0]``` and then we make sure to create links
(sometimes called hard-links) with the names that cause the behavior.

In our Makefile, the ```unmeow``` link is created using this recipe:

```Makefile
    # Makefile
    ...
    $(DECODER): $(ENCODER)
            $(LN) -f $< $@
	...
```

I tend to parameterize everything in my Makefiles, rarely using a
"bare" string. I group all the definitions at the top of the Makefile
which makes it easy to find them and change them. This makes a big
difference when you are trying to port software to a new platform
and you need to change all your rules to use ```xcc``` instead of
```cc```.

The recipe should appear relatively straightforward except for the two
built-in variables ```$@``` and ```$<```. The first is a short cut for
the target of the recipe; in this case ```$(DECODER)```. I remember that
because the at-sign looks like a target to me. The second, ```$<``` is
the rule dependency, in this case it resolves to ```$(ENCODER)```.

Things are getting complex for sure, but it's managed.

### Exploring ```main.c```

The structure of the ```main.c``` file for ```meow```/```unmeow```
should be familiar to my [long-time readers][1], and has the
following general outline:

```C
   /* main.c - MeowMeow, a stream encoder/decoder */

   /* 00 system includes */
   /* 01 project includes */
   /* 02 externs */
   /* 03 defines */
   /* 04 typedefs */
   /* 05 globals (but don't)*/
   /* 06 ancillary function prototypes if any */
   
   int main(int argc, char *argv[])
   {
     /* 07 variable declarations */
     /* 08 check argv[0] to see how the program was invoked */
     /* 09 process the command line options from the user */
     /* 10 do the needful */
   }
   
   /* 11 ancillary functions if any */
```

### Including Project Header Files

The second section, ```/* 01 project includes /*``` reads like this from the source:

```C
   /* main.c - MeowMeow, a stream encoder/decoder */
   ...
   /* 01 project includes */
   #include "main.h"
   #include "mmecode.h"
   #include "mmdecode.h"
```

The ```#include``` directive is a C pre-processor command that causes
the contents of the named file to be "included" at this point in the file.
If the programmer uses double-quotes around the name of the header file,
the compiler will look for that file in the current directory. If the file
is enclosed in &lt;&gt;, it will look for the file in a set of predefined
directories. 

The file [```main.h```][main_h] contains definitions and typedefs that are used
in [```main.c```][main_c]. I like to collect these things here in case I want
to use those definitions elsewhere in my program.

The files [```mmencode.h```][mmencode_h] and [```mmdecode.h```][mmdecode_h]
are nearly identical so I'll break down ```mmencode.h```.

```C
    /* mmencode.h - MeowMeow, a stream encoder/decoder */
    
    #ifndef _MMENCODE_H
    #define _MMENCODE_H
    
    #include <stdio.h>
    
    int mm_encode(FILE *src, FILE *dst, int verbose);
    
    #endif	/* _MMENCODE_H */
```

The **```#ifdef, #define, #endif```** construction is collectively
known as a "guard". This keeps the C compiler from including this file
more than once per file. The compiler will complain if it finds
multiple definitions/prototypes/declarations so the guard is a **must
have** for header files.

Inside the guard, there are only two things; a ```#include```
directive and a function prototype declaration. I include ```stdio.h```
here to bring in the definition of ```FILE``` which is used in
the function prototype. The function prototype can be included
by other C files to establish that function in the file's namespace.
You can think of each file a seperate **namespace** which means
variables and functions in one file are not usable by functions or
variables in another file. 

Writing header files is complex and it is tough to manage in larger
projects. Use guards.

### MeowMeow Encoding

The meat and potatoes of this program, encoding and decoding bytes
into/out of "MeowMeow" strings is actually the easy part of this project.

```C
    /* mmencode.c - MeowMeow, a stream encoder/decoder */
    ...
        while (!feof(src)) {
	    
          if (!fgets(buf, sizeof(buf), src)) {
            ferror(src);
            return -1;
          }
	      
          for(i=0; i<strlen(buf); i++) {
            lo = (buf[i] & 0x000f);
            hi = (buf[i] & 0x00f0) >> 4;
            fputs(tbl[hi], dst);
            fputs(tbl[lo], dst);
          }
	    }
```

In plain English, this loop reads in a chunk of the file while there
are chunks left to read (```feof()``` and ```fgets()```). Then it splits
each byte in the chunk into ```hi``` and ```lo``` nibbles. Remember, a
nibble is half of a byte, or four bits. The real magic here is
realizing that four bits can encode sixteen values. I use ```hi``` and
```lo``` as indices into a sixteen string lookup table, ```tbl```,
that contains the **MeowMeow** strings that encode each nibble. Those
strings are written to the destination ```FILE``` stream and we move
on to the next byte in the buffer.

The table is initialized with a macro defined in [```table.h```][table_h]
for no particular reason except to demonstrate including another project
local header file and I like initialization macros.

### MeowMeow Decoding

Alright, I'll admit it took me a couple of runs at this before I got
it working. The decode loop is similar; read a buffer full of **MeowMeow**
strings and then reverse the encoding from strings to bytes. 

```C
    /* mmdecode.c - MeowMeow, a stream decoder/decoder */
    ...
    int mm_decode(FILE *src, FILE *dst)
    {
      if (!src || !dst) {
        errno = EINVAL;
        return -1;
      }
      return stupid_decode(src, dst);
    }
```
	
Not what you were expecting?

Here, I'm exposing ```stupid_decode()``` via the externally visible
```mm_decode()``` function. When I say "externally" I mean outside this
file. Since ```stupid_decode()``` isn't in the header file, it isn't
available to be called in other files.

Sometimes we do this when we want to publish a solid public interface
but we aren't quite done noodling around with functions to solve a
problem. In my case, I've written a I/O intensive function that reads
eight bytes at a time from the source buffer to decode one byte to
write to the destination buffer.

```C
    /* mmdecode.c - MeowMeow, a stream decoder/decoder */
    ...
    int stupid_decode(FILE *src, FILE *dst)
    {
      char           buf[9];
      decoded_byte_t byte;
      int            i;
      
      while (!feof(src)) {
        if (!fgets(buf, sizeof(buf), src)) {
          ferror(src);
          return -1;
        }
        byte.field.f0 = isupper(buf[0]);
        byte.field.f1 = isupper(buf[1]);
        byte.field.f2 = isupper(buf[2]);
        byte.field.f3 = isupper(buf[3]);
        byte.field.f4 = isupper(buf[4]);
        byte.field.f5 = isupper(buf[5]);
        byte.field.f6 = isupper(buf[6]);
        byte.field.f7 = isupper(buf[7]);
        
        fputc(byte.value, dst);
      }
      return 0;
    }
```

Instead of using bit-shifting like in the encoder, I elected to
create a custom data structure called ```decoded_byte_t```.

```C
    /* mmdecode.c - MeowMeow, a stream decoder/decoder */
    ...

    typedef union {
      struct {
    #if __BYTE_ORDER != __BIG_ENDIAN
        unsigned int f0:1;
        unsigned int f1:1;
        unsigned int f2:1;
        unsigned int f3:1;
        unsigned int f4:1;
        unsigned int f5:1;
        unsigned int f6:1;
        unsigned int f7:1;
    #else
        unsigned int f7:1;
        unsigned int f6:1;
        unsigned int f5:1;
        unsigned int f4:1;
        unsigned int f3:1;
        unsigned int f2:1;
        unsigned int f1:1;
        unsigned int f0:1;
    #endif
      } field;
      unsigned char value;
    } decoded_byte_t;
```

This data structure is a ```union``` of a ```char``` and a
```struct``` which has eight one bit fields. The ```char``` and the
```struct``` are the same size and the union makes **field** and
**value** "aliases" for the same byte-sized chunk of memory. The names
of the fields are reversed depending on whether the current platform
is [big-endian][7] or [little-endian][8], using values included from
```/usr/include/ctype.h```.

This complex looking data structure makes it simple to access each bit
in the byte by field name, regardless of endian-ness, and then access
the assembled value via the ```value``` field of the union. We depend
on the compiler to generate the correct bit-shifting instructions to
access the fields, which can save you a lot heartburn when you are
debugging.

Lastly, ```stupid_decode()``` is *stupid* because it only reads eight
bytes at time from the source FILE stream. Normally, we try to
minimize the number of reads to improve performance. I won't try to
explain the cost of system calls right now, just remember reading or
writing a bigger chunk less often is better than a lot of smaller chunks
more frequently.

### The Wrap Up

Writing a multi-file program in C requires a little more planning on
behalf of the programmer than just a single ```main.c```. But just a
little effort up front can save you a lot of refactoring down the
road as you add functionality.

I like to have a lot of files with a few short functions in them. I
like to expose a small subset of the functions in those files via
header files. I like to keep my constants in header files, both
numeric and string constants. I **love** Makefiles and use them
instead of ```bash``` scripts to automate all sorts of things.

I know I've only touched the surface of what's going on in this simple
program and I'm excited to learn what things were helpful to you and
what things need better explanations. Drop me a line in the comments to
let me know.


[1]: https://opensource.com/article/19/5/how-write-good-c-main-function
[2]: https://FIXME/link_to_posix_unix_def?
[3]: http://www.jabberwocky.com/software/moomooencode.html
[4]: https://FIXME/link_to_nyan_cat_gif
[5]: https://github.com/JnyJny/meowmeow.git
[6]: https:///FIXME/make_documentation
[7]: https:///FIXME/wikipedia/big-endian
[8]: https:///FIXME/wikipedia/little-endian
[main_c]: https:///FIXME/link/main.c
[main_h]: https:///FIXME/link/main.h
[mmencoder_h]: https:///FIXME/link/mmencoder.h
[mmencoder_c]: https:///FIXME/link/mmencoder.c
[mmdecoder_h]: https:///FIXME/link/mmdecoder.h
[mmdecoder_c]: https:///FIXME/link/mmdecoder.c
[makefile]: https:///FIXME/link/Makefile

