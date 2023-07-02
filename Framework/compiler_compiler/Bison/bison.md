# [Bison](https://www.gnu.org/software/bison/manual/bison.html#C_002b_002b-Parsers)

> This manual (10 September 2021) is for GNU Bison (version 3.8.1), the GNU parser generator.
>

## The Concept of Bison

### Languages and Context-Free Grammars

Backus-Naur Form or “BNF” 
LR(1) grammars
LALR(1)
IELR(1) or canonical LR(1)

The Bison parser reads a sequence of tokens as its input, and groups the tokens using the grammar rules. If the input is valid, the end result is that the entire token sequence reduces to a single grouping whose symbol is the grammar’s start symbol. If not, the parser reports a syntax error.