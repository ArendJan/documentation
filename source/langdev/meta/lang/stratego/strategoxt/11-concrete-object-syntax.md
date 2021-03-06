```eval_rst
.. highlight:: str
```

# 11. Concrete Object Syntax

Stratego programs can be used to analyze, generate, and transform object programs. In this process object programs are structured data represented by terms. Terms support the easy composition and decomposition of abstract syntax trees. For applications such as compilers, programming with abstract syntax is adequate; only small fragments, i.e., a few constructors per pattern, are manipulated at a time. Often, object programs are reduced to a core language that only contains the essential constructs. The abstract syntax can then be used as an intermediate language, such that multiple languages can be expressed in it, and meta-programs can be reused for several source languages.

However, there are many applications of program transformation in which the use of abstract syntax is not adequate since the distance between the concrete programs that we understand and the abstract syntax trees used in specifications is too large. Even with pattern matching on algebraic data types, the construction of large code fragments in a program generator can become painful. For example, even the following tiny program pattern is easier to read in the concrete variant

    let d*
     in let var x ta := (e1*) in e2* end
    end

than the abstract variant

    Let(d*, [Let([VarDec(x, ta, Seq(e1*))], e2*)])

While abstract syntax is manageable for fragments of this size (and sometimes even more concise!), it becomes unpleasant to use when larger fragments need to be specified.

Besides the problems of understandability and complexity, there are other reasons why the use of abstract syntax may be undesirable. Desugaring to a core language is not always possible. For example, in the renovation of legacy code the goal is to repair the bugs in a program, but leave it intact otherwise. This entails that a much larger abstract syntax needs to be dealt with. Another occasion that calls for the use of concrete syntax is the definition of transformation or generation rules by users (programmers) rather than by compiler writers (meta-programmers). Other application areas that require concrete syntax are application generation and structured document (XML) processing.

Hence, it is desirable to have a meta-language that lets us write object-program fragments in the concrete syntax of the object language. This requires the extension of the meta-language with the syntax of the object language of interest, such that expressions in that language are interpreted as terms. In this chapter it is shown how the Stratego language based on abstract syntax terms is extended to support the use of concrete object syntax for terms.

## 11.1. Instrumenting Programs

To appreciate the need for concrete syntax in program transformation, it is illuminating to contrast the use of concrete syntax with the traditional use of abstract syntax in a larger example. Program _instrumentation_ is the extension of a program in a systematic way in order to obtain measurements during run-time. Instrumentation is used, for example, in debugging to get information about the run-time behavior of a program, and in profiling to collect statistics about about run-time and call frequency of program elements. Here we consider a simple instrumentation scheme that instruments Tiger functions with calls to trace functions.

The following Stratego fragment shows rewrite rules that instrument a function `f` such that it prints `f entry` on entry of the function and `f exit` at the exit. The actual printing is delegated to the functions `enterfun` and `exitfun`. Functions are instrumented differently than procedures, since the body of a function is an expression statement and the return value is the value of the expression. It is not possible to just glue a print statement or function call at the end of the body. Therefore, a let expression is introduced, which introduces a temporary variable to which the body expression of the function is assigned. The code for the functions `enterfun` and `exitfun` is generated by rule `IntroducePrinters`. Note that the declarations of the `Let` generated by that rule have been omitted.

    instrument =
      topdown(try(TraceProcedure + TraceFunction))
      ; IntroducePrinters
      ; simplify

    TraceProcedure :
      FunDec(f, x*, NoTp(), e) ->
      FunDec(f, x*, NoTp(),
             Seq([Call(Var("enterfun"),[String(f)]), e,
                  Call(Var("exitfun"),[String(f)])]))

    TraceFunction :
      FunDec(f, x*, Tp(tid), e) ->
      FunDec(f, x*, Tp(tid),
             Seq([Call(Var("enterfun"),[String(f)]),
                  Let([VarDec(x,Tp(tid),NilExp)],
                      [Assign(Var(x), e),
                       Call(Var("exitfun"),[String(f)]),
                       Var(x)])]))

    IntroducePrinters :
      e -> Let(..., e)

The next program program fragment implements the same instrumentation transformation, but now it uses _concrete syntax_.

    TraceProcedure :
      |[ function f(x*) = e ]| ->
      |[ function f(x*) = (enterfun(s); e; exitfun(s)) ]|
      where !f => s

    TraceFunction :
      |[ function f(x*) : tid = e ]| ->
      |[ function f(x*) : tid =
           (enterfun(s);
            let var x : tid
             in x := e; exitfun(s); x
            end) ]|
      where new => x ; !f => s

    IntroducePrinters :
      e -> |[ let var ind := 0
                  function enterfun(name : string) = (
                    ind := +(ind, 1);
                    for i := 2 to ind do print(" ");
                    print(name); print(" entry\\n"))
                  function exitfun(name : string) = (
                    for i := 2 to ind do print(" ");
                    ind := -(ind, 1);
                    print(name); print(" exit\\n"))
               in e end ]|

It is clear that the concrete syntax version is much more concise and easier to read. This is partly due to the fact that the concrete version is shorter than the abstract version: 225 bytes vs 320 bytes after eliminating all non-significant whitespace. However, the concrete version does not use much fewer lines. A more important reason for the increased understandability is that in order to read the concrete version it is not necessary to mentally translate the abstract syntax constructors into concrete ones. The implementation of `IntroducePrinters` is only shown in concrete syntax since its encoding in abstract syntax leads to unreadable code for code fragments of this size.

Note that these rewrite rules cannot be applied as such using simple innermost rewriting. After instrumenting a function declaration, it is still a function declaration and can thus be instrumented again. Therefore, we use a single pass top-down strategy for applying the rules.

## 11.2. Observations about Concrete Syntax Specifications

The example gives rise to several observations. The concrete syntax version can be read without knowledge of the abstract syntax. On the other hand, the abstract syntax version makes the tree structure of the expressions explicit. The abstract syntax version is much more verbose and is harder to read and write. Especially the definition of large code fragments such as in rule `IntroducePrinters` is unattractive in abstract syntax.

The abstract syntax version _implements_ the concrete syntax version. The concrete syntax version has all properties of the abstract syntax version: pattern matching, term structure, can be traversed, etc.. In short, the concrete syntax is just _syntactic sugar_ for the abstract syntax.

**Extension of the Meta Language.** The instrumentation rules make use of the concrete syntax of Tiger. However, program transformation should not be restricted to transformation of Tiger programs. Rather we would like to be able to handle arbitrary object languages. Thus, the object language or object languages that are used in a module should be a parameter to the compiler. The specification of instrumentation is based on the real syntax of Tiger, not on some combinators or infix expressions. This entails that the syntax of Stratego should be extended with the syntax of Tiger.

**Meta-Variables.** The patterns in the transformation rules are not just fragments of Tiger programs. Rather some elements of these fragments are considered as meta-variables. For example in the term

    |[ function f(x*) = e ]|

the identifiers `f`, `x*`, and `e` are not intended to be Tiger variables, but rather meta-variables, i.e., variables at the level of the Stratego specification, ranging over identifiers, lists of function arguments, and expressions, respectively.

**Antiquotation.** Instead of indicating meta-variables implicitly we could opt for an antiquotation mechanism that lets us splice in meta-level expressions into a concrete syntax fragment. For example, using `~` and `~*` as antiquotation operators, a variant of rule `TraceProcedure` becomes:

    TraceProcedure :
      |[ function ~f(~* x*) = ~e ]| ->
      |[ function ~f(~* x*) =
           (enterfun(~String(f)); ~e; exitfun(~String(f))) ]|

With such antiquotation operators it becomes possible to directly embed meta-level computations that produce a piece of code within a syntax fragment.

In the previous section we have seen how the extension of Stratego with concrete syntax for terms improves the readability of meta-programs. In this section we describe the techniques used to achieve this extension.

### 11.2.1. Extending the Meta Language

To embed the syntax of an object language in the meta language the syntax definitions of the two languages should be combined and the object language sorts should be injected into the appropriate meta language sorts. In the Stratego setting this is achieved as follows. The syntax of a Stratego module `m` is declared in the `m.meta` file, which declares the name of an SDF module. For instance, for modules using Tiger concrete syntax, i.e., using the extension of Stratego with Tiger, the `.meta` would contain

    Meta([Syntax("StrategoTiger")])

thus declaring SDF module `StrategoTiger.sdf` as defining the extension.

The SDF module combines the syntax of Stratego and the syntax of the object language(s) by importing the appropriate SDF modules. The syntax definition of Stratego is provided by the compiler. The syntax definition of the object language is provided by the user. For example, the following SDF module shows a fragment of the syntax of Stratego:

    module Stratego-Terms
    exports
      context-free syntax
        Int                             -> Term {cons("Int")}
        String                          -> Term {cons("Str")}
        Var                             -> Term {cons("Var")}
        Id "(" {Term ","}* ")"          -> Term {cons("Op")}
        Term "->" Term                  -> Rule {cons("RuleNoCond")}
        Term "->" Term "where" Strategy -> Rule {cons("Rule")}

The following SDF module `StrategoTiger`, defines the extension of Stratego with Tiger as object language.

    module StrategoTiger
    imports Stratego [ Term => StrategoTerm
                       Var => StrategoVar
                       Id => StrategoId
                       StrChar => StrategoStrChar ]
    imports Tiger Tiger-Variables
    exports
      context-free syntax
        "|[" Dec     "]|"    -> StrategoTerm {cons("ToTerm"),prefer}
        "|[" TypeDec "]|"    -> StrategoTerm {cons("ToTerm"),prefer}
        "|[" FunDec  "]|"    -> StrategoTerm {cons("ToTerm"),prefer}
        "|[" Exp     "]|"    -> StrategoTerm {cons("ToTerm"),prefer}
        "~"  StrategoTerm    -> Exp          {cons("FromTerm"),prefer}
        "~*" StrategoTerm    -> {Exp ","}+   {cons("FromTerm")}
        "~*" StrategoTerm    -> {Exp ";"}+   {cons("FromTerm")}
        "~int:" StrategoTerm -> IntConst     {cons("FromTerm")}

The module illustrates several remarkable aspects of the embedding of object languages in meta languages using SDF.

A combined syntax definition is created by just importing appropriate syntax definitions. This is possible since SDF is a modular syntax definition formalism. This is a rather unique feature of SDF and essential to this kind of language extension. Since only the full class of context-free grammars, and not any of its subclasses such as LL or LR, are closed under composition, modularity of syntax definitions requires support from a generalized parsing technique. SDF2 employs scannerless generalized-LR parsing.

The syntax definitions for two languages may partially overlap, e.g., define the same sorts. SDF2 supports renaming of sorts to avoid name clashes and ambiguities resulting from them. In the StrategoTiger module several sorts from the Stratego syntax definition (`Term`, `Id`, `Var`, and `StrChar`) are renamed since the Tiger definition also defines these names. In practice, instead of doing this renaming for each language extension, module `StrategoRenamed` provides a syntax definition of Stratego in which all sorts have been renamed.

The embedding of object language expressions in the meta-language is implemented by adding appropriate injections to the combined syntax definition. For example, the production

    "|[" Exp "]|" -> StrategoTerm {cons("ToTerm"),prefer}

declares that a Tiger expression (`Exp`) between `|[` and `]|` can be used everywhere where a `StrategoTerm` can be used. Furthermore, abstract syntax expressions (including meta-level computations) can be spliced into concrete syntax expressions using the `~` splice operators. To distinguish a term that should be interpreted as a list from a term that should be interpreted as a list _element_, the convention is to use a `~*` operator for splicing a list.

The declaration of these injections can be automated by generating an appropriate production for each sort as a transformation on the SDF definition of the object language. It is, however, useful that the embedding can be programmed by the meta-programmer to have full control over the selection of the sorts to be injected, and the syntax used for the injections.

Using the injection of meta-language `StrategoTerm`s into object language `Exp`ressions it is possible to distinguish meta-variables from object language identifiers. Thus, in the term `|[ var ~x := ~e]|`, the expressions `~x` and `~e` indicate meta-level terms, and hence `x` and `e` are meta-level variables.

However, it is attractive to write object patterns with as few squiggles as possible. This can be achieved through another feature of SDF, i.e., variable declarations. The following SDF module declares syntax schemata for meta variables.

    module Tiger-Variables
    exports
      variables
        [s][0-9]*      -> StrConst    {prefer}
        [xyzfgh][0-9]* -> Id          {prefer}
        [e][0-9]*      -> Exp         {prefer}
        "ta"[0-9]*     -> TypeAn      {prefer}
        "x"[0-9]* "*"  -> {FArg ","}+ {prefer}
        "d"[0-9]* "*"  -> Dec+        {prefer}
        "e"[0-9]* "*"  -> {Exp ";"}+  {prefer}

According to this declaration `x`, `y`, and `g10` are meta-variables for identifiers and `e`, `e1`, and `e1023` are meta-variables of sort `Exp`. The last three productions declare variables over lists using the convention that these are distinguished from other variables with an asterisk. Thus, `x*` and `x1*` range over lists of function arguments. The prefer attribute ensures that these identifiers are preferred over normal Tiger identifiers.

Parsing a module according to the combined syntax and mapping the parse tree to abstract syntax results in an abstract syntax tree that contains a mixture of meta- and object-language abstract syntax. Since the meta-language compiler only deals with meta-language abstract syntax, the embedded object-language abstract syntax needs to be expressed in terms of meta abstract syntax. For example, parsing the following Stratego rule

    |[ x := let d* in ~* e* end ]| -> |[ let d* in x := (~* e*) end ]|

with embedded Tiger expressions, results in the abstract syntax tree

    Rule(ToTerm(Assign(Var(meta-var("x")),
                       Let(meta-var("d*"),FromTerm(Var("e*"))))),
         ToTerm(Let(meta-var("d*"),
                    [Assign(Var(meta-var("x")),
                            Seq(FromTerm(Var("e*"))))])))

containing Tiger abstract syntax constructors (e.g., `Let`, `Var`, `Assign`) and meta-variables (`meta-var`). The transition from meta language to object language is marked by the `ToTerm` constructor, while the transition from meta-language to object-language is marked by the constructor `FromTerm`.

Such mixed abstract syntax trees can be normalized by _exploding_ all embedded abstract syntax to meta-language abstract syntax. Thus, the above tree should be exploded to the following pure Stratego abstract syntax:

    Rule(Op("Assign",[Op("Var",[Var("x")]),
                      Op("Let",[Var("d*"),Var("e*")])]),
         Op("Let",[Var("d*"),
                   Op("Cons",[Op("Assign",[Op("Var",[Var("x")]),
                                           Op("Seq",[Var("e*")])]),
                              Op("Nil",[])])]))

Observe that in this explosion all embedded constructor applications have been translated to the form `Op(C,[t1,...,tn])`. For example, the Tiger _variable_ constructor `Var(_)` becomes `Op("Var",[_])`, while the Stratego meta-variable `Var("e*")` remains untouched, and `meta-var`s become Stratego `Var`s. Also note how the list in the second argument of the second `Let` is exploded to a `Cons`/`Nil` list.

The resulting term corresponds to the abstract syntax for the rule

    Assign(Var(x),Let(d*,e*)) -> Let(d*,[Assign(Var(x),Seq(e*))])

written with abstract syntax notations for terms.

The explosion of embedded abstract syntax does not depend on the object language; it can be expressed generically, provided that embeddings are indicated with the `FromTerm` constructor.

**Disambiguating Quotations.** Sometimes the fragments used within quotations are too small for the parser to be able to disambiguate them. In those cases it is useful to have alternative versions of the quotation operators that make the sort of the fragment explicit. A useful, although somewhat verbose, convention is to use the sort of the fragment as operator:

    "exp" "|[" Exp "]|" -> StrategoTerm {cons("ToTerm")}

**Other Quotation Conventions.** The convention of using `|[...]|` and `~` as quotation and anti-quotation delimiters is inspired by the notation used in texts about semantics. It really depends on the application, the languages involved, and the _audience_ what kind of delimiters are most appropriate.

The following notation was inspired by active web pages is developed. For instance, the following quotation `%>...<%` and antiquotation `<%...%>` delimiters are defined for use of XML in Stratego programs:

    context-free syntax
      "%>" Content "<%" -> StrategoTerm {cons("ToTerm"),prefer}
      "<%=" StrategoTerm     "%>" -> Content {cons("FromTerm")}
      "<%"  StrategoStrategy "%>" -> Content {cons("FromApp")}

**Desugaring Patterns.** Some meta-programs first desugar a program before transforming it further. This reduces the number of constructs and shapes a program can have. For example, the Tiger binary operators are desugared to prefix form:

    DefTimes : |[ e1 * e2 ]|  -> |[ *(e1, e2) ]|
    DefPlus  : |[ e1 + e2 ]|  -> |[ +(e1, e2) ]|

or in abstract syntax

    DefPlus : Plus(e1, e2) -> BinOp(PLUS, e1, e2)

This makes it easy to write generic transformations for binary operators. However, all subsequent transformations on binary operators should then be done on these prefix forms, instead of on the usual infix form. However, users/meta-programmers think in terms of the infix operators and would like to write rules such as

    Simplify : |[ e + 0 ]| -> |[ e ]|

However, this rule will not match since the term to which it is applied has been desugared. Thus, it might be desirable to desugar embedded abstract syntax with the same rules with which programs are desugared. This phenomenon occurs in many forms ranging from removing parentheses and generalizing binary operators as above, to decorating abstract syntax trees with information resulting from static analysis such as type checking.

We have seen how the use of concrete object syntax can make the definition of transformation rules more readable.
