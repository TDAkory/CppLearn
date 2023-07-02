# The Overall Layout of a Bison Grammar

```
%{
Prologue
%}

Bison declarations

%%
Grammar rules
%%
Epilogue
```

The ‘%%’, ‘%{’ and ‘%}’ are punctuation that appears in every Bison grammar file to separate the sections.

The prologue may define types and variables used in the actions. You can also use preprocessor commands to define macros used there, and use #include to include header files that do any of these things. You need to declare the lexical analyzer `yylex` and the error printer `yyerror` here, along with any other global identifiers used by the actions in the grammar rules.