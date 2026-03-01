# HL86 - Pseudo-language for Reverse Engineering

A VSCode extension that provides syntax highlighting for my self-defined pseudo-language, along with snippets, both intended to speed up the reverse engineering process. Somewhat ironically, to ensure a consistent implementation of this pseudo-language, a specification of its intended use is provided below:

## Specification

#### Literals

There are `3` supported literal types: `bool`, `hex`, and `string`. There is *no* support for other numeric bases, as they were never required.

```c
true | false
1020304F
"string"
```

#### Containers

Unlike most languages, this language does not use variables in the traditional sense. Instead, it relies on predefined containers that represent common storage systems in assembly. These include `registers`, `arguments`, `locals` and `constants`.

```c
arg 10 = eax
var 1C = arg 10

// for repetitively used statics (should be defined before any function)
$MyExecutable!ConstantValue2 = "Hello world!", 0
```

#### Control

To indicate program control flow, a minimal set of keywords is provided to avoid overthinking functionality: `if`, `else`, `goto`, and `return` (`functions` are described later).

Control assistance truly shines in the snippet subsystem, which converts assembly branches into their HL86 equivalents. Below is a simple example of such a conversion:

```c
> je

if (a == b):
    <end-position cursor>
else:
    ... idk // more on descriptors later
```

This would be the snippet you choose if you **want to take the branch** to reach the desired code block during your reverse engineering session. If, however, you **do not want to take the branch**, you can use the following snippet:

```c
> je! // notice the negation '!'

if (a == b):
    ... irr
else:
    <end-position cursor>
```

There is one more control-flow keyword: `goto`. It will be applied properly in the _Functions_ section.

#### Descriptors

To describe certain segments of code, this language pack includes a set of keywords for that exact purpose. There are `5` descriptors: `entry`, `desc`, `vuln`, `irr`, and `idk`. An example of their usage is shown below:

```c
entry // our breakpoint activates here

if (eax == -1):
    // briefly describe what this block does (not comments)
    ... desc "Socket connection failed"

else if (eax == ebx):
    // we do not know what this part does (yet)
    ... idk

else:

    if (arg 1C != ebx):
        // we do not want to reach this part of the code
        ... irr // means irrelevant

    else:
        // we know this part is vulnerable
        eax = vuln std!memcpy(dest: edi, src: esi, count: ecx)
```

Sometimes you want to label a generic data block with a name, so that you can distinct it during runtime. It however, doesn't make sense to assign these items to a variable because their identity change drastically throughout the execution (e.g. memory address changes). To counter this the `~*` descriptor has been added. With this you can "mark" a piece of data that you _know_ has a specific label:

```c
var 8 = wsock32!recv(s: irr, 
    buf: ~buffer, len: 4400, flags: irr);

// ... many blocks later (different function)
std!memcpy(dst: idk, src: ~buffer, count: 4) // they're the same to us but in-code they differ
```

#### Functions

These are arguably the most uncertain structures to define, likely because you rarely understand them completely during reverse engineering. To provide some form of syntax, functions are assumed to be incomplete by default. Function blocks are annotated using offsets relative to the function entry. Both function declarations as well as calls follow the consistent naming convention `<namespace>!<function>(<args>)`:

```c
MyProgram!SomeFunction()
{
<<< +4D: // must be hexadecimal

    if (arg 4 == 0):
        ... idk

    else if (arg 4 == 1):
        goto irr MyProgram!Exit->80C

    else:
        goto this->512A

    ... irr

<<< +512A:

    eax = MyProgram!AnotherFunction()
    ebx = eax & (ecx + 5)

    if (ebx):
        return true
    else:
        return false
}
```

## Snippets

Snippets are a big part of this language pack, they aim to speed-up the reversing process through logical systems in addition to large volumes of constants. Below are some of the non-conditional snippets where the placeholders represent the cursor its position depending on the amount of types you've pressed `<tab>`:

```c
// functions
> fn
<3>!<4>()
{
<<< +<1>:
    
    <2>
}

> fnc
eax = <1>!<2>(<3>)

// jumps (this and external)
> jmp
goto this-><1>

> jmpx
goto <1>-><2>

// standard libraries
> std!memcpy
std!memcpy(dest: <1>, src: <2>, count: <3>)

> std!malloc
std!malloc(size: <1>)
```