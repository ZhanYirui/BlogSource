---
title: "编译原理上机(三) - 绘图器"
date: 2022-11-21
draft: false
description: "编译原理上机(三) - 绘图器"
tags: ["编译原理"]
---
**西电编译原理大作业 - 函数绘图语言编译器C++实现 - Part 3 - 绘图器**

author:En1y

time:2022-11-20

school:Xidian university

**第三部分:绘图器 (完整代码在最后)**

>  测试文本:
>
> ```
> Origin Is (350,400);
> scale is (10,10);
> rot is pi;
> for t from 0 to 2*pi step pi/10000 draw(16*sin(t)*sin(t)*sin(t),13*cos(t)-5*cos(2*t)-2*cos(3*t)-cos(4*t)); --桃心线
> origin is (200,200);
> scale is (100,100);
> for t from 0 to 2*pi step pi/10000 draw(cos(t)*cos(t)*cos(t),sin(t)*sin(t)*sin(t)); --星型线
> ```

​    ![img](https://p.ipic.vip/58nucj.png)绘图器的代码是在第二部分语法分析器的基础上改编来的。

首先我们把第二部分语法分析器Parsers类里面的Parser()、Program()...、ForStatement()成员函数里面的"enter in"、"exit from"以及"matchtoken"语句删除。

在Parsers类的OriginStatement()、ScaleStatement()、RotStatement()、ForStatement()函数里面，我们得到了很多棵表达式树，在语法分析器里面我们是将它们用OutExprNode()函数先序遍历打印出来的，但是在绘图器里面，我们不需要把表达式树打印出来，我们需要表达式树它最终求出来的值，所以我们要把OutExprNode()函数改成GetExpValue()函数，两个函数很相似都是利用递归，我在语法分析器里已经分析过了，下面直接给出GetExpValue()函数的代码:

```cpp
double GetExpValue(struct ExprNode* Tree) {
        if (Tree == NULL) return 0.0;
        switch (Tree->OpCode) {
            case PLUS:
                return GetExpValue(Tree->Content.CaseOperator.Left) + GetExpValue(Tree->Content.CaseOperator.Right);
            case MINUS:
                return GetExpValue(Tree->Content.CaseOperator.Left) - GetExpValue(Tree->Content.CaseOperator.Right);
            case MUL:
                return GetExpValue(Tree->Content.CaseOperator.Left) * GetExpValue(Tree->Content.CaseOperator.Right);
            case DIV:
                return GetExpValue(Tree->Content.CaseOperator.Left) / GetExpValue(Tree->Content.CaseOperator.Right);
            case POWER:
                return pow(GetExpValue(Tree->Content.CaseOperator.Left), GetExpValue(Tree->Content.CaseOperator.Right));
            case FUNC:
                return Tree->Content.CaseFunc.MathFuncPtr(GetExpValue(Tree->Content.CaseFunc.Child));
            case CONST_ID:
                return Tree->Content.CaseConst;
            case T:
                return *(Tree->Content.CaseParmPtr);
            default:
                return 0.0;
        }
}
```

我们将语法分析器中的OutExprNode()函数换成GetExpValue()函数，这时候我们就能得到原点平移的横纵坐标的值origin_x和origin_y，横纵坐标放缩的值scale_x和scale_y，旋转的弧度值rot_ang，绘图时候参数T的起始值start、结束值end和步长step，但是现在注意了，FOR语句后面括号里面那两棵表达式树(struct ExprNode*)for_xptr,for_yptr里面带有参数T，所以我们不能直接在ForStatement()函数里面将这两棵表达式树的值求出来，而且根据要求，文件中Origin、Scale、Rot语句只影响它们后面的For语句，而且一个文件里面可以有多个FOR语句，所以我们在获得FOR语句括号里的两棵表达式树之后，我们要拿七个double容器将origin_x,origin_y,scale_x,scale_y,start,end,step装起来，拿两个struct ExprNode*容器将for_xptr和for_yptr装起来，这个时候这九个容器同一个下标里的内容就可以画出一个图形，ForStatement()函数改编后如下:

```cpp
   void ForStatement(){
        MatchToken(FOR);
        MatchToken(T);
        MatchToken(FROM);
        start_ptr = Expression();
        start = GetExpValue(start_ptr);
        MatchToken(TO);
        end_ptr = Expression();
        end = GetExpValue(end_ptr);
        MatchToken(STEP);
        step_ptr = Expression();
        step = GetExpValue(step_ptr);
        MatchToken(DRAW);
        MatchToken(L_BRACKET);
        for_xptr = Expression();
        For_X.push_back(for_xptr);
        MatchToken(COMMA);
        for_yptr = Expression();
        For_Y.push_back(for_yptr);
        Origin_X.push_back(origin_x);
        Origin_Y.push_back(origin_y);
        Scale_X.push_back(scale_x);
        Scale_Y.push_back(scale_y);
        Rot_ang.push_back(rot_ang);
        Start.push_back(start);
        End.push_back(end);
        Step.push_back(step);
        MatchToken(R_BRACKET);
}
```

现在我们有九个容器，每个容器都装着图形的一个参数，下一步就是来画图了。

我利用的是EasyX，Clion下EasyX的配置教程参考:[(8条消息) 在Clion中使用EasyX配置_Shine.Zhang的博客-CSDN博客](https://blog.csdn.net/qq_32939413/article/details/125393910)

只要能用<graphics.h>库就行了，各位自己去寻找导入方法吧。

接下来先写一个画单个点的函数void DrawXY(double for_x, double for_y, double origin_x, double origin_y, double scale_x, double scale_y, double rot_ang)，传进去的是点的坐标、横纵坐标平移值、横纵坐标拉伸值以及旋转弧度值，代码如下:

```cpp
void DrawXY(double for_x, double for_y, double origin_x,double origin_y,double scale_x,double scale_y,double rot_ang){
    double tempx, tempy;
    for_x *= scale_x;
    for_y *= scale_y;
    tempx = for_x;
    tempy = for_y;
    for_x = tempx * cos(rot_ang) + tempy * sin(rot_ang);
    for_y = tempy * cos(rot_ang) - tempx * sin(rot_ang);
    for_x +=  origin_x;
    for_y +=  origin_y;
    putpixel(for_x,for_y,RGB(255,255,255));
}
```

上面这段代码先将横纵坐标分别乘横纵坐标拉伸值，再根据旋转的规则:旋转后横坐标=旋转前横坐标*cos(旋转弧度)+旋转前纵坐标*sin(旋转弧度)，旋转后纵坐标=旋转前纵坐标*cos(旋转弧度)-旋转前横坐标*sin(旋转弧度) 将旋转后的横纵坐标求出来，再分别加上横纵坐标平移值，就得到了操作后的点，最后用putpixel()函数将点画出来。

接下来我们就要通过循环令Parameter变量不断变化从而画出所有点，DrawTotalXY函数如下:

```cpp
void DrawTotalXY(Parsers P, double Origin_X,double Origin_Y,double Scale_X,double Scale_Y,double Rot_ang, double Start, double End, double Step, struct ExprNode* For_X, struct ExprNode* For_Y) {
    double x,y;
    Parameter = Start;
    if (Step > 0) {
        while (Parameter <= End) {
            x = P.GetExpValue(For_X) ;
            y = P.GetExpValue(For_Y) ;
            DrawXY(x,y,Origin_X,Origin_Y,Scale_X,Scale_Y,Rot_ang);
            Parameter += Step;
        }
    }
    else if(Step < 0){
        while (Parameter >= End) {
            x = P.GetExpValue(For_X) ;
            y = P.GetExpValue(For_Y) ;
            DrawXY(x,y,Origin_X,Origin_Y,Scale_X,Scale_Y,Rot_ang);
            Parameter += Step;
        }
    }
}
```

最后在main函数里面，我们通过下标从九个容器中取参数传到DrawTotalX函数里面画出所有的图像即可，完整代码如下，绘图器部分就完成了，这次上机也完成了。

## **绘图器完整代码**

```cpp

#include <cmath>
#include <cctype>
#include <fstream>
#include <string>
#include <iostream>
#include <algorithm>
#include <vector>
#include <cstdlib>
#include <cstdarg>
#include <graphics.h>
#include <conio.h>
#define pi acos(-1.0)
#define e exp(1.0)
using namespace std;

enum Token_Type{
    ORIGIN,SCALE,ROT,IS,TO,STEP,DRAW,FOR,FROM,//保留字
    T, //参数
    SEMICO,L_BRACKET,R_BRACKET,COMMA, //; ( ) ,分隔符
    PLUS,MINUS,MUL,DIV,POWER,  //+ - * / **运算符
    FUNC, //函数
    CONST_ID, //常数
    NONTOKEN, //空记号
    ERRTOKEN //出错记号
};

typedef struct Tokens{
    Token_Type type;
    string lexeme;
    double value;
    double (*FuncPtr)(double);
    int TokenLine;
} Tokens;


static Tokens TokenTab[] = {
        {ORIGIN, "ORIGIN", 0.0, NULL,0},
        {SCALE, "SCALE", 0.0, NULL,0},
        {ROT, "ROT", 0.0, NULL,0},
        {IS, "IS", 0.0, NULL,0},
        {TO, "TO", 0.0, NULL,0},
        {STEP, "STEP", 0.0, NULL,0},
        {DRAW, "DRAW", 0.0, NULL,0},
        {FOR, "FOR", 0.0, NULL,0},
        {FROM, "FROM", 0.0, NULL,0},
        {T, "T", 0.0, NULL,0},
        {FUNC, "SIN", 0.0, sin,0},
        {FUNC, "COS", 0.0, cos,0},
        {FUNC, "TAN", 0.0, tan,0},
        {FUNC, "LN", 0.0, log,0},
        {FUNC, "EXP", 0.0, exp,0},
        {FUNC, "SQRT", 0.0, sqrt,0},
        {CONST_ID, "PI", pi, NULL,0},
        {CONST_ID, "E", e, NULL,0}
};

class Scanner{
public:
    void OpenFile(){ //打开文件
        cin >> FileName;
        F.open(FileName,ios::in|ios::out);
    }
    void CloseFile(){ //关闭文件
        F.close();
    }
    void EmptyBuffer(){ //清空缓冲区
        TokenBuffer = "";
    }
    void AddCharToBuffer(char TempC){ //将字符添加到缓冲区
        TokenBuffer += TempC;
    }
    Tokens SearchCharInDict(string TempS){ //查字典
        Tokens T = {ERRTOKEN, TempS, 0.0, NULL,0};
        transform(TempS.begin(),TempS.end(),TempS.begin(),::toupper);
        for(int i = 0; i < 18; i ++) {
            if (TempS == TokenTab[i].lexeme){
                T.type = TokenTab[i].type;
                T.value = TokenTab[i].value;
                T.FuncPtr = TokenTab[i].FuncPtr;
            }
        }
        return T;
    }
    Tokens CreateToken(Token_Type type, string lexeme, double value, double (*FuncPtr)(double), int Line){
        Tokens TempToken;
        TempToken.type = type;
        TempToken.lexeme = lexeme;
        TempToken.value = value;
        TempToken.FuncPtr = FuncPtr;
        TempToken.TokenLine = Line;
        return TempToken;
    }
    Tokens GetToken(){
        char c;
        static int line = 1; //行数
        Tokens TempT;
        EmptyBuffer();   //清空缓冲区
        while(1){
            c = F.get();
            if(c == EOF){
                TempT = CreateToken(NONTOKEN,"EOF",0.0,NULL,line);
                return TempT;
            }
            else if(c == ' ' || c == '\t') continue;
            else if(c == '\n'){ line++; continue;}
            else if(isalpha(c)){
                AddCharToBuffer(c);
                while(1){
                    c = F.get();
                    if(isalnum(c)) AddCharToBuffer(c);
                    else break;
                }
                F.putback(c);
                TempT = SearchCharInDict(TokenBuffer);
                TempT.TokenLine = line;
                return TempT;
            }
            else if(isdigit(c)){
                AddCharToBuffer(c);
                while(1){
                    c = F.get();
                    if(isdigit(c)) AddCharToBuffer(c);
                    else break;
                }
                if(c == '.') {
                    AddCharToBuffer(c);
                    while (1){
                        c = F.get();
                        if(isdigit(c)) AddCharToBuffer(c);
                        else break;
                    }
                }
                F.putback(c);
                TempT = CreateToken(CONST_ID,TokenBuffer, stod(TokenBuffer),NULL,line);
                return TempT;
            }
            else if(c == ';'){
                TempT = CreateToken(SEMICO,";",0.0,NULL,line);
                return TempT;
            }
            else if(c == '('){
                TempT = CreateToken(L_BRACKET,"(",0.0,NULL,line);
                return TempT;
            }
            else if(c == ')'){
                TempT = CreateToken(R_BRACKET,")",0.0,NULL,line);
                return TempT;
            }
            else if(c == ','){
                TempT = CreateToken(COMMA,",",0.0,NULL,line);
                return TempT;
            }
            else if(c == '+'){
                TempT = CreateToken(PLUS,"+",0.0,NULL,line);
                return TempT;
            }
            else if(c == '-') {
                c = F.get();
                if(c == '-'){
                    while(c!='\n' && c!=EOF) c = F.get();
                    if(c == EOF){
                        TempT = CreateToken(NONTOKEN,"EOF",0.0,NULL,line);
                        return TempT;
                    }
                    else {
                        line++;
                        continue;
                    }
                }
                else {
                    F.putback(c);
                    TempT = CreateToken(MINUS,"-",0.0,NULL,line);
                    return TempT;
                }
            }
            else if(c == '/'){
                c = F.get();
                if(c == '/'){
                    while(c!='\n'&&c!=EOF) c = F.get();
                    if(c == EOF){
                        TempT = CreateToken(NONTOKEN,"EOF",0.0,NULL,line);
                        return TempT;
                    }
                    else {
                        line++;
                        continue;
                    }
                }
                else {
                    F.putback(c);
                    TempT = CreateToken(DIV,"/",0.0,NULL,line);
                    return TempT;
                }
            }
            else if(c == '*'){
                c = F.get();
                if(c == '*'){
                    TempT = CreateToken(POWER,"**",0.0,NULL,line);
                    return TempT;
                }
                else{
                    F.putback(c);
                    TempT = CreateToken(MUL,"*",0.0,NULL,line);
                    return TempT;
                }
            }
            else{
                TempT = CreateToken(ERRTOKEN,"EOF",0.0,NULL,line);
                return TempT;
            }
        }
    }
private:
    string FileName, TokenBuffer = ""; //文件名和缓冲区
    fstream F;
};

vector<Tokens> TokenStream;
void LoadFileTokens(){
    Scanner s;
    Tokens t;
    cout << "Please Enter A File With Path:" << endl;
    s.OpenFile();
    t = s.GetToken();
    TokenStream.push_back(t);
    while (t.type != NONTOKEN){
        t = s.GetToken();
        TokenStream.push_back(t);
    }
    s.CloseFile();
}

typedef double (*FuncPtr)(double);
struct ExprNode{
    Token_Type OpCode;
    union{
        struct{
            ExprNode *Left, *Right;
        } CaseOperator;
        struct{
            ExprNode *Child;
            FuncPtr MathFuncPtr;
        }CaseFunc;
        double CaseConst;
        double *CaseParmPtr;
    }Content;
};

vector<double> Origin_X;
vector<double> Origin_Y;
vector<double> Scale_X;
vector<double> Scale_Y;
vector<double> Rot_ang;
vector<double> Start;
vector<double> End;
vector<double> Step;
vector<struct ExprNode*> For_X;
vector<struct ExprNode*> For_Y;

class Parsers{
public:
    Parsers(){LoadFileTokens();}
    struct ExprNode  *origin_xptr, *origin_yptr, *scale_xptr, *scale_yptr, *rot_ptr, *start_ptr, *end_ptr, *step_ptr, *for_xptr, *for_yptr;
    double Parameter = 0, origin_x = 0, origin_y = 0, scale_x = 1, scale_y = 1, rot_ang = 0, start, end, step;
    static int cnt;
    Tokens TempToken;
    struct ExprNode* MakeExprNode(Token_Type opcode,...){
        struct ExprNode *ExprPtr = new (struct ExprNode);
        ExprPtr -> OpCode = opcode;
        va_list ArgPtr;
        va_start(ArgPtr,opcode);
        switch (opcode) {
            case CONST_ID:ExprPtr->Content.CaseConst =(double) va_arg(ArgPtr,double );break;
            case T:ExprPtr->Content.CaseParmPtr = &Parameter;break;
            case FUNC:
                ExprPtr->Content.CaseFunc.MathFuncPtr = (FuncPtr) va_arg(ArgPtr,FuncPtr);
                ExprPtr->Content.CaseFunc.Child = (struct ExprNode*) va_arg(ArgPtr,struct ExprNode*);break;
            default:
                ExprPtr->Content.CaseOperator.Left = (struct ExprNode*) va_arg(ArgPtr,struct ExprNode*);
                ExprPtr->Content.CaseOperator.Right = (struct ExprNode*) va_arg(ArgPtr,struct ExprNode*);break;
        }
        va_end(ArgPtr);
        return ExprPtr;
    }
    double GetExpValue(struct ExprNode* Tree) {
        if (Tree == NULL) return 0.0;
        switch (Tree->OpCode) {
            case PLUS:
                return GetExpValue(Tree->Content.CaseOperator.Left) + GetExpValue(Tree->Content.CaseOperator.Right);
            case MINUS:
                return GetExpValue(Tree->Content.CaseOperator.Left) - GetExpValue(Tree->Content.CaseOperator.Right);
            case MUL:
                return GetExpValue(Tree->Content.CaseOperator.Left) * GetExpValue(Tree->Content.CaseOperator.Right);
            case DIV:
                return GetExpValue(Tree->Content.CaseOperator.Left) / GetExpValue(Tree->Content.CaseOperator.Right);
            case POWER:
                return pow(GetExpValue(Tree->Content.CaseOperator.Left), GetExpValue(Tree->Content.CaseOperator.Right));
            case FUNC:
                return Tree->Content.CaseFunc.MathFuncPtr(GetExpValue(Tree->Content.CaseFunc.Child));
            case CONST_ID:
                return Tree->Content.CaseConst;
            case T:
                return *(Tree->Content.CaseParmPtr);
            default:
                return 0.0;
        }
    }
    void FetchToken(){
        TempToken = TokenStream[cnt];
        cnt ++;
        if(TempToken.type == ERRTOKEN) SyntaxError(1);
    }
    void MatchToken(Token_Type t){
        if(TempToken.type != t) SyntaxError(2);
        FetchToken();
    }
    void SyntaxError(int x){
        if(x == 1){
            cout << "line No:" << TempToken.TokenLine << " Has An Error Token: " << TempToken.lexeme << endl;
            exit(0);
        }
        else if(x == 2) {
            cout << "line No:" << TempToken.TokenLine << " Has An Unexpected Token: " << TempToken.lexeme  << endl;
            exit(0);
        }
    }
    void Parser(){
        FetchToken();
        Program();
    }
    void Program(){
        while (TempToken.type != NONTOKEN){
            Statement();
            MatchToken(SEMICO);
        }
    }
    void Statement(){
        if(TempToken.type == ORIGIN) OriginStatement();
        else if(TempToken.type == SCALE) ScaleStatement();
        else if(TempToken.type == ROT) RotStatement();
        else if(TempToken.type == FOR) ForStatement();
        else SyntaxError(2);
    }
    void OriginStatement(){
        MatchToken(ORIGIN);
        MatchToken(IS);
        MatchToken(L_BRACKET);
        origin_xptr = Expression();
        origin_x = GetExpValue(origin_xptr);
        MatchToken(COMMA);
        origin_yptr = Expression();
        origin_y = GetExpValue(origin_yptr);
        MatchToken(R_BRACKET);
    }
    void RotStatement(){
        MatchToken(ROT);
        MatchToken(IS);
        rot_ptr = Expression();
        rot_ang = GetExpValue(rot_ptr);
    }
    void ScaleStatement(){
        MatchToken(SCALE);
        MatchToken(IS);
        MatchToken(L_BRACKET);
        scale_xptr = Expression();
        scale_x = GetExpValue(scale_xptr);
        MatchToken(COMMA);
        scale_yptr = Expression();
        scale_y = GetExpValue(scale_yptr);
        MatchToken(R_BRACKET);
    }
    void ForStatement(){
        MatchToken(FOR);
        MatchToken(T);
        MatchToken(FROM);
        start_ptr = Expression();
        start = GetExpValue(start_ptr);
        MatchToken(TO);
        end_ptr = Expression();
        end = GetExpValue(end_ptr);
        MatchToken(STEP);
        step_ptr = Expression();
        step = GetExpValue(step_ptr);
        MatchToken(DRAW);
        MatchToken(L_BRACKET);
        for_xptr = Expression();
        For_X.push_back(for_xptr);
        MatchToken(COMMA);
        for_yptr = Expression();
        For_Y.push_back(for_yptr);
        Origin_X.push_back(origin_x);
        Origin_Y.push_back(origin_y);
        Scale_X.push_back(scale_x);
        Scale_Y.push_back(scale_y);
        Rot_ang.push_back(rot_ang);
        MatchToken(R_BRACKET);
    }

    struct ExprNode* Expression(){
        struct ExprNode *left, *right;
        Token_Type token_tmp;
        left = Term();
        while (TempToken.type == PLUS || TempToken.type == MINUS)
        {
            token_tmp = TempToken.type;
            MatchToken(token_tmp);
            right = Term();
            left = MakeExprNode(token_tmp,left,right);
        }
        return left;
    }

    struct ExprNode* Term(){
        struct ExprNode *left, *right;
        Token_Type token_tmp;
        left = Factor();
        while (TempToken.type == MUL || TempToken.type == DIV)
        {
            token_tmp = TempToken.type;
            MatchToken(token_tmp);
            right = Factor();
            left = MakeExprNode(token_tmp,left,right);
        }
        return left;
    }

    struct ExprNode* Factor(){
        struct ExprNode *left, *right;
        if(TempToken.type == PLUS){
            MatchToken(PLUS);
            right = Factor();
            left = NULL;
            right = MakeExprNode(PLUS,left,right);
        }
        else if(TempToken.type == MINUS){
            MatchToken(MINUS);
            right = Factor();
            left = MakeExprNode(CONST_ID,0.0);
            right = MakeExprNode(MINUS,left,right);
        }
        else right = Component();
        return right;
    }

    struct ExprNode* Component(){
        struct ExprNode *left, *right;
        left = Atom();
        if(TempToken.type == POWER){
            MatchToken(POWER);
            right = Component();
            left = MakeExprNode(POWER,left,right);
        }
        return left;
    }

    struct ExprNode* Atom(){
        struct ExprNode *address, *tmp;
        double const_value;
        FuncPtr funcPtr_value;
        if(TempToken.type == CONST_ID){
            const_value = TempToken.value;
            MatchToken(CONST_ID);
            address = MakeExprNode(CONST_ID,const_value);
        }
        else if(TempToken.type == T){
            MatchToken(T);
            address = MakeExprNode(T);
        }
        else if(TempToken.type == FUNC){
            funcPtr_value = TempToken.FuncPtr;
            MatchToken(FUNC);
            MatchToken(L_BRACKET);
            tmp = Expression();
            address = MakeExprNode(FUNC,funcPtr_value,tmp);
            MatchToken(R_BRACKET);
        }
        else if(TempToken.type == L_BRACKET){
            MatchToken(L_BRACKET);
            address = Expression();
            MatchToken(R_BRACKET);
        }
        else SyntaxError(2);
        return address;
    }
};

int Parsers::cnt = 0;

void DrawXY(double for_x, double for_y, double origin_x,double origin_y,double scale_x,double scale_y,double rot_ang){
    double tempx, tempy;
    for_x *= scale_x;
    for_y *= scale_y;
    tempx = for_x;
    tempy = for_y;
    for_x = tempx * cos(rot_ang) + tempy * sin(rot_ang);
    for_y = tempy * cos(rot_ang) - tempx * sin(rot_ang);
    for_x +=  origin_x;
    for_y +=  origin_y;
    putpixel(for_x,for_y,RGB(255,255,255));
}

void DrawTotalXY(Parsers P, double Origin_X,double Origin_Y,double Scale_X,double Scale_Y,double Rot_ang, double Start, double End, double Step, struct ExprNode* For_X, struct ExprNode* For_Y) {
    double x,y;
    P.Parameter = Start;
    if (Step > 0) {
        while (P.Parameter <= End) {
            x = P.GetExpValue(For_X) ;
            y = P.GetExpValue(For_Y) ;
            DrawXY(x,y,Origin_X,Origin_Y,Scale_X,Scale_Y,Rot_ang);
            P.Parameter += Step;
        }
    }
    else if(Step < 0){
        while (P.Parameter >= End) {
            x = P.GetExpValue(For_X) ;
            y = P.GetExpValue(For_Y) ;
            DrawXY(x,y,Origin_X,Origin_Y,Scale_X,Scale_Y,Rot_ang);
            P.Parameter += Step;
        }
    }
}


int main() {
    Parsers P;
    P.Parser();
    initgraph(1000, 700);
    for(int i = 0; i < Start.size(); i ++)
        DrawTotalXY(P,Origin_X[i],Origin_Y[i],Scale_X[i],Scale_Y[i],Rot_ang[i],Start[i],End[i],Step[i],For_X[i],For_Y[i]);
    _getch();
    closegraph();
    return 0;
}


```



