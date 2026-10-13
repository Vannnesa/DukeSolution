# About `squares_test.o`

The assignment README says:

> We have provided squares_test.o, which has a main function ... You can compile
> your code and link it with this object file

The file the course shipped was a Linux x86-64 object, which cannot be linked
on macOS or Windows. Instead of shipping a binary that only works on one
platform, this folder carries the driver as **`squares_test.c`**, and
`squares_test.o` is built from it for your machine the first time you run
`grade` in this folder.

That means the command in the README works as written:

    gcc -o squares -Wall -Werror -std=gnu99 --pedantic squares.c squares_test.o

If `squares_test.o` is ever missing — because you deleted it, or because you
copied this folder somewhere else — running `grade` rebuilds it.
