### Introduction to GNU `make` utility
`make` is the utility designed to compile chain of files using simple make commands, instead of compiling each file individually.

*Makefile consists of "rules" that has a definite structure.*

```
target: prerequisites
 <TAB>recipe
```

**Target** - A target is usually the name of executables or object files. It can also be the name of an action such as clean.  
**Prerequisites** - A prerequisite is a file name that is used as an input for target. Target could depend of multiple files which are separated by space. Some rule may not require prerequistes.
**Recipe** - A reciple is a command(s) that make executes. It may have more than one command either in the same line or each on its line.

> Note: A rule is preceded by a <TAB> character.

* Starting the recipe with @ will not print the command on terminal.
* Variables are in ALL_CAPS followed by a = sign. Its best practice to place all variables at the top of the file.
* To use variable later in the file use $(VARIABLE_NAME) or ${VARIABLE NAME}. The parens/brackets are mandatory.

Example 01:
```
NAME=ashwin

test:
    @echo $(NAME) # due to @; command won't be printed on terminal.
    echo $(CXX)
```

Example 02:
compile a program from 2 files and one shared header

```
CFLAGS=-g

all: program

source1.o: source1.c
    $(CXX) $(CFLAGS) -c source1.c

source2.o: source2.c
    $(CXX) $(CFLAGS) -c source2.c

program: source1.o source2.o
    $(CXX) $(CFLAGS) -o program source1.o source2.o
```

#### Automatic variables

* `$@` - Name of the target.
* `$^` - All dependencies/prerequisites

Example 03:

```
program: source1.o source2.o
    $(CXX) $(CFLAGS) -o program source1.o source2.o
```
Can be written as:
```
program: source1.o source2.o
    $(CXX) $(CFLAGS) -o $@ $^
```

#### Makefile is not only for compiling programs and generating object files, it is used to write rules to make a directory `clean` by deleting the generated object files and executables.

```
clean:
    rm edit $(objects)
```

In practice a clean rule is written with conditional instructions to handle unanticipated situations.

```
.PHONY: clean

clean:
    -rm edit $(objects)
```

Writing the `clean` make target in the `.PHONY` list keeps make from confusing it as a file name. As it is possible that their exist a file named *clean*.

#### .PHONY
Here we explicitely tell `make` that the listed targets are not files, treat them as targets. 
So for:
```
.PHONY: clean

clean:
    rm $(objects)
```

Even if a file named *clean* exists in the present directory still make will execute the `clean` target.

### Recompilation

Make actively tracks all the source files, for example the project have 3 source files and 1 header file then;
First compilation:
```
make all
```
```
editor.c    -> editor.o 
buffer.c    -> buffer.o
helper.c    -> helper.o     editor

header.h    -> header
```

Now lets say I edited the buffer.c source file only, and recompiled it;
```
make all
```

```
editor.c    -> editor.o - same 
buffer.c    -> buffer.o - recompiled
helper.c    -> helper.o - same          editor

header.h    -> header   - same
```


`make` actively tracks all the source files, thus it will only recompile `buffer.c` source file, as its the only file changed, others will be untouched.

#### But what will happen when a header.h file is changed/edited
In that case all the source files which uses/imports that header file will be recompiled along with the header file.

**make tracks dependencies, not just direct source-file changes.**


So make effectively does:

1. Check source/header dependencies
2. Recompile whatever needs recompilation
3. If object files changed → relink executable
4. Otherwise → don't relink


### Makefile naming

By default `make` command looks for Makefile named `Makefile` or `makefile`, but we can tell explicitely name of our makefile with `make -f` or `make --file` flag.

```
make -f mymakefile

OR

make --file=mymakefile
```

### Override makefile variables from command line while invoking

Example:
```
CC = gcc

program:
    $(CC) main.c -o $@
```

Here if we want to change the compiler while running makefile, so instead of editing the makefile we can override it from the command line.

```
make CC=clang
```

So ``CC`` will be set to ``clang``.

#### Real example
Based on use case set `DEBUG` variable to True from command line;
```
make DEBUG=1
```

```
ifeq ($(DEBUG),1)
    ...
endif
```

### 3 exit status

When we run `make` the shell receives exit status from make. There are 3 important possiblities of these exit status.

**0 - Success**
Everything worked
```
make
echo $?
```
produces
```
0
```

**1 - special case with -q**
This is special case with `-q` flag, in this case we tell `make` to not build anything, but just tell me if rebuilding is needed or not.
```
make -q
```
If everything is same: (no rebuilding is needed)
```
exit status = 0
```
If something needs rebuilding:
```
exit status = 1
```

So you can use it in scripts:
```
make -q
if [ $? -eq 1 ]; then
    echo "Something needs rebuilding"
fi
```
*The important thing is that 1 doesn't normally mean "ordinary build failed." It has this special meaning for -q.*

**2 - error**
Something actually went wrong  
For example:
```
make
```
and compilation fails:
```
gcc: error: ...
make: *** [Makefile:10: program] Error 1
```
Make exits with:
```
2
```

#### = vs :=

**=** is lazy assignment
```
A=$(B)
B=hello
```

Here with `=` operator, when control reaches to this line, it does not try to evaluate `$(B)` immediately, but it just knows that whatever is or will be the value of `B` is the value `A` will have.

So control will reach to next line:
```
B=hello
```
And will store *hello* into variable `B`, and when `A` will be used it will also have the same value.

**:=** immediate assignment  
```
A:=$(B)
B:=hello
```
With operator `:=` we are telling `make` to evaluate the value of this variable RN.
Here when control reaches to first line, it does not know whats the value of variable `B`, but `:=` operator forces `make` to resolve it at that moment only, thus it will evaluate to something, but not value of `B`.


- `=` - Define variable
- `:=` - Calculate the value and assign immediately


#### $(shell ...)
```
PLATFORM := $(shell ./what-platform)
```

`make` runs the provided file/script using shell and substitutes with its output into the Makefile expression.  

#### $(shell ...) + sed
```
DIST := $(shell ./what-platform | sed \
        -e `s/rhel\|centos/el' \
        -e 's/sles\|leap/suse/' \
        -e 's/\..*//')
```
This is Make executing a normal Unix pipeline:
```
./what-platform
       ↓
      sed
       ↓
 transformed result
```
Suppose:
```
./what-platform
```
returns:
```
rhel9.4
```
Then:
```
sed -e 's/rhel\|centos/el/' -e 's/\..*//'
```
turns it approximately into:
```
el9
```