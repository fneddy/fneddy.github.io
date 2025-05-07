# TOC
<!-- toc -->
# TLDR
Use the GCC `__attribute__((section("...")))` to place (*from many different places in your code*) data into a global `array`. Final code is [here](#final-code).

# The Attribute Section Story

Some time ago, I encountered a tricky problem in C. I needed to push data into an array from all over my code. Not just when the code flow eventually reached a location, but instantly at program start.

I was sure there must be a solution—and there is!

**`__attribute__((section("...")))` to the rescue!!!**

**`__attribute__(...)`**: is a way in GCC and Clang to tell the compiler specifics about how to handle data or functions. There is a great list of all possible attributes [here](https://gcc.gnu.org/onlinedocs/gcc/Attributes.html).

**`section("...")`**: refers to `elf section`. When you compile your code into an executable, the resulting binary is packed into a binary format - namely `elf` on a linux system. This `elf` file will have some predefined sections like `.text` `.bss` [and so on](https://refspecs.linuxfoundation.org/LSB_5.0.0/LSB-Core-generic/LSB-Core-generic/specialsections.html). But you are not limited to this sections. GCC allows you to create your own sections for your *special* data.

For my case I wanted to collect many `struct MyMetaData {...}` instances in a `MetaDataSection` section. Then, at program start, this data would be somehow processed. Later when the program arrives at the place *where* this struct is actually created, it would simply ignore it. 

All this collecting data into a section can be easily done by adding `__attribute__((section("MyMetaDataSection")))` to every variable declaration.

<div class="warning">
<b>However: To make working with sections somewhat feasible, we need to take some care and precaution</b>
</div>

> - Only place data of the same type in a section.
> - Tell the compiler to layout the data like an array.
> - Ensure the compiler will not strip out our data.
> - Avoid name collisions.



# Placing Data in a Section

Placing (pushing) data into an elf section is done by adding the `__attribute__((section(...)))` to the variable definition. 

```C
/* place the variable FOO in MySection */
int  __attribute__((section("MySection"))) FOO = 4;
```

Conveniently, the linker will automatically create two symbols for us: `__start_MySection` and `__stop_MySection`. This symbols are placed at the begin and end of section. 
<div class="warning">
NOTE: this symbols are **NOT** pointers to the start/stop,themselves are located at the positions!! Since the symbols are created by the linker we do not see them during compilation.
So we need to declare them ourselves:
</div>

```C
/* place the variable FOO in MySection */
int  __attribute__((section("MySection"))) FOO = 4;

/* declare the __start and __stop markers of our new section.
 * remember to make them extern otherwise we will create two variables instead. */
extern int __start_MySection;
extern int __start_MySection;
```

To access the data in the section, simply look at all the data in between the `__start...` and `__stop...` marker.

```C
#include <stdint.h>
#include <stdio.h>

/* place the variable FOO in MySection */
int  __attribute__((section("MySection"))) FOO = 0x4;
int  __attribute__((section("MySection"))) BAR = 0x8;
int  __attribute__((section("MySection"))) BAZ = 0x10;

/* declare the __start and __stop markers of our new section.
 * remember to make them extern otherwise we will create two variables instead. */
extern int __start_MySection;
extern int __stop_MySection;

int main() {
	/* create a pointer -> to the first element in the section */
	int *i = &__start_MySection;

	/* go through every element in the section */
	for (;i < &__stop_MySection; i++) {
		printf("i:%d\n",*i);
	}

	/* only global data can be put in a section. 
	 * static will put a variable lifetime in global scope
	 */
	static int  __attribute__((section("MySection"))) FAZ = 0x20;

	return 0;
}
```
Compile and run it and voila:

```ignore
$ ./a.out 
 i:4
 i:8
 i:16
 i:32
```

# Setting the Data Layout

Now that we can collect our data from all over the code, let's change the example to use `structs`.

<div class="warning">
NOTE: bad example, do not use the following code!
</div>

```c
#include <stdint.h>
#include <stdio.h>

struct MetaData {
	char name[32];
	int id;
};

/* place the variable STRUCT_FOO in MySection */
struct MetaData STRUCT_FOO __attribute__((section("MyStructSection"))) = {
	.name = "FOO",
	.id = 1
};

struct MetaData STRUCT_BAR __attribute__((section("MyStructSection"))) = {
	.name = "BAR",
	.id = 2
};

struct MetaData STRUCT_BAZ __attribute__((section("MyStructSection"))) = {
	.name = "BAZ",
	.id = 3
};


/* declare the __start and __stop markers of our new section.
 * remember to make them extern otherwise we will create two variables instead. */
extern struct MetaData __start_MyStructSection;
extern struct MetaData __stop_MyStructSection;

int main() {
	/* create a pointer -> to the first element in the section */
	struct MetaData *i = &__start_MyStructSection;

	/* go through every element in the section */
	for (;i < &__stop_MyStructSection; i++) {
		printf("0x%x name:%s id:%d\n",i, i->name, i->id);
	}

	/* only global data can be put in a section. 
	* static will put a variable lifetime in global scope
	*/
	static struct MetaData STRUCT_FAZ __attribute__((section("MyStructSection"))) = {
		.name = "FAZ",
		.id = 4
	};

	return 0;
}
```

Compile and run it and **BAM**:

```ignore
 $ ./a.out 
0x403020 name:FOO id:1
0x403044 name: id:0
0x403068 name: id:0
0x40308c name: id:0
0x4030b0 name: id:0
0x4030d4 name: id:0
0x4030f8 name: id:0
```

Ok, clearly something went wrong. So lets do some **debugging**

# Some Debugging

First, lets check the size of our struct:
```
$ gdb ./a.out -ex="p sizeof(struct MetaData)" -ex="q"
...
$1 = 36
```
Looking at the program output, we see that the loop indeed steps `36(0x24) bytes`. It seems the `__start...` and `__stop..` symbol are farer apart as we would expect?! 

Hex-dumping[^HEXDUMP] the complete section will reveal the mystery:

[^HEXDUMP]: objdump allows us to specify only a specific section to hexdump. Same could have been done with any other hexdump or re tool. 

```ignore
$ objdump -s -j MyStructSection  a.out 

a.out:     file format elf64-x86-64

Contents of section MyStructSection:
 403020 464f4f00 00000000 00000000 00000000  FOO.............
 403030 00000000 00000000 00000000 00000000  ................
 403040 01000000 00000000 00000000 00000000  ................
 403050 00000000 00000000 00000000 00000000  ................
 403060 42415200 00000000 00000000 00000000  BAR.............
 403070 00000000 00000000 00000000 00000000  ................
 403080 02000000 00000000 00000000 00000000  ................
 403090 00000000 00000000 00000000 00000000  ................
 4030a0 42415a00 00000000 00000000 00000000  BAZ.............
 4030b0 00000000 00000000 00000000 00000000  ................
 4030c0 03000000 00000000 00000000 00000000  ................
 4030d0 00000000 00000000 00000000 00000000  ................
 4030e0 46415a00 00000000 00000000 00000000  FAZ.............
 4030f0 00000000 00000000 00000000 00000000  ................
 403100 04000000                             ....            
```

All our structs are actually `64(0x40) bytes` apart! This is due to optimizations GCC does to `global` variables. 

But we want our section to be layout like in an array. And that we can iterate with a pointer over all entries. This can be done by forcing the compiler to align all or structs, as it would align them in an array. The `alignof` compiler builtin will tell us what the default (and also Array) alignment of a type is. And with the `__attribute__((aligned(...)))` we set the alignment of or structs to that specific value: `__attribute__((aligned(alignof(struct MetaData))))`

# Final Code

Adding this second attribute will definitely make the code unreadable. so lets create a macro for that.

And while we are at it lets also add the third and fourth requirement. 
- Prevent the compiler from striping the data by marking it as being used with `__attribute((used))` 
- Hide the variable names to prevent the names from cluttering the global namespace[^namespace] `__attribute((visibility ("hidden")))`.

[^namespace]: namespace in the sense of a c link unit, not a C++ namespace.

Luckily we can combine all attributes: 

```c
#include <stdint.h>
#include <stdio.h>

#define META_DATA_SECTION __attribute__(( \
	/* place the data into META_DATA_SECTION */ \
	section("META_DATA_SECTION"), \
	/* do not insert empty space between elements */ \
	aligned(alignof(struct MetaData)), \
	/* mark this data as used so the compiler will not remove it */ \
	used, \
	/* optionally hide the variable name from other parts of the code */ \
	visibility ("hidden") \
)) 

/* our actual payload data */
struct MetaData {
  char name[32];
  int id;
};


/* now create a MetaData point and push it to META_DATA_SECTION */
struct MetaData _MY_DATA_1 META_DATA_SECTION = {
	.name = "foo", 
	.id = 0xc0ffe
};
struct MetaData _MY_DATA_2 META_DATA_SECTION = {
	.name = "bar", 
	.id = 0xdead
};

/* we can also use this in a function */
void function_a() {
	/* only global or static variables can be pushed to a section. */
	static struct MetaData _MY_DATA_XX META_DATA_SECTION = {
		.name = __FILE__, 
		.id = 0xc0de
	};
}

/**
 * to access the data we need this external symbols the linker
 * will automatically create symbols for a section called __start_SECTION_NAME 
 * and __stop_SECTION_NAME however to satisfy the compiler we still need to 
 * declare the symbols as external here
 **/
extern struct MetaData __start_META_DATA_SECTION;
extern struct MetaData __stop_META_DATA_SECTION;

int main() {
	/**
	 * the __start symbol will be located (and not pointing!!) at the start of
	 * the section. Therefor we take its &address here which will point to the
	 * first element in the section.
	 **/
	struct MetaData* iter = &__start_META_DATA_SECTION;

	/**
	 * now we iterate over all elements until we reach the end of the section
	 **/
	for (; iter < &__stop_META_DATA_SECTION; iter++) {
		printf("0x%x {name:%s,id:0x%x}\n",iter,  iter->name, iter->id);
	}

}


struct MetaData _MY_DATA_3 META_DATA_SECTION = {
	.name = "baz", 
	.id = 0xc0de
};

```
we run it and **YES** it works!

```ignore
$ ./a.out 
0x40300c {name:foo,id:0xc0ffe}
0x403030 {name:bar,id:0xdead}
0x403054 {name:sections.c,id:0xc0de}
0x403078 {name:baz,id:0xc0de}
```