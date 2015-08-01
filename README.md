walkast
=======

The objective
-------------

This package aims to provide a simple way to access a language object and you can manipulate it as you want by passing a visitor-function for printing, transforming, analyzing, and generating a new code.

OOP or Functional Style?
------------------------

It is natural for an OOP language to use visitor-pattern, and it is common for a functional language to use pattern-matching. R, I think, is not suitable for the use of those ideas because semantics itself does not support those functionalities.

Related functions that already exist
------------------------------------

-   `codetools::walkCode` by [Luke Tierney](https://cran.r-project.org/web/packages/codetools/index.html)
-   `pryr::call_tree` (NSE version is `pryr::ast`) by [Hadley Wickham](https://cran.r-project.org/web/packages/pryr/index.html)

see [vignette](./vignettes/related_ast_functions.md) for a comparison among these functions.

Installation
------------

``` r
## install.packages("devtools")
devtools::install_github("tobcap/walkast")
library("walkast")
```

Main functions
--------------

#### `walk_ast(expr, visitor)`

-   expr: a language object (not accept an expression object)
-   visitor: an environment whose class name includes "visitor" and which must have six functions (leaf, call, hd, tl, first, last)

#### `make_visitor(leaf, call, hd, tl, initial, final, ...)`

A helper which creates visitor-class.

-   `leaf()`: a function that manipulates leaf part of langauge object (a symbol or an atomic)
-   `call()`: a function that manipulates call part of langauge object (call object)
-   `hd()`: a function that manipulates caller part (head of call object)
-   `tl()`: a function that manipulates arguments part of call (tail of call object)
-   `initial()`: a function that manipulates expr before running AST
-   `final()`: a function that manipulates expr after running AST
-   `...`: arbitrary functions or variables that you want to use

    ``` r
    # you can use R6 class to make `visitor()` if you want
    v0 <- make_visitor(leaf = function(x) if(is.numeric(x)) x * 2 else x)

    # need to define all functions
    library(R6)
    v1 <- R6Class(
        "visitor"
      , public = list(
          leaf = function(x) if(is.numeric(x)) x * 2 else x
        , call = identity
        , hd = identity
        , tl = identity
        , initial = identity
        , final = identity
        )
      )$new()

    walk_ast(quote(1 + 2 * 3), v0)
    walk_ast(quote(1 + 2 * 3), v1)
    ```

Other helper functions
----------------------

#### Printing

-   `show_tree()`
-   `show_lisp(quote_bin = FALSE)`
-   `show_r()`

#### Replacing

-   `replace(before, after)`
-   `nest_expr(expr, target, count)` this recursively calls walk\_ast()

#### Conversion

-   `to_list()`
-   `to_call()`

#### Combination

-   `%then%`

Examples
--------

``` r
library("walkast")
e1 <- quote(1 + 2)
```

``` r
walk_ast(e1)
#> 1 + 2
```

``` r
walk_ast(e1, show_tree())
#> List of 3
#>  $ : symbol +
#>  $ : num 1
#>  $ : num 2
#> NULL
```

``` r
mult2 <- make_visitor(leaf = function(x) if (is.numeric(x)) x * 2 else x)
walk_ast(e1, mult2)
#> 2 + 4

add1 <- make_visitor(leaf = function(x) if (is.numeric(x)) x + 1 else x)
walk_ast(e1, add1)
#> 2 + 3

walk_ast(e1, add1 %then% mult2)
#> 4 + 6
walk_ast(e1, mult2 %then% add1)
#> 3 + 5
```

``` r
walk_ast(e1, replace(quote(`+`), quote(`-`)))
#> 1 - 2
```

``` r
walk_ast(e1, replace(2, quote(x)))
#> 1 + x
```

``` r
e2 <- quote((1 + x) ^ 2)

nest_expr(e2, quote(x), 3)
#> (1 + (1 + (1 + x)^2)^2)^2

nest_expr(e2, quote(1 + x), 3)
#> (((1 + x)^2)^2)^2

nest_expr(quote(1 + 1 / x), quote(x), 5)
#> 1 + 1/(1 + 1/(1 + 1/(1 + 1/(1 + 1/x))))
```

``` r
e3 <- quote({
    x <- 1
    ++x
    print(x)
})

plus_plus <- make_visitor(
  call = function(x) 
    if (length(x) == 2 && identical(x[[1]], quote(`+`)) &&
        length(x[[2]]) == 2 && identical(x[[2]][[1]], quote(`+`)) &&
        is.symbol(sym <- x[[2]][[2]]))
    base::call("<-", sym, base::call("+", sym, 1)) else x
)

walk_ast(e3, plus_plus)
#> {
#>     x <- 1
#>     x <- x + 1
#>     print(x)
#> }

plus_plus2 <- make_visitor(
  call = function(x) {
    syms <- all.names(x)
    if (length(syms) == 3 &&
        syms[1:2] == c("+", "+") &&
        is.symbol(sym <- x[[2]][[2]]))
    base::call("<-", sym, base::call("+", sym, 1)) else x
  }
)

walk_ast(e3, plus_plus2)
#> {
#>     x <- 1
#>     x <- x + 1
#>     print(x)
#> }
```

ToDo (someday)
--------------

-   Get information of Type of each Node
-   Analyze context and identify an environment where a function is called
-   Check if expression is reducable

R's grammer
-----------

-   see <https://github.com/wch/r-source/blob/trunk/src/main/gram.y#L334-L454>

        %%
        prog    :   END_OF_INPUT
        |   '\n'
        |   expr_or_assign '\n'
        |   expr_or_assign ';'
        |   error
        ;
        expr_or_assign  :    expr
                    |    equal_assign
                    ;
        equal_assign    :    expr EQ_ASSIGN expr_or_assign
                    ;
        expr    :   NUM_CONST
        |   STR_CONST
        |   NULL_CONST
        |   SYMBOL
        |   '{' exprlist '}'
        |   '(' expr_or_assign ')'
        |   '-' expr %prec UMINUS
        |   '+' expr %prec UMINUS
        |   '!' expr %prec UNOT
        |   '~' expr %prec TILDE
        |   '?' expr
        |   expr ':'  expr
        |   expr '+'  expr
        |   expr '-' expr
        |   expr '*' expr
        |   expr '/' expr
        |   expr '^' expr
        |   expr SPECIAL expr
        |   expr '%' expr
        |   expr '~' expr
        |   expr '?' expr
        |   expr LT expr
        |   expr LE expr
        |   expr EQ expr
        |   expr NE expr
        |   expr GE expr
        |   expr GT expr
        |   expr AND expr
        |   expr OR expr
        |   expr AND2 expr
        |   expr OR2 expr
        |   expr LEFT_ASSIGN expr
        |   expr RIGHT_ASSIGN expr
        |   FUNCTION '(' formlist ')' cr expr_or_assign %prec LOW
        |   expr '(' sublist ')'
        |   IF ifcond expr_or_assign
        |   IF ifcond expr_or_assign ELSE expr_or_assign
        |   FOR forcond expr_or_assign %prec FOR
        |   WHILE cond expr_or_assign
        |   REPEAT expr_or_assign
        |   expr LBB sublist ']' ']'
        |   expr '[' sublist ']'
        |   SYMBOL NS_GET SYMBOL
        |   SYMBOL NS_GET STR_CONST
        |   STR_CONST NS_GET SYMBOL
        |   STR_CONST NS_GET STR_CONST
        |   SYMBOL NS_GET_INT SYMBOL
        |   SYMBOL NS_GET_INT STR_CONST
        |   STR_CONST NS_GET_INT SYMBOL
        |   STR_CONST NS_GET_INT STR_CONST
        |   expr '$' SYMBOL
        |   expr '$' STR_CONST
        |   expr '@' SYMBOL
        |   expr '@' STR_CONST
        |   NEXT
        |   BREAK
        ;
        cond    :   '(' expr ')'
        ;
        ifcond  :   '(' expr ')'
        ;
        forcond :   '(' SYMBOL IN expr ')'
        ;
        exprlist:
        |   expr_or_assign
        |   exprlist ';' expr_or_assign
        |   exprlist ';'
        |   exprlist '\n' expr_or_assign
        |   exprlist '\n'
        ;
        sublist :   sub
        |   sublist cr ',' sub
        ;
        sub :
        |   expr
        |   SYMBOL EQ_ASSIGN
        |   SYMBOL EQ_ASSIGN expr
        |   STR_CONST EQ_ASSIGN
        |   STR_CONST EQ_ASSIGN expr
        |   NULL_CONST EQ_ASSIGN
        |   NULL_CONST EQ_ASSIGN expr
        ;
        formlist:
        |   SYMBOL
        |   SYMBOL EQ_ASSIGN expr
        |   formlist ',' SYMBOL
        |   formlist ',' SYMBOL EQ_ASSIGN expr  
        ;
        cr  :                   { EatLines = 1; }
        ;
        %%
