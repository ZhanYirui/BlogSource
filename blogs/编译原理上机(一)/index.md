---
title: "编译原理上机（一) - 词法分析器"
date: 2022-11-19
draft: false
description: "编译原理上机 - 词法分析器"
tags: ["编译原理"]
---
**西电编译原理大作业 - 函数绘图语言编译器C++实现 - Part 1 - 词法分析器**

author : Enly

time : 2022-11-19

school : Xidian university



**写在前头:**

下面给出的代码是我第一版写出来的，勉强能符合要求。我后来进行了代码的优化和功能的添加，希望各位能够借鉴我的思路自己写一遍。



**上机目的:**

> 例如:  ORIGIN IS (360,240); -- 将原点平移到(360,240)位置
>
> ​      SCALE IS (100,100); // 将横坐标和纵坐标都放大100倍
>
> ​      SCALE IS (100,100/3); -- 横坐标放大100倍，纵坐标放大100/3倍
>
> ​      ROT IS PI/2; // 图形逆时针旋转90度
>
> ​      FOR T FROM 0 TO 2PI STEP PI/50 DRAW(cos(T),sin(T)); //画圆
>
> ​      这是给定的一个绘图语言，有四种语句。ORIGIN语句，作用是将原点平移；SCALE语句，作用是将横纵坐标放大；ROT语句，作用是将图形旋转；FOR语句，作用是画图；还有--和//，作用是注释。

> 所以上机分三个部分:
>
> ​       (1) 词法分析器:目的是将这些语句中的单词一个个识别出来
>
> ​      (2) 语法分析器:目的是判断词法分析器得到的单词是否有错误，以及识别到的单词组成的语句符不符合规定
>
> ​      (3) 绘图器:通过识别到的语句进行画图

![img](https://p.ipic.vip/sfw238.png)

![img](https://p.ipic.vip/2ubtv0.png)





**第一部分:词法分析器 (完整代码在结尾)**

> 测试文本:
>
> ```
> origin is (350,400);
> scale is (10,10);
> rot is pi;
> for t from 0 to 2*pi step pi/10000 draw(16*sin(t)*sin(t)*sin(t),13*cos(t)-5*cos(2*t)-2*cos(3*t)-cos(4*t)); --桃心线
> origin is (200,200);
> scale is (100,100);
> for t from 0 to 2*pi step pi/10000 draw(cos(t)*cos(t)*cos(t),sin(t)*sin(t)*sin(t)); --星型线
> ```

​    ![img](https://p.ipic.vip/70axzu.png)![img](https://p.ipic.vip/mvnnxr.png)

词法分析器的目的说白了就是**将文件中的单词一个个提取出来并分类**

比如:ORIGIN IS (360,240); 将这个句子中的单词提取出来就是ORIGIN、IS、(、360、,、240、)、;  我们将提取到的单词进行分类，比如360和240就是同一类都是常数，于是我们可以将单词的种类分为:

> ​    **保留字: ORIGIN,SCALE,ROT,IS,TO,STEP,DRAW,FOR,FROM,T**
>
> ​    **分隔符: SEMICO**(;)**,L_BRACKET**(()**,R_BRACKET**())**,COMMA**(,)
>
> ​    **运算符: PLUS**(+)**,MINUS**(-)**,MUL**(*)**,DIV**(/)**,POWER**(**)
>
> ​    **参数: T**
>
> ​    **函数: FUNC**
>
> ​    **常数: CONST_ID**
>
> ​    **结尾记号: NONTOKEN**
>
> ​    **出错记号: ERRTOKEN**

我们利用一个枚举enum类型来将单词的种类给装起来:

```cpp
enum Token_Type
{
    ORIGIN,SCALE,ROT,IS,TO,STEP,DRAW,FOR,FROM,//保留字
    T, //参数
    SEMICO,L_BRACKET,R_BRACKET,COMMA, //; ( ) ,分隔符
    PLUS,MINUS,MUL,DIV,POWER,  //+ - * / **运算符
    FUNC, //函数
    CONST_ID, //常数
    NONTOKEN, //结尾记号
    ERRTOKEN //出错记号
};
```

单词有自己的属性，上面我们用enum装起来的只是单词的一个**单词种类属性**，单词还有**文本属性**，常数还有自己的**值属性**，函数还有自己的**函数指针属性**，还有每个单词在文件中的**行数属性**。比如Origin这个单词，它的单词种类是ORIGIN，文本是Origin，它不是常数所以值属性规定为0，也不是函数所以函数指针设为NULL；再比如360这个单词，它的单词种类是CONST_ID，文本是360，它是常数所以值属性为360，它不是函数所以函数指针设为NULL;再比如sin这个单词，它的单词种类是FUNC，文本是sin，它不是常数所以值属性为0，它是函数所以函数指针设为sin。

所以我们用struct定义一个单词结构体:

```cpp
typedef struct Tokens
{
    Token_Type type; //单词种类
    string lexeme; //单词的文本
    double value;  // 常数单词的值
    double (*FuncPtr)(double); // 函数单词的函数指针
    int TokenLine; // 单词在文件中的行数
} Tokens;
```

这时候我们意识到一个问题，当我们在文件里识别到Origin、Scale、sin、Pi这些字符串单词的时候我们要给它归类到保留字、函数、常数等我们预先定义的单词类型，但当我们识别到abcd、xyz这些字符串单词的时候，我们要给这些单词归类到ERRTOKEN类型。所以我们需要一个单词字典，当我们识别到字符串单词的时候，要先查找这个单词字典，如果在字典里没有找到，我们就要将它们归类到ERRTOKEN类型。

所以我们定义一个单词字典:

```cpp
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
```

利用C++的类，我们定义一个Scanner类。

这个类里有三个成员变量:

> ​    **string FileName;**
>
> ​    **string TokenBuffer;**
>
> ​    **fstream F;**

其中TokenBuffer是字符串缓冲区，因为我们**读文件是利用循环一个字符一个字符地读，所以我们要有一个字符串缓冲区来将字符一个个装起来**。比如文件里有个Origin单词，我们读文件时，先读到一个O字母，我们把O装到缓冲区，然后再读r字母，再装到缓冲区，以此类推，到最后读完这个单词的时候，缓冲区就装了Origin这个单词的文本值了。

类里还有七个成员函数:

> ​    **void OpenFile();**
>
> ​    **void CloseFile();**
>
> ​    **void EmptyBuffer();** 
>
> ​    **void AddCharToBuffer(char TempC);** 
>
> ​    **Tokens SearchCharInDict(string TempS);** 
>
> ​    **Tokens CreateTokens(Token_Type type,string lexeme,double value,double (\*FuncPtr)(double),int Line);**
>
> ​    **Tokens GetToken();**

接下来我依次来说明这些成员函数的作用:

void OpenFile();  这个函数作用很简单，就是输入文件名后打开文件。

```cpp
void OpenFile(){ // 输入文件名打开文件
        cin >> FileName;
        F.open(FileName,ios::in|ios::out);
}
```

void CloseFile();  这个函数就是关闭文件。

```cpp
void CloseFile(){   //关闭文件
        F.close();
}
```

void EmptyBuffer();  这个函数的作用是清空缓冲区，读上一个单词的时候上一个单词的文本值还在缓冲区里面，我们读下一个单词的时候就要先清空缓冲区。

```cpp
void EmptyBuffer(){  //清空缓冲区
        TokenBuffer = "";
}
```

void AddCharToBuffer(char TempC);  这个函数的作用就是将一个字符装到缓冲区里面。

```cpp
void AddCharToBuffer(char TempC){ //将字符添加到缓冲区
        TokenBuffer += TempC;
}
```

Tokens SearchCharInDict(string TempS);  这个函数的作用是查单词字典，上面我们说过了，当我们读字符串单词的时候，要先在单词字典里面进行查找，如果没找到的话要将其归类为ERRORTOKEN类型。任务规定语言对大小写不敏感，所以Origin,origin,ORIGIN都要识别为ORIGIN类型，所以我们将存在缓冲区的单词文本值传进去的时候，将其全部转换为大写字母，我这里是利用C++自带的算法库进行大写转换的。

```cpp
Tokens SearchCharInDict(string TempS){  //查单词字典
        Tokens T = {ERRTOKEN, TempS, 0.0, NULL,0};
        //利用C++算法库进行大写转换
        transform(TempS.begin(),TempS.end(),TempS.begin(),::toupper);
        for(int i = 0; i < 18; i ++) {
            if (TempS == TokenTab[i].lexeme)
            {
                T.type = TokenTab[i].type;
                T.value = TokenTab[i].value;
                T.FuncPtr = TokenTab[i].FuncPtr;
            }
        }
        return T;
}
```

Tokens CreateTokens(Token_Type type,string lexeme,double value,double (*FuncPtr)(double),int Line);  这个函数作用是传入单词的各个属性，返回一个单词，因为底下GetToken()函数在生成单词时有很多重复代码，所以就专门写了个函数进行单词生成。

```cpp
Tokens CreateToken(Token_Type type, string lexeme, double value, double (*FuncPtr)(double), int Line){    // 生成单词
        Tokens TempToken;
        TempToken.type = type;
        TempToken.lexeme = lexeme;
        TempToken.value = value;
        TempToken.FuncPtr = FuncPtr;
        TempToken.TokenLine = Line;
        return TempToken;
}
```

Tokens GetToken();  这个函数是Scanner类最核心的成员函数，作用是获取文件中的一个单词。

设计思路:我们读文件是一个字符一个字符地读。所以我们根据读到的字符进行不同的设计。

> ​    **当我们读到的字符是文件结尾字符 'EOF' 的时候，我们返回一个类型为NONTOKEN的单词。**
>
> ​    **当我们读到的字符是空格 ' ' 或者制表符 '\t' 的时候，不作处理，继续读下一个字符。**
>
> ​    **当我们读到的字符是换行符 '\n' 的时候，我们将行数加1，然后继续读下一个字符。**
>
> ​    **当我们读到的字符是字母 '[a-zA-Z]' 的时候，我们将字符装到缓冲区，然后读下一个字符，如果下一个字符还是字母，那就继续装到缓冲区，直到读的字符不是字母为止，这个时候缓冲区里面装的就是单词的文本值了。然后查单词字典，是预设的单词的话就返回一个预设的单词，不然就返回一个类型为ERRTOKEN的单词。**
>
> ​    **当我们读到的字符是数字 '[0-9]' 的时候，我们将字符装到缓冲区，然后读下一个字符，如果下一个字符还是数字，就继续装到缓冲区，直到读的字符不是数字为止。由于数字还能有小数，所以这时候如果读的字符是小数点 '.' ，那就继续装到缓冲区，然后读下一个字符，如果是数字就装到缓冲区，直到读的字符不是数字为止，这个时候缓冲区里面就是常数的文本值。最后返回一个类型为CONST_ID，值属性为利用stod()函数转换缓冲区里的文本值为常数值 的单词。**
>
> ​    **当我们读到的字符是分号 ';' 的时候，返回一个类型为SEMICO的单词。**
>
> ​    **当我们读到的字符是左括号 '(' 的时候，返回一个类型为L_BRACKET单词。**
>
> ​    **当我们读到的字符是分号右括号 ')' 的时候，返回一个类型为R_BRACKET的单词。**
>
> ​    **当我们读到的字符是逗号 ',' 的时候，返回一个类型为COMMA的单词。**
>
> ​    **当我们读到的字符是加号 '+' 的时候，返回一个类型为PLUS的单词。**
>
> ​    **当我们读到的字符是减号 '-' 的时候，由于 -- 是注释，我们需要继续读下一个字符，如果下一个字符不是 '-'，那么我们返回一个类型为MINUS的单词，如果下一个字符是 '-'，我们就要继续往后读，直到读的字符是换行符 '\n' 或者结束符 'EOF' 的时候结束，要是读的是 '\n'就将行数加1然后继续下一个循环，要是读的是 'EOF' 就返回一个类型为NONTOKEN的单词。**
>
> ​    **当我们读到的字符是除号 '/' 的时候，跟读减号 '-' 的方法一样**
>
> ​    **当我们读到的字符是乘号 '\*' 的时候，由于 \** 是幂次运算，我们需要继续读下一个字符，如果下一个字符不是 '\*'，那么我们返回一个类型为MUL的单词，如果下一个字符是 '\*'，我们就返回一个类型为POWER的单词。** 

```cpp
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
            else if(c == '-'){
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
```

Scanner类我们就已经设计好了，这个时候我选择用一个循环反复调用GetToken()函数将文件里面的所有单词存放到一个容器里面，后面设计语法分析器的时候就可以直接从容器里面取了，不过这也是看个人喜好，用一次调一次GetToken()函数也可以。

```cpp
vector<Tokens> TokenStream;
void LoadFileTokens()
{
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
```

再然后就是打印所有单词了，就不写了，直接放到完整代码里面，词法分析器部分就结束了。

## **词法分析器完整代码**

```cpp
#include <cmath>
#include <cctype>
#include <fstream>
#include <string>
#include <iostream>
#include <algorithm>
#include <vector>
#include <iomanip>
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

void Cout(string type, string lexeme, double value, string funcptr){
    cout << left << setw(10) << type << "\t|\t" << setw(12) << lexeme << "\t|\t" << setw(14) << value << "\t|\t" << setw(16) << funcptr << endl;
}

void COUT(Tokens t){
     switch (t.type) {
         case ORIGIN:Cout("ORIGIN",t.lexeme,t.value,"NULL");break;
         case SCALE:Cout("SCALE",t.lexeme,t.value,"NULL");break;
         case ROT:Cout("ROT",t.lexeme,t.value,"NULL");break;
         case IS:Cout("IS",t.lexeme,t.value,"NULL");break;
         case TO:Cout("TO",t.lexeme,t.value,"NULL");break;
         case STEP:Cout("STEP",t.lexeme,t.value,"NULL");break;
         case DRAW:Cout("DRAW",t.lexeme,t.value,"NULL");break;
         case FOR:Cout("FOR",t.lexeme,t.value,"NULL");break;
         case FROM:Cout("FROM",t.lexeme,t.value,"NULL");break;
         case T:Cout("T",t.lexeme,t.value,"NULL");break;
         case SEMICO:Cout("SEMICO",t.lexeme,t.value,"NULL");break;
         case L_BRACKET:Cout("L_BRACKET",t.lexeme,t.value,"NULL");break;
         case R_BRACKET:Cout("R_BRACKET",t.lexeme,t.value,"NULL");break;
         case COMMA:Cout("COMMA",t.lexeme,t.value,"NULL");break;
         case PLUS:Cout("PLUS",t.lexeme,t.value,"NULL");break;
         case MINUS:Cout("MINUS",t.lexeme,t.value,"NULL");break;
         case MUL:Cout("MUL",t.lexeme,t.value,"NULL");break;
         case DIV:Cout("DIV",t.lexeme,t.value,"NULL");break;
         case POWER:Cout("POWER",t.lexeme,t.value,"NULL");break;
         case FUNC:{
             string temp = t.lexeme;
             transform(temp.begin(),temp.end(),temp.begin(),::toupper);
             Cout("FUNC",t.lexeme,t.value,temp);
         };break;
         case CONST_ID:Cout("CONST_ID",t.lexeme,t.value,"NULL");;break;
         case NONTOKEN:Cout("NONTOKEN",t.lexeme,t.value,"NULL");;break;
         case ERRTOKEN:Cout("ERRTOKEN",t.lexeme,t.value,"NULL");;break;
     }
}

int main() {
    LoadFileTokens();
    cout << left << setw(10) << "Token_Type" << "\t \t" << setw(12) << "String_Value" << "\t \t" << setw(14) << "Constant_Value" << "\t \t" << setw(16) << "Function_Pointer" << endl;
    cout << "-------------------------------------------------------------------------------------" << endl;
    for(int i = 0; i < TokenStream.size(); i ++) COUT(TokenStream[i]);
    return 0;
}
```
