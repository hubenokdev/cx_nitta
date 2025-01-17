# Compiler Development

Goyacc is a version of yacc generating Go programs.

<br />

Installation
<br />
$ go get modernc.org/goyacc
<br />
Documentation: godoc.org/modernc.org/goyacc
<br />

When submitting changes to CX core, the development team uses Git pull requests for staging. Submit changes as a pull request and after review by the development team additional changes may be suggested prior to merging.

# CX Lexer
CX in house `lexer` generates a chain of tokens which is used by yacc for parsing.
(found in)[lex.go] (https://github.com/skycoin/cx/cxparser/cxpartialparsing/lex.go )



# CX Compiler Stages

`Preliminarystage`

The compiler creates an AST first by using regular expression parsing (found in [utils.go](https://github.com/skycoin/cx/blob/develop/cxparser/cxparsing/utils.go#L21) which structures the AST to include all package declarations, import chains, struct declarations, and globals, skipping over comments. 
This preliminary stage of parsing aids further stages since the structure of a CX repository and the names of custom types are already known. 

`Passone`

After this preliminary stage, the first parsing stage compiles function primitives and types, along with the structure of their parameters and global variables. This stage also finalizes compilation of structs and ensures packages and their imports are correct. This uses `goyacc` for parsing and creates a chain of tokens for the parser using an in-house lexer known as `Lexer`. 

`Passtwo`

The second parsing stage fully compiles functions and all expressions. This functions similarly to the first stage, using `Lexer` and `goyacc`, but also uses `cxparser/actions` for each action the parser should take after encountering a valid syntactical production rule. These work entirely to validate program structure and to build the AST for a CX program. 

Finally, the `main` function and invisible `*init` functions are created, the latter of which acts as an initializer for global variables. 

The lifetime of a CX program in depth is given here:
1.Preliminarystage ](https://github.com/skycoin/cx/blob/develop/cxparser/cxparsing/utils.go#L21) - here, a copy of the source code is passed in as an array of strings, along with the file names. A series of regular expressions are compiled to check for various things. Finally, the source code is iterated over, and for each file, we iterate over a scanner of the file. Then, for each successful scan:
    1. We first check if we’re in a comment or not. If we are, we continue scanning and ignore the following steps.
    2. Next, we check if this current line contains a package declaration. If so, then we check if the program has added a package with that name. If not, then we add that package to the CXProgram (PRGM0, basically the main program). Remember, now that package is the current package.
    3. We do the same thing for structures.
2. We now start a second iteration over the source code, the same as before, and for each successful scan, after checking for comments like before:
    1. We check for import statements, and for each one, add the CXPackage to the Imports of the active CXPackage.
    2. We check for global variables, and for each one found, add it to Globals of the active CXPackage. Additionally, we now have an inBlock counter, and when positive, it means we are inside a block, and should not add any variable declarations to globals. Things like variable type, etc will be added in later steps.
3. We can now do the third step, which is the first call to the parser (cxgo0). This parses function declarations and builds global variables fully, including custom types (CXStructs) and packages.
4. As an intermediate step, after the program has been selected for and after the program has been checked for any compile or parsing errors, os arguments are added to the program. 
    1. The OS package builtin for CX is grabbed, and for all globals in the OS package not currently added to said package, they are declared, and DeclareGlobalInPackage is called on them, after getting rid of the first two arguments, presumed to be the call to CX and the name of the program to call.
5. Finally, the program is fully parsed by the cxgo/parser code.
6. After all of this, we check if a main package exists. If not, one is automatically generated by CX, and a blank main function too. We also create something called the `*init` function, which is an automatically generated entry-point function for CX, which initializes all the global variables with the values they need to be initialized with, since the CX compiler doesn’t automatically set the data segment with said values.
7. Finally, the program is executed by running RunCompiled. This function takes two arguments - nCalls and args - which correspond to the initial number of calls made and the OS arguments to parse into the entry point function (main). For our intents and purposes, nCalls is zero, and args is already verified.
8. The program is selected, and the heap is initialized to its starting size, and the random seed for random functions is initialized. Additionally, untilEnd (saying whether or not to run until end) is set to true, since nCalls is zero. Finally, prior to beginning, the main package is selected.
9. If CallStack’s first CXCall’s operator is nil, that means `*init` needs to be executed. (We can safely check this since CallStack is already guaranteed to be a non-zero length array of default value structures). `*init` is selected, and made into a call with the appropriate maker, with StackPointer set to `*init`’s Size. `*init` is first called, then, obviously, it loads all the expressions onto the callstack to initialize the global variables. After this finishes, the call state is fully reset.
10. The main function is selected, and made into a call, with the frame pointer being set and the stack pointer updated by the main function’s Size (which, remember, is the size of the inputs and outputs of the function). 
11. For each `os` argument, the argument is initialized onto the heap.
12. Finally, CXProgram.Run is called. While the program isn’t terminated, and while untilEnd isn’t set or while nCalls isn’t zero, and while the CallCounter isn’t ever greater than untilCall (the final call):
    1. A call is grabbed from the CallStack at CallCounter
    2. Check for stackoverflow
    3. While not untilEnd, each expression is executed and popped from the CallStack. (etc...)


# CX AST

The CX AST, or abstract syntax tree, is the intermediate representation of a CX program which the CX runtime executes. It consists of multiple flavors of nodes, each one broken down in the following subsections.

## CXArgument

The atomic unit of data in a program is the [CXArgument](https://github.com/skycoin/cx/blob/develop/cx/cxargument.go), which stores a symbol and a type, as well as metadata regarding said type and the relationship with other CXArguments. The object is composed of the following fields:
* Lengths - an array of integers which specify the dimensions of a multidimensional array (first element is the outermost array length, etc).
* DereferenceOperations - an array of integers which specify, in bytes, how much in program memory this CXArgument is offsetted from its base CXArgument. For example, for a struct field, this is the struct field’s offset in bytes. For an array element in an array of an array, this is the first array’s offset times the second array’s size times the second array’s element type size, plus the offset in the second array times the second array’s element type size.
* DeclarationSpecifiers - an array of integer constants which specify the type of the CXArgument. For type `[][]ui8`, this is `[]int{TYPE_SLICE, TYPE_SLICE, TYPE_UI8}`.
* Indexes - an array of CXArguments which correspond to how to dereference the current CXArgument. For example, for an array dereference with index “i”, i is also a CXArgument. So the array dereference CXArgument stores i in Indexes.
* Fields - an array of CXArguments which store, in order, the field dereferences of structs or nested structs. So this would be `struct_name.field_of_first_struct.field_of_this_struct`.
* Inputs, Outputs  - the input and output parameters of a CXArgument of type `TYPE_FUNC`. Usually used for function calls.
* Name - the symbol of the CXArgument.
* FileName - the file this symbol originates from.
* Type - the specific integer constant that reflects the type of this CXArgument (one of `TYPE_SLICE`, `TYPE_UI8`, etc.)
* Size - the size of the underlying basic type.
* Offset - the location in the program memory this CXArgument resides in.
* PassBy - an int constant representing how the variable is passed - pass by value, or pass by reference.
* FileLine - an int used for crash behavior, storing which file line this CXArgument originates from.
* StructType - a CXStruct representing the custom type this variable might be (if struct).
* Package - the CXPackage this CXArgument resides in.
* IsSlice, IsStruct - name says it all.
* IsShortDeclaration - is this CXArgument the result of a `CASSIGN` operation (`:=`)?
* IsInnerReference - is this CXArgument a reference to the field or element of another CXArgument? (`&array[0]` or `&struct.field`)
* PreviouslyDeclared - used by compiler to check if this variable has been declared yet or not, or if there are duplicate declarations.
* DoesEscape - should this variable be kept alive after the scope ends? (for example, a function returning a pointer to a variable created in the function should keep that variable alive after the scope ends, hence, this should be set to true then).

CXArguments, as you can see, are vital to a CX AST. CX has seen overwhelming change over the years, and as the 1.0 version is nearing completion, much of this may be subject to change.

CXArguments not only represent named variables such as globals, locals, functions in call expressions, and other things, but also represent temporary outputs of expressions. For example, a statement in CX such as `i = 2 * j + 3` requires storing the result of `2 * j` in a temporary variable on the stack. Hence, the operation will need a CXExpression with inputs 2 and `j` with operation `OP_I32_MUL`, and output of name generated by `gensym()`. Then, this will be inputted to the next CXExpression, along with 3, and output to `i`, with op `OP_I32_ADD`. 

The functions used with CXArguments are as follows:
* MakeArgument (name, fileName, fileLine) - returns a CXArgument with fields Name, FileName, and FileLine set accordingly.
* MakeField (name, typ, fileName, fileLine) - same as MakeArgument, but sets Type to typ.
* MakeGlobal (name, typ, fileName, fileLine) - same as MakeField, but calculates argument size with GetArgSize(typ), updates HeapOffset, and sets Offset to previous HeapOffset and sets Size to calculated size.

The methods CXArgument possesses are as follows:
* AddPackage (pkg) - returns the CXArgument with the Package field set to pkg.
* AddType (typ) - Checks to see if typ is an existing TypeCode. If so, Type is changed to typ, Size and TotalSize are changed to GetArgSize of typ, and DeclarationSpecifiers appends TYPE_BASIC. If not, Type is changed to TYPE_UNDEFINED. Returns updated CXArgument.
* AddInput, AddOutput (inp) - appends inp to Inputs or Outputs, then, if inp.Package is nil, sets inp.Package to Package, and returns updated CXArgument.

It is worth noting that CXArguments contain all the necessary data for calculating dereferences at runtime, making dereference calculations not instructions themselves, but something that is expected to be possible just from the data in a single CXArgument. Moreover, CXArguments comprise the root pointers for the garbage collector, meaning the entire heap object tree for a single CXArgument should fully represent all heap objects referenced currently under this one CXArgument. Albeit the garbage collector relies on CXFunctions to store these root CXArguments in a ListOfPointers, when it comes to traversing these objects (MarkObjectsTree and updatePointerTree in [cx/op.go](https://github.com/skycoin/cx/blob/develop/cx/op.go), the GC logic), CXArgument needs to provide everything.

## CXExpression

The core of the CX program structure are [CXExpressions](https://github.com/skycoin/cx/blob/develop/cx/cxexpression.go), which each represent a single instruction (not the operation itself, but the operation, inputs, and outputs) to perform. They comprise user-declared functions, and are the muscles of a CX program. 

CXExpressions have the following fields:
* Inputs - an array of CXArguments which are the inputs to the instruction.
* Outputs - the outputs.
* Label - the label of the expression, used by GOTO.
* Operator - a CXFunction representing the operation or function call to perform.
* Function - which function in the source code does this belong to? 
* Package - the CXPackage this expression resides in.
* FileName, FileLine - location in source where this CXExpression was created.
* ThenLines, ElseLines - used to calculate current line while executing for instructions that jump, like for, if/else, and goto.
* ScopeOperation - used to either initiate a new scope or exit the existing scope, or nothing if zero. 1 = begin, -1 = end.
* IsMethodCall, IsStructLiteral, IsArrayLiteral, IsUndType, IsBreak, IsContinue.

CXExpression functions/methods (first one is function, rest are methods):
* MakeAtomicOperatorExpression(op, fileName, fileLine) - creates CXExpression with Operator op, etc.
* GetInputs() - returns Inputs if present, or an error if there are no inputs.
* AddInput (param) - appends param to Inputs, & sets param.Package to Package if nil.
* RemoveInput() - removes the last element in Inputs, if not length zero.
* AddOutput, RemoveOutput - same thing as previous, but for Outputs.
* AddLabel (lbl) - sets Label to lbl.

## CXFunction

[CXFunctions](https://github.com/skycoin/cx/blob/develop/cx/cxfunction.go) are the operators in CX, whether they be custom operations, internal library operations, opcodes, or functions defined in CX. The Inputs and Outputs of the CXFunction differ from the CXExpression Inputs and Outputs because the ones the CXFunction have are parameters, not arguments. So CXExpressions store CXArguments that represent the actual input and output data during evaluation, while CXFunctions store inputs and outputs corresponding to: what kind of data types they accept; what kind of named parameters to declare; or otherwise, just type.

They have the following fields:
* Name - name of the function, if named.
* Package - the package this CXFunction resides in.
* IsNative - is this function native to CX? (i32.add, etc)
* OpCode - if IsNative, then this is non-zero and set to the operation it correlates to.
* Inputs, Outputs - Input and output parameters to the CXFunction.
* Expressions - all expressions in the function.
* Length - number of expressions in the function.
* Size - size, in memory, of the function.
* FileName, FileLine - used for debugging.
* ListOfPointers - used by the garbage collector. These are CXArguments that are the root pointers of the object trees in the heap. 
* CurrentExpression - used by the REPL and parser when checking function validity and processing expressions at compile time.

Functions/Methods (first two are functions, rest are methods): 
* MakeFunction - same as other makers for the previous objects.
* MakeNativeFunction - used to create Opcodes.
* GetExpressions, GetExpressionByLabel, GetExpressionByLine - getters for exactly what they sound like. For GetExpressionByLine, and for other “Line” things, line just means “get the expression in Expressions at index ‘line’”.
* GetCurrentExpression - returns CurrentExpression if not nil, otherwise first element in Expressions if not nil, otherwise an error.
* AddInput, RemoveInput, AddOutput, RemoveOutput - same as CXExpression.
* AddExpression - adds expression to the CXFunction’s Expressions. Also sets the expression’s Package to Package, Function to the caller, and sets CurrentExpression to the input expression, and increments Length.
* RemoveExpression - Not sure when this is used, but does the same thing, but does not unset expression’s Package or Function fields, nor unsets CurrentExpression, nor decrements Length.
* SelectExpression - takes in an integer, and tries to return the element with that index from Expressions. Throws error if Expressions is nil or length zero. If index is out of bounds, it either returns the first or last expression from Expressions, depending on which end the index goes out of bounds at. Also sets CurrentExpression to the grabbed expression.

## CXStruct

[CXStructs](https://github.com/skycoin/cx/blob/develop/cx/cxstruct.go) are the custom struct types in CX, albeit all struct types in CX are currently custom types. They have a Name, a Package they reside in, an array of CXArguments which represent the field names and types, and a Size, which is the precomputed CXStruct’s size in memory. They have a MakeStruct which takes in a name and outputs a blank CXStruct with said name, and also GetFields(), GetField(name), and more:
* AddField - tries to add the CXArgument input to Fields. If the input’s name shares the name of another field, then CX panics. Otherwise, the input’s Offset is set to the previous field’s Offset (or zero if there are no other fields), plus the last field’s * TotalSize (or zero, if nonexistent). Size is incremented by the size of the input.
* RemoveField - same thing as previous, but forgets to update the offset, Size, and other things.

## CXPackage

[CXPackages](https://github.com/skycoin/cx/blob/develop/cx/cxpackage.go) are just like Go packages, and store globals and other things:
* Name - name of the package.
* Imports - CXPackages which this CXPackage imports.
* Functions, Structs, Globals - exactly what you think they are. Globals has CXArguments.
* CurrentFunction, CurrentStruct - same as the case with CXFunction CurrentExpression.

Additionally, CXPackages have the same behavior as the getters, setters, makers, and selectors with CXFunctions, including the removal behavior being forgetful of things.

## CXProgram / Runtime

The CX Runtime consists mainly of a [CXProgram](https://github.com/skycoin/cx/blob/develop/cx/cxprogram.go) object, which itself is composed of:
* An array of bytes - the program memory (code, data, stack, heap segments)
* A callstack of CXCalls, consisting of the operator (a CXFunction) and frame pointer
* Several state registers:
  * Stacksize and Heapsize
  * Heapstartsat 
  * Stackpointer and Heappointer
  * Callcounter
* Additionally, a CXProgram stores two arrays of CXArguments - Inputs and Outputs - which represent OS arguments and outputs.
* A Version string and a CurrentPackage, used by the REPL and by the compiler
* The full array of Packages (CXPackage) in the program (essentially the AST/IR)
* Terminated - is the program terminated?
* CurrentPackage - the currently active package in the program. Used by REPL, Compiler.

CX Runtime also has CXCalls, which store an Operator, FramePoiner, and Line. 

CXPrograms are the root object of the entire AST for CX. 

The maker for CXProgram sets the CallStack to a default size, Memory, StackSize, HeapSize, HeapPointer, and Packages to default sizes.

The getters are GetCurrentPackage, GetCurrentStruct (CurrentPackage.CurrentStruct), GetCurrentFunction (CurrentPackage.CurrentFunction), GetCurrentExpression (CurrentPackage.CurrentFunction.CurrentExpression), GetGlobal(name), GetPackage(name), GetStruct(structname, packagename), GetFunction(funcname, packagename), GetCall (returns CallStack\[CallCounter\]), GetExpr (returns GetCall().Operator.Expressions\[GetCall().Line\]), GetOpCode (returns GetExpr().Operator.OpCode), and GetFramePointer.

There are also AddPackage and RemovePackage, which do exactly what they sound like. And also GetProgram, which returns PROGRAM, or an error if no program exists yet.

There are also Selectors like previously defined for CXFunctions and CXPackages-- specifically SelectPackage, SelectStruct, SelectFunction, and SelectExpression.

Finally, there is PrintAllObjects, a printing algorithm used for printing the program state when debugging.

Some miscellaneous makers in [cx/makers.go](https://github.com/skycoin/cx/blob/develop/cx/makers.go): generateTempVarName(name) (returns unique identifier with prefix name_); MakeDefaultValue (returns a blank `[]byte` the size of the specified type name), MakeValue (serializes the value and returns `*[]byte` of said value), MakeCall (returns a blank CXCall with the specified CXFunction as the Operator), and MakeIdentityOpName (used for making identity functions for storage).

The specific opcodes used by the CX runtime, among many other utilities, are found in the `cx/` [directory](https://github.com/skycoin/cx/tree/develop/cx) of this repository. 

