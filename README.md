# TIP 672 Prototype: $(expr) Syntax for Tcl

This repository contains a working prototype implementation of [TIP 672](https://core.tcl-lang.org/tips/doc/trunk/tip/672.md), which adds the `$(expression)` syntax to Tcl as a more intuitive alternative to `[expr {expression}]`.

## Overview

The `$(...)` syntax provides a cleaner, more modern, and more secure way to evaluate expressions in Tcl:
```tcl
# Traditional syntax
set result [expr {$a + $b * $c}]
puts "Total: [expr {$sum + $tax}]"

# New $(expr) syntax
set result $($a + $b * $c)
puts "Total: $($sum + $tax)"
```

## Key Features

✅ **Full bytecode compilation** - Generates identical optimized bytecode as `[expr {...}]`  
✅ **String interpolation** - Works seamlessly inside quoted strings  
✅ **Proper error handling** - Clear error messages without crashes  
✅ **Variable scoping** - Respects Tcl's normal scoping rules  
✅ **Interactive mode** - Handles incomplete input correctly  
✅ **More Secure** - Automatically encloses expressions inside braces 

## Examples

### Basic Usage
```tcl
% set a 25
25
% set b 50
50
% set c $($a + $b)
75
```

### String Interpolation
```tcl
% puts "The sum of $a and $b is $($a + $b)"
The sum of 25 and 50 is 75
```

### Inside Procedures
```tcl
% proc calculate {x y} {
    return $($x * $y + 10)
}
% calculate 5 3
25
```
## Bytecode Compilation

The `$(...)` syntax produces identical, optimized bytecode:
```tcl
% proc foo arg {return $($arg + 10)}
% tcl::unsupported::disassemble proc foo
ByteCode 0x..., refCt 1, epoch 17, interp 0x... (epoch 17)
  Source "return $($arg+10)"
  Cmds 2, src 16, inst 6, litObjs 1, aux 0, stkDepth 2, code/src 0.00
  Proc 0x..., refCt 1, args 1, compiled locals 1
      slot 0, scalar, arg, "arg"
  Commands 2:
      1: pc 0-5, src 0-15
      2: pc 0-4, src ...
  Command 1: "return $($arg+10)"
  Command 2: "expr {$arg+10}..."
    (0) loadScalar1 %v0     # var "arg"
    (2) push1 0             # "10"
    (4) add
    (5) done
```

The compiler recognizes the expression pattern and generates direct arithmetic operations with no runtime parsing overhead. Because the expressions are passed on to expr inside of braces, as can be seen above in Command 2,  there is also much less chance of security problems. If a braced expr won't accept it, neither will $(...).




### Nested example and bytecode
```tcl
% set a 1; set b 2; set c $( $a + $b + [string length [string range abcdefghijk 1 end-$($a+3)]])
9
# note, there's no reason to use a nested $(...) as for example, $( $a * $($b+$c) )
# since  $( $a * ($b+$c) ) will be accepted, and would be more efficient anyway.
# If one is entered, it (actually expr) will throw an error for an invalid $.

% tcl::unsupported::disassemble script {set a 1; set b 2; set c $( $a + $b + [string length [string range abcdefghijk 1 end-$($a+3)]])}
ByteCode 0x13d1307ce70, refCt 1, epoch 22, interp 0x13d0d6c1990 (epoch 22)
  Source "set a 1; set b 2; set c $( $a + $b + [string length [st..."
  Cmds 7, src 94, inst 76, litObjs 8, aux 0, stkDepth 7, code/src 0.00
  Commands 7:
      1: pc 0-11, src 0-6        2: pc 12-23, src 9-15
      3: pc 24-74, src 18-93        4: pc 29-73, src 105969-106042
      5: pc 42-72, src 105987-106040        6: pc 42-71, src 106002-106039
      7: pc 57-68, src -65567--65557
  Command 1: "set a 1..."
    (0) push 0 	# "a"
    (5) push 1 	# "1"
    (10) storeStk 
    (11) pop 
  Command 2: "set b 2..."
    (12) push 2 	# "b"
    (17) push 3 	# "2"
    (22) storeStk 
    (23) pop 
  Command 3: "set c $( $a + $b + [string length [string range abcdefg..."
    (24) push 4 	# "c"
  Command 4: "expr { $a + $b + [string length [string range abcdefghi..."
    (29) push 0 	# "a"
    (34) loadStk 
    (35) push 2 	# "b"
    (40) loadStk 
    (41) add 
  Command 5: "string length [string range abcdefghijk 1 end-$($a+3)]..."
  Command 6: "string range abcdefghijk 1 end-$($a+3)..."
    (42) push 5 	# "abcdefghijk"
    (47) push 1 	# "1"
    (52) push 6 	# "end-"
  Command 7: "expr {$a+3}..."
    (57) push 0 	# "a"
    (62) loadStk 
    (63) push 7 	# "3"
    (68) add 
    (69) strcat 2 
    (71) strrange 
    (72) strlen 
    (73) add 
    (74) storeStk 
    (75) done 

```

Note that in this example, there **is** a nested $(...) but it is inside of a [...] command substitution. Also note that it efficiently processes the end- which it concatonates to the single numerical result needed for a valid index expression. It is, of course, just the same as doing end-[expr {...}] which is pefectly valid tcl syntax.

The best way of thinking about the $(...) construct is that it is like a macro expansion except there is no preprocessor phase like in the C compiler #define macros.

### Complex Multi-line Expressions
```tcl
# Natural multi-line formatting without backslashes (but can be added if desired)
set total $(
    $price * $quantity +
    ( $shipping - $discount )
)
```

## Implementation Details

### Files Modified

Only two files in the Tcl core were modified:

1. **tclParse.c** - Parser modifications (2 functions)
2. **tclNamesp.c** - Error reporting fix (1 function)

### Changes Made

#### 1. `Tcl_ParseVarName` - Main Implementation (~70 lines)

Converted the existing two-way branch to a three-way branch:
```c
if (*src == '{') {
    // Handle ${varname} or ${array(index)}
} else if (*src == '(') {
    // NEW: Handle $(expression)
} else {
    // Handle $varname or $varname(index)
}
```

The new `else if` block:
- Finds matching closing parenthesis
- Creates synthetic string `[expr {expression}]`
- Parses synthetic string for validation
- Generates `TCL_TOKEN_COMMAND` token
- Returns successfully

#### 2. `ParseTokens` - Size Adjustment (~13 lines)

After calling `Tcl_ParseVarName` for `$(...)`, adjusts for the size difference between the original source text and the synthetic string to ensure correct source position tracking.

#### 3. `TclLogCommandInfo` - Crash Prevention

Added null-byte check to prevent crashes when error reporting encounters synthetic strings:
```c
        // Only scan if command appears to be within script's memory region
        if (command >= script) {
            for (p = script; p != command && *p != '\0'; p++) {
                if (*p == '\n') {
                    iPtr->errorLine++;
                }
            }
        }
        // If command < script, something is wrong - just use line 1

```

#### 4. & 5. `numBytes` Boundary Checks

Two defensive checks changed from `==` to `>` and `<=` comparisons to handle edge cases:
- `while (numBytes > 0 && ...)` instead of `while (numBytes && ...)`
- `if (numBytes <= 0 || ...)` instead of `if (numBytes == 0 || ...)`

## Known Limitations

### 1. Memory Leak (Requires Core Team Input)

The synthetic string `[expr {expression}]` is allocated but not tracked for cleanup in `Tcl_FreeParse`. Potential solutions:

- Add a cleanup tracking field to `Tcl_Parse` structure (acceptable for Tcl 9.x)
- Implement thread-local side table for synthetic string management

Note that this memory leak only occurs if the source code using a '$(...)' is redefined. Once code is compiled, there is no additional memory and the synthetic string might be used over and over, for example, if there are many thrown errors such as undefined variables in the expression. 

Provided the '$(...)' use is not in a loop of eval's or in procedures that are redefined constantly, such as in a loop doing source statements, there will be no additional memory used. For each use of '$(...)' there will be an allocation of the size of the ... expression + ~10 bytes. Calling a procedure (or method) over and over that has a '$(...)' expression will not allocate any additional memory per call since the bytecode won't be changing, so no reparse or recompile is performed. 

Eventually, this might be handled by using the bytecode epoch to know when bytecode is invalidated. If one really must use eval or source in a loop, then one should revert to using expr instead.

### 2. Error Messages Show Synthetic Command

When an expression evaluation error occurs, the error message shows the synthetic command rather than the original:
```tcl
% set d $($a + $b)
can't read "a": no such variable
    while executing
"expr {$a + $b}"
```

This is actually helpful as it shows what's being executed, but differs from the user's input.

### 3. Empty Array Name Conflict

The construct `set x $($varname)` creates an ambiguity—it could mean either:
- Reading an empty array element with index from `$varname`, or
- Evaluating the expression containing the value of `$varname`

The `$(...)` syntax takes precedence, so empty array element access with a variable index is no longer possible using this pattern.

**Setting** empty array elements still works:
```tcl
set (index) value      # Still works - sets empty array element
set ($varname) value   # Still works - variable index
```

**Reading** empty array elements with variable indices requires an alternative:
```tcl
# Old way (no longer works):
set x $($varname)      # Now evaluates as expression

# Alternatives that work:
set x ${($varname)}    # Braced form
set x [set ($varname)] # Command substitution
```

This is an extremely rare edge case—empty array names are almost never used in practice.
###  Parenthesis and Brace Balancing in Expressions

Parentheses within `$(...)` must be balanced, but only if they are outside of (possibly nested) command `[...[...]...]` substitutions. That is just par for parenthesis in everyday expression usage. However, inside of (possibly nested) brackets, there can be quoted or unquoted strings and these may have unbalanced parenthesis.

Braces, on the other hand, will need to be balanced exactly the same as they do when using `[expr {...}]` since `$(...)` is identical. This means that just as `[expr {[string length "abc\{def"] + 1}]` will need an escape for unbalanced braces, so it is with `$(...)`. This is just the way Tcl works. So, in this case, one would also need to escape the unbalanced brace as `$([string length "abc\{def"] + 1)`.

Unfortunately, at present, expressions that do relational compares against literal quoted strings, and not inside of [commands] will stil need parens to be escaped, for example `set x $("1\)\}" eq "1\)\}")` but this may be fixed although it will require some tricky bookkeeping. Braces in these litteral strings have always needed to be escaped in expr and so also here too.

**Example not requiring escape for parens, but needing one for a brace:**
```tcl
# Unbalanced paren in string literal - need not escape, but to do so causes no change
% set result $([string length "abc(def"] + 1)
8
% set result $([string length "abc\(def"] + 1)
8

# note, same behavior as expr

% expr {[string length "abc\(def"] + 1}
8
% expr {[string length "abc(def"] + 1}
8

% puts $([string length "abc\{def"] + 1)
8
% puts [expr {[string length "abc\{def"] + 1}]
8
```
## No support for Comments

This $(expr) shorthand is for short expressions therefore it does not support comments. If these are needed, one should use the traditional expr command.

## Precedent

[Jim Tcl](http://jim.tcl.tk/) has successfully used the `$(...)` syntax for years demonstrating real-world viability.



## Testing

The prototype has been tested with:
- ✅ Basic arithmetic expressions
- ✅ String interpolation
- ✅ Nested expressions
- ✅ Variable scoping (local and global)
- ✅ Error conditions (undefined variables, syntax errors)
- ✅ Interactive mode (incomplete input handling)
- ✅ Multi-command lines

## Viewing the Changes

To see exactly what was modified from the original Tcl source to implement the $(expr) syntax:

**[View complete implementation diff](https://github.com/rocketship88/tcl-tip-672-prototype/commit/7dd22b8cf76a82c65b6c5751b1de2b7058477365)**

This shows all changes made to `tclParse.c` and `tclNamesp.c`. in the first commit. Additional refinements and improvements are visible in subsequent commits. As of this time, there is one change to tclParse.c as well. Clicking on that commit, one can see what changed there.

You can also:
1. Click on the "Commits" link above
2. Select the commit "Implement TIP 672: $(expr) syntax for expression evaluation"
3. Use the settings icon (top right of diff view) to switch between unified and split views

## Questions for Tcl Core Team

1. What is the preferred approach for tracking synthetic strings for cleanup?
2. Is this prototype sufficient for consideration in Tcl 9.x?
3. Are there architectural concerns with the synthetic string approach?
4. Should error messages be adjusted to show the original `$(...)` syntax instead of the synthetic command?

## Repository Structure

- `tclParse.c` - Modified parser with `$(expr)` support
- `tclNamesp.c` - Modified error reporting to prevent crashes
- Commit history shows original → modified versions for easy comparison

## References

- [TIP 672](https://core.tcl-lang.org/tips/doc/trunk/tip/672.md) - Original proposal
- [Jim Tcl Documentation](https://jim.tcl-lang.org/home/doc/www/www/documentation/) - Existing implementation

## License

This code is derived from Tcl source code. See the [Tcl License](https://www.tcl.tk/software/tcltk/license.html) for terms.

---

**Author:** E. Taylor
**Date:** October 17, 2025  
**Tcl Version:** Based on Tcl 9.x development source
