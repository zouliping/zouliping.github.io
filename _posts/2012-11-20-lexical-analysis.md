---
layout: post
title: Python实现的C语言词法分析
category: 编译原理
comments: true
image:
  feature: texture-feature-02.jpg
excerpt: 词法分析器的功能是输入源程序，输出单词符号。当初定义Token（单词种别，属性值）序列的时候，是将单词种别用数字来表示，后来再做语法分析的时候，发现用数字时不太合理的，所以又对单词的种别码进行了一番修改。
---

<section id="table-of-contents" class="toc">
  <header>
    <h3>Overview</h3>
  </header>
<div id="drawer" markdown="1">
*  Auto generated table of contents
{:toc}
</div>
</section>

词法分析器的功能是输入源程序，输出单词符号。当初定义Token（单词种别，属性值）序列的时候，是将单词种别用数字来表示，后来再做语法分析的时候，发现用数字时不太合理的，所以又对单词的种别码进行了一番修改。

我的程序的总体思路是先对源程序进行一遍扫描，将多余的空格和注释去除，然后再读一遍已经进行过预处理的源程序，进行单词的识别，转换成二元组，保存到token文件中，并建立符号表对标识符进行管理，如果发现了错误，对其的位置和错误信息进行打印。

在对单词的识别部分，我采用了有穷自动机的理论来进行识别。这样就可以根据现在的状态和输入符号决定其后继行为。因此在对单词的识别中，我画了很多的状态图来识别不同的单词，如字符串、数字等等。状态图的绘制中，本来想用visio来画的，后来的后来觉得太麻烦了，还是用了最快的手画的方法。

![image]({{ site.url }}/assets/images/c1.jpg)

![image]({{ site.url }}/assets/images/c2.jpg)

![image]({{ site.url }}/assets/images/c3.jpg)

![image]({{ site.url }}/assets/images/c4.jpg)

![image]({{ site.url }}/assets/images/c5.jpg)

![image]({{ site.url }}/assets/images/c6.jpg)

关于错误处理的方面，我对于词法分析阶段所能遇到的几种错误，如下图所示中的四种中的前三种都进行了相应的处理。但是对于第三点做的不太好，对字符常数中可以出现的字符限制的有点过于厉害，例如分号等在我的词法分析器中是不能再字符串中出现的。

测试程序如下，内包含主要的C语言的各种语句，含有少量的错误：

{% highlight C linenos %}
int main()
{
    int _a;
         char ch = 'f;
    floatb,centigrade,fahrj@enheit;
         char fd = '\n';
 
    printf("please inputa);
   scanf("%d",&a); /*mycomment1or***2*/
    printf("please inputb");
   scanf("%f",&b);
    if (a==8.1.6)
    {
       centigrade=095*(b-32)/9; /*itismyc5435omment*/
        printf("TheCentigrade is ",centigrade);  /*mess/age*/
    }
    else if (a!=0)
    {
       fahrenheit=(9/5.0)*b++32; /*mycontent*/
        printf("TheFahrenheit is fahrenheit);   /*hello****/
    }
    return 0;
}
{% endhighlight %}

运行结果如下图所示：

![image]({{ site.url }}/assets/images/c7.jpg)

这是用Python写的第一个稍微像点样的东西，所以很多地方写的不大好，代码结构也是有点混乱。总而言之，就是在这样的条件下把编译原理的第一次实验给写完了。接下来是我的水水的代码了。

{% highlight Python linenos %}
# -*- coding: utf-8 -*- 
'''
Created on 2012-10-18

@author: zouliping
'''
import string

_key = ("auto","break","case","char","const","continue","default",

"do","double","else","enum","extern","float","for",

"goto","if","int","long","register","return","short",

"signed","static","sizeof","struct","switch","typedef","union",

"unsigned","void","volatile","while")  # c语言的32个关键字

_abnormalChar = '@#$%^&*~' #标识符中可能出现的非法字符

_syn = ''  #单词的种别码
_p = 0  #下标
_value = ''  #存放词法分析出的单词
_content = '' #程序内容
_mstate = 0 #字符串的状态
_cstate = 0 #字符的状态
_dstate = 0 #整数和浮点数的状态
_line = 1 #代码的第几行
_mysymbol = [] #符号表

def outOfComment():
    '''去除代码中的注释'''
    global _content
    state = 0
    index = -1
    
    for c in _content:
        index = index + 1
        
        if state == 0:
            if c == '/':
                state = 1
                startIndex = index
            
        elif state == 1:
            if c == '*':
                state = 2
            else:
                state = 0
                
        elif state == 2:
            if c == '*':
                state = 3
            else:
                pass
            
        elif state == 3:
            if c == '/':
                endIndex = index + 1
                comment = _content[startIndex:endIndex]
                _content = _content.replace(comment,'') #将注释替换为空，并且将下标移动
                index = startIndex - 1
                state = 0
                
            elif c == '*':
                pass
            else:
                state = 2
                
def getMyProm():
    '''从文件中获取代码片段'''
    global _content
    myPro = open(r'E://test.txt','r')

    for line in myPro:
        if line != '\n':
            _content = "%s%s" %(_content,line.lstrip()) #效率更高的字符串拼接方法
        else:
            _content = "%s%s" %(_content,line)
    myPro.close()

def analysis(mystr):
    '''分析目标代码，生成token'''
    global _p,_value,_syn,_mstate,_dstate,_line,_cstate
    
    _value = ''
    ch = mystr[_p]
    _p += 1
    while ch == ' ':
        ch = mystr[_p]
        _p += 1
    if ch in string.letters or ch == '_':    ###############letter(letter|digit)*
        while ch in string.letters or ch in string.digits or ch == '_' or ch in _abnormalChar:
            _value += ch
            ch = mystr[_p]
            _p += 1
        _p -= 1
        
        for abnormal in _abnormalChar:
            if abnormal in _value:
                _syn = '@-6' #错误代码，标识符中含有非法字符
                break
            else:
                _syn = 'ID'
        
        for s in _key:
            if cmp(s,_value) == 0:
                _syn = _value.upper()               #############关键字
                break
        if _syn == 'ID':
            inSymbolTable(_value)
            
    elif ch == '\"':                        #############字符串
        while ch in string.letters or ch in '\"% ' :
            _value += ch
            if _mstate == 0:
                if ch == '\"':
                    _mstate = 1
            elif _mstate == 1:
                if ch == '\"':
                    _mstate = 2

            ch = mystr[_p]
            _p += 1
            
        if _mstate == 1:
            _syn = '@-2'     #错误代码，字符串不封闭
            _mstate = 0
            
        elif _mstate == 2:
            _mstate = 0
            _syn = 'STRING'
            
        _p -= 1    
        
    elif ch in string.digits:
        while ch in string.digits or ch == '.' or ch in string.letters:
            _value += ch
            if _dstate == 0:
                if ch == '0':
                    _dstate = 1
                else:
                    _dstate = 2
                    
            elif _dstate == 1:
                if ch == '.':
                    _dstate = 3
                else:
                    _dstate = 5
                    
            elif _dstate == 2:
                if ch == '.':
                    _dstate = 3
                
            ch = mystr[_p]
            _p += 1
        
        for char in string.letters:
            if char in _value:
                _syn = '@-7' #错误代码，数字和字母混合，如12AB56等
                _dstate = 0
                
                
        if _syn != '@-7':
            if _dstate == 5:
                _syn = '@-3' #错误代码，数字以0开头
                _dstate = 0
            else:    
                _dstate = 0
                if '.' not in _value:
                    _syn = 'DIGIT'               ##################digit digit*
                else:
                    if _value.count('.') == 1:
                        _syn = 'FRACTION'           ################## 浮点数    
                    else:
                        _syn = '@-5' #错误代码，浮点数中包含多个点，如1.2.3                
        _p -= 1
            
                
    elif ch == '\'':                    ################## 字符
        while ch in string.letters or ch in '@#$%&*\\\'\"':
            _value += ch
            if _cstate == 0:
                if ch == '\'':
                    _cstate = 1
                    
            elif _cstate == 1:
                if ch == '\\':
                    _cstate = 2
                elif ch in string.letters or ch in '@#$%&*':
                    _cstate = 3
                    
            elif _cstate == 2:
                if ch in 'nt':
                    _cstate = 3
                    
            elif _cstate == 3:
                if ch == '\'':
                    _cstate = 4
            ch = mystr[_p]
            _p += 1
        
        _p -= 1
        if _cstate == 4:
            _syn = 'CHARACTER'
            _cstate = 0
        else:
            _syn = '@-4'   #错误代码，字符不封闭
            _cstate = 0
                    
    elif ch == '<': 
        _value = ch
        ch = mystr[_p]
        
        if ch == '=':           ###########  '<='
            _value += ch
            _p += 1
            _syn = '<='
        else:                   ###########  '<'
            _syn = '<'
        
    elif ch == '>': 
        _value = ch
        ch = mystr[_p]
        
        if ch == '=':           ########### '>='
            _value += ch
            _p += 1
            _syn = '>='
        else:                   ########## '>'
            _syn = '>'
            
    elif ch == '!': 
        _value = ch
        ch = mystr[_p]
        
        if ch == '=':           ########## '!='
            _value += ch
            _p += 1
            _syn = '!='
        else:                   ########## '!'
            _syn = '!'
            
        
    elif ch == '+': 
        _value = ch
        ch = mystr[_p]
        
        if ch =='+':            ############ '++'
            _value += ch
            _p += 1
            _syn = '++'
        else :                  ############ '+'
            _syn = '+'
        
    elif ch == '-': 
        _value = ch
        ch = mystr[_p]
        
        if ch =='-':            ########### '--'
            _value += ch
            _p += 1
            _syn = '--'
        else :                  ########### '-'
            _syn = '-'
            
    elif ch == '=':  
        _value = ch
        ch = mystr[_p]
        
        if ch =='=':            ########### '=='
            _value += ch
            _p += 1
            _syn = '=='
        else :                  ########### '='
            _syn = '='
    
    elif ch == '&':
        _value = ch 
        ch = mystr[_p]
        
        if ch == '&':           ########### '&&'
            _value += ch
            _p += 1
            _syn = '&&'
        else:                   ########### '&'
            _syn = '&'
            
    elif ch == '|':
        _value = ch
        ch = mystr[_p]
        
        if ch == '|':           ########## '||'       
            _value += ch
            _p += 1
            _syn = '||'
        else:                   ########## '|'
            _syn = '|'
            
    elif ch == '*':             ########## '*'
        _value = ch
        _syn = '*'
        
    elif ch == '/':             ########## '/'
        _value = ch
        _syn = '/'
        
    elif ch ==';':              ########## ';'
        _value = ch
        _syn = ';'
        
    elif ch == '(':             ##########  '('
        _value = ch
        _syn = '('
        
    elif ch == ')':             ########### ')'
        _value = ch
        _syn = ')'
        
    elif ch == '{':             ########### '{'
        _value = ch
        _syn = '{'
        
    elif ch == '}':             ########### '}'    
        _value = ch
        _syn = '}'       
        
    elif ch == '[':             ########### '['    
        _value = ch
        _syn = '['   
        
    elif ch == ']':             ########### ']'    
        _value = ch
        _syn = ']'  
        
    elif ch == ',':             ########## ','    
        _value = ch
        _syn = ','   
    elif ch == '\n':
        _syn = '@-1'
         
def inSymbolTable(token):
    '''将关键字和标识符存进符号表'''
    global _mysymbol
    if token not in _mysymbol:
        _mysymbol.append(token)

if __name__ == '__main__': 
    getMyProm()
    outOfComment()
    
    symbolTableFile = open(r'E://symbol_table.txt','w')
    tokenFile = open(r'E://token.txt','w')
    while _p != len(_content):
        analysis(_content)
        if _syn == '@-1':
            _line += 1 #记录程序的行数
        elif _syn == '@-2':
            print '字符串 ' + _value + ' 不封闭! Error in line ' + str(_line)
        elif _syn == '@-3':
            print '数字 ' + _value + ' 错误，不能以0开头! Error in line ' + str(_line)
        elif _syn == '@-4':
            print '字符 ' + _value + ' 不封闭! Error in line ' + str(_line)
        elif _syn == '@-5':
            print '数字 ' + _value + ' 不合法! Error in line ' + str(_line)
        elif _syn == '@-6':
            print '标识符' + _value + ' 不能包含非法字符!Error in line ' + str(_line)
        elif _syn == '@-7':
            print '数字 ' + _value + ' 不合法,包含字母! Error in line ' + str(_line)
        else: #若程序中无词法错误的情况
            #print (_syn,_value)
            tokenFile.write(str(_syn)+'@'+_value+'\n')
        
    tokenFile.close()
    symbolTableFile.write('入口地址\t变量名\n')
    i = 0
    for symbolItem in _mysymbol:
        symbolTableFile.write(str(i)+'\t\t\t'+symbolItem+'\n')
        i += 1
    symbolTableFile.close()
{% endhighlight %}
