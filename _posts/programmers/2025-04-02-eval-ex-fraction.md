---
title: "[Flex and Bison] 백준 - Fraction(No. 30855)"
last_modified_at: 2025-04-02T21:05:12+09:00
categories:
    - programmers
tags:
    - Flex
    - Bison
author_profile: true

---
# 문제 링크
[백준 - Fraction](https://www.acmicpc.net/problem/30855 "문제링크")

이번 글은 위 문제를 Flex와 Bison을 사용하여 구문 분석 및 의미 분석을 하여 해결하였다.

컴파일러 수업을 듣고, 해당 문제를 컴파일러 관점에서 풀 수 있을 것 같아 아래와 같이 해결하였다.
# 문제 풀이

[문제 풀이 전체 코드](https://github.com/minchoCoin/eval_ex_fraction)

## Lexical analysis with Flex
token.l
```c
%{
    #include <stdlib.h>
    #include "ast.h"
    #include "y.tab.h"
    
%}

%%
-?[0-9]     { yylval.ival = atoi(yytext); return NUM; }

[ \t]   ;
\(  return ('(');
\)  return (')');
\n  return (0);
.   {printf("'%c': illegal character\n"),yytext[0]; exit(-1);}    
%%
int yywrap()    {return 1;}
```

## Syntax analysis with Bison
ast.y
```c
 %{  #include <stdio.h>
#include <stdio.h>
#include "ast.h"
int yyerror(const char *msg), yylex();
Node *Root;
%}

%union {
int ival;
Node *pval;
}
%token <ival> NUM
%type <pval> Exp

%%
Prg : Exp { Root = $1; }
    ;

Exp : '(' Exp Exp Exp ')' { $$ = mkFractionNode($2,$3,$4);}
    | NUM {$$ = mkIntNode($1);}

%%
int main() { 
    yyparse();
    printTree(Root,0);
    Fraction result = evaluate(Root);
    printf("\n");
    printf("Answer: %d / %d\n",result.num, result.den);
}
int yyerror(const char *msg) { fputs(msg, stderr); return -1; }
```

ast.h
```cpp
#ifndef AST_H
#define AST_H


typedef enum {
    INT_NODE,
    FRACTION_NODE
} NodeType;


typedef struct{
    int num; // numerator
    int den; //denominator
} Fraction;

//(left + numerator/denominator)
typedef struct Node {
    NodeType type;
    union {
        int ival;                  
        struct {          
            struct Node *left;  
            struct Node *numerator;
            struct Node *denominator;
            
        } exFraction;
    } data;
} Node;


Node *mkIntNode(int n);
Node *mkFractionNode(Node* a, Node* b, Node* c);


void printTree(Node *node, int indent);

Fraction evaluate(Node* node);

#endif
```

ast.c
```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "ast.h"


Node *mkIntNode(int n) {
    Node *node = (Node *)malloc(sizeof(Node));
    node->type = INT_NODE;
    node->data.ival = n;
    return node;
}





Node *mkFractionNode(Node* a, Node* b, Node* c) {
    Node *node = (Node *)malloc(sizeof(Node));
    node->type = FRACTION_NODE;
    node->data.exFraction.left=a;
    node->data.exFraction.numerator=b;
    node->data.exFraction.denominator=c;
    return node;
}



void printTree(Node *node, int indent) {
    if (node == NULL) return;

    
    
    for (int i = 0; i < indent; i++) printf("   ");
    switch (node->type) {
        case INT_NODE:
            printf("Int(%d)\n", node->data.ival);
            break;
        case FRACTION_NODE:
            printf("Op(mkFraction)\n");
            printTree(node->data.exFraction.left,indent+1);
            printTree(node->data.exFraction.numerator,indent+1);
            printTree(node->data.exFraction.denominator,indent+1);
            break;
    }
}

int gcd(int a, int b){
    if(a<b) return gcd(b,a);
    if (b==0) return (a<0 ? -a:a);
    return gcd(b,a%b);
}

/* making irreducible fraction */
Fraction reduce(Fraction f){
    int g = gcd(f.num, f.den);
    f.num/=g;
    f.den/=g;
    if(f.den<0){
        f.den=-f.den;
        f.num=-f.num;
    }
    return f;
}

/* fraction addition: a + b */
Fraction fraction_add(Fraction a, Fraction b) {
    Fraction r;
    r.num = a.num * b.den + b.num * a.den;
    r.den = a.den * b.den;
    return reduce(r);
}

/* fraction division: a / b */
Fraction fraction_div(Fraction a, Fraction b) {
    Fraction r;
    r.num = a.num * b.den;
    r.den = a.den * b.num;
    return reduce(r);
}

Fraction evaluate(Node* node) {
    if (node == NULL) {
        //  (null node)
        Fraction zero = {0, 1};
        return zero;
    }

    if (node->type == INT_NODE) {
        // leaf node(integer node)
        Fraction r;
        r.num = node->data.ival;
        r.den = 1;
        return r;
    }
    
    
    Fraction A = evaluate(node->data.exFraction.left);
    Fraction B = evaluate(node->data.exFraction.numerator);
    Fraction C = evaluate(node->data.exFraction.denominator);

    
    Fraction divPart = fraction_div(B, C);
    Fraction result  = fraction_add(A, divPart);
    return result;
    
}
```

## compile with flex and bison

Makefile
```makefile
all:main

main: lex.yy.c y.tab.c y.tab.h ast.h
		gcc lex.yy.c y.tab.c ast.c -o main

lex.yy.c: token.l y.tab.h ast.h
		flex token.l

y.tab.c: ast.y ast.h
		bison -dy ast.y

y.tab.h: ast.y ast.h
		bison -dy ast.y

clean: main lex.yy.c y.tab.c y.tab.h
		rm main lex.yy.c y.tab.c y.tab.h
zip: Makefile token.l ast.y ast.c ast.h
	zip main.zip Makefile token.l ast.y ast.c ast.h
```

# 결과
## example 1
### input
```
(1 2 3)
```
### output
```
Op(mkFraction)
   Int(1)
   Int(2)
   Int(3)

Answer: 5 / 3
```

## example 2
### input
```
(2 ( 3 4 5) 7)
```
### output
```
Op(mkFraction)
   Int(2)
   Op(mkFraction)
      Int(3)
      Int(4)
      Int(5)
   Int(7)

Answer: 89 / 35
```

## example 3
### input
```
( ( 1 2 4 ) ( 5 2 3 ) ( 4 3 ( 2 7 3 ) ) )
```
### output
```
Op(mkFraction)
   Op(mkFraction)
      Int(1)
      Int(2)
      Int(4)
   Op(mkFraction)
      Int(5)
      Int(2)
      Int(3)
   Op(mkFraction)
      Int(4)
      Int(3)
      Op(mkFraction)
         Int(2)
         Int(7)
         Int(3)

Answer: 991 / 366
```