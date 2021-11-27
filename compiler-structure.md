# Kotlin compiler structure
Kotlin compiler is built from two major parts, usually referred to as **frontend** and **backend**.

In frontend, compiler takes input text files and tries to create a model of the program. It includes parsing the files and analyzing them for correctness. A large part of compiler frontend is shared with Intellij IDEs, so you may find a lot of similarities between them. In the end, frontend generates intermediate representation (IR) of the compiled module which passes it over to backend.

Taking over from frontend stage, backend receives IR and generates platform specific artifacts from it. Backend stages include optimizing IR and shaping it for platform codegen ("lowering") as well as generation of binaries.

Both frontend and backend can be separated into multiple smaller stages. The most notable of them for plugin development are presented below:
## Frontend
### Stage 1: Lexing and parsing

The first part of compiler pipeline, lexer, annotates code string with a sequence of tokens. These tokens attach some meaning to particular substrings (keywords, operators, etc).

<table>
<tr>
  <th>Example input</th>
  <th>Example output (formatted)</th>
</tr>
<tr>
<td>

```kotlin
class Test {  
    fun example() {  }
}
```

</td>
<td>

```
CLASS_KEYWORD WHITE_SPACE IDENTIFIER("Test") WHITE_SPACE LBRACE
     FUN_KEYWORD IDENTIFIER("example") LPAR RPAR LBRACE WHITE_SPACE RBRACE
RBRACE
``` 

</td>
</tr>
</table>

With these tokens ready, parser creates a tree representation of the compilation module (commonly referred as AST - abstract syntax tree). This AST is built from [`ASTNode`]) instances, but current compiler frontend doesn't use it directly. Instead, it is modified into PSI (Program Structure Interface), a higher level abstraction which creates an appropriate instance of `KtElement` out of `ASTNode`.

> Note: new compiler frontend (built with FIR) operates quite a bit differently. See [[fir]] to find out more about they way it works.

<table>
<tr>
  <th>Example input</th>
  <th>Example output (formatted tree)</th>
</tr>
<tr>
<td>

```kotlin
class Test {  
    fun example() {  }
}
```

</td>
<td>

```
KtFile
  KtClass 
    PsiElement(CLASS_KEYWORD)
    PsiWhiteSpace
    PsiElement("Test")
    PsiWhiteSpace
    KtClassBody
      PsiElement(LBRACE)
      PsiWhiteSpace
      KtFunction
        PsiElement(FUN_KEYWORD)
        PsiWhiteSpace
        KtParameterList
          PsiElement(LPAR)
          PsiElement(RPAR)
        PsiWhiteSpace  
        KtBlockExpression
          PsiElement(LBRACE)
          PsiWhiteSpace
          PsiElement(RBRACE)
      PsiWhiteSpace
      PsiElement(RBRACE)
```

</td>
</tr>
</table>

Compiler doesn't provide any extension points for plugins in this stage, as it would directly affect syntax of the language. To learn more about internals of this stage, you can refer to [custom language support section](https://plugins.jetbrains.com/docs/intellij/implementing-parser-and-psi.html) of IDEA documentation, as Kotlin uses the same mechanisms. 
### Stage 2: Analysis
With PSI ready, frontend launches analyzer. It does a pass through the PSI tree, perfoming various checks and identifying classes, functions, types and many other things. The result of this stage is the additional metadata for PSI tree saved in `BindingContext`. Some examples:
  - Declarations (e.g. classes, functions and variables) get attached information about their types, parameters and declarations they contain in form of `descriptors`.
  - Function calls and variable references are associated with the `descriptors` they are calling through `ResolvedCall`.
  - Types of functions and variables are resolved and stored in a form of `KotlinType`

Similarly, Kotlin compiler reads through contents of dependency artifacts (.jar, .klib and others) and retrieves similar metadata about declarations to make sure that calls to external libraries are also resolved correctly.

<table>
<tr>
  <th>Example input</th>
  <th>Example analysis metadata</th>
</tr>
<tr>
<td>

```kotlin
class Test {  
    fun example() { 
      println("Hello world!") 
    }
}
```

</td>
<td>

```
# Produced class and function descriptors
ClassDescriptor: 
  name: Test 
  typeParameters: []
    FunctionDescriptor: 
      name: example 
      returnType: Unit
      valueParameters: []
      typeParameters: []

# Call is associated with a PSI element, but not the function it is contained in,
# as descriptors don't keep information about bodies
ResolvedCall
  descriptor: FunctionDescriptor(kotlin.println)
  valueArguments: ["Hello world"]
  typeArguments: []
  returnType: Unit
```

</td>
</tr>
</table>


Compiler plugins are able to control analysis through `AnalysisHandlerExtension` and add custom descriptors through `SyntheticResolveExtension`. For additional checks, compiler provides `CallChecker` and `DeclarationChecker` to perform checks on calls and declarations respectively. 

### Stage 3: Creating intermediate representation (Psi2IR)

After analysis is finished, compiler uses all collected information about source code to produce IR modules for backend. IR is represented as a structured tree (somewhat similar to PSI), combining information from `BindingContext` and PSI nodes, making codegen process more efficient.

<table>
<tr>
  <th>Example input</th>
  <th>Example IR</th>
</tr>
<tr>
<td>

```kotlin
class Test {  
    fun example() { 
      println("Hello world!") 
    }
}
```

</td>
<td>

```
IR_MODULE
  IR_FILE name: "Test.kt"
    IR_CLASS 
      symbol: IrClassSymbol(descriptor for class "Test")
      name: Test 
      typeParameters: [] 
      declarations:
        IR_FUNCTION
          symbol: IrFunctionSymbol(descriptor for function "example")
          name: example
          valueParameters: []
          typeParameters: []
          type: Unit
          body:
            IR_BLOCK_BODY:
              IR_CALL
                symbol: IrFunctionSymbol(descriptor for function "println")
                valueArguments: [IR_CONST("Hello world!")]
                typeArguments: []
                returnType: Unit
```

</td>
</tr>
</table>

If compiler reaches this stage, it is safe to assume that Kotlin code for compilation module is valid, as later phase - backend - is only responsible for generating artifacts. For compiler plugin authors, it means that any custom checks should be performed **before** the IR stage. Backend extensions provide no way to report errors by default and, although workarounds exist, it is highly recommended to validate everything during analysis.
## Backend
Backend transforms and optimizes IR, later converting it to platform specific codegen.

### Stage 1: IR Lowerings
Lowering is a single pass over IR elements, performing certain logic and mutations over the tree. It splits backend logic into smaller pieces, with each pass focusing on one particular task. Some lowerings are shared between backends for each platform, allowing for effectively sharing optimization logic.

Compiler plugins can define their own lowerings, which are executed as a post-processing pass on the original IR. As plugin authors, we can change IR tree in any compatible way, with all downstream optimizations applied to it by default. However, it is the only IR extension point available at this particular moment of time, with compiler providing no access to the final IR it uses for codegen.

> Modules that are distributed as KLIB artifacts (JS and Native) don't go through backend step. They are still affected by compiler plugins (as they are technically not connected to backend, but Psi2IR postprocessing). 
> 
> Serialized IR and metadata (descriptors) from this step form the KLIB artifact and platform lowerings and the final codegen step are only run when building platform executable.

Every platform defines its own set of lowerings that optimize and shape IR structure to simplify the final conversion: generating platform binaries. You can find configurations for each platform in compiler source code ([example](https://github.com/JetBrains/kotlin/blob/a7fef487c1a8bbfe777135141447a431d322ce78/compiler/ir/backend.jvm/lower/src/org/jetbrains/kotlin/backend/jvm/JvmLower.kt#L278)).

### Stage 2: Codegen
Codegen is the final stage of compilation where compiler transforms lowered IR into platform-specific output:
- Kotlin/JVM generates JVM bytecode with ObjectWeb ASM;
- Kotlin/JS creates JS code (with experimental WebAssembly support coming soon);
- Kotlin/Native transforms its IR into LLVM IR and uses LLVM to compile native binaries.

> Each platform has varied requirements for IR, as each backend accesses it slightly differently. It is highly recommended testing compiler plugins on all supported platforms, as plugin that works on JVM might easily fail on JS or Native.

Overall, platform specific codegen can easily be affected by errors in plugin generated IR, as those inconsistencies are frequently small enough to pass validation and not affect IR lowerings. Unfortunately, information provided about errors from that layer is often scarse, so the best advice when analyzing them is to have compiler sources at hand to look up failing assertions and potentially debug compiler when it fails. See [[tooling]] for more information.

## See also:
- [[plugin]]
- [[fir]]
- [[tooling]]

[//begin]: # "Autogenerated link references for markdown compatibility"
[fir]: frontend/fir.md "FIR: New compiler frontend"
[tooling]: tooling.md "tooling"
[plugin]: plugins/plugin.md "Anatomy of a compiler plugin"
[//end]: # "Autogenerated link references"