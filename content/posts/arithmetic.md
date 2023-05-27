---
title: 四则运算
date: 2019-01-11 17:51:33
---

### 项目github地址

[这里](https://github.com/GPN211314/arithmetic)

### 表达式生成部分（含开发时间表）

[这里](https://blog.csdn.net/weixin_40261309/article/details/86373440)

### 表达式计算部分

​	采用调度场算法直接计算中缀表达式的值或者先转换为后缀表达式再求值，在项目的主体部分采用了直接计算中缀表达式的值，在扩展部分，采用haskell语言先将中缀表达式转换为后缀表达式再求值。

​	在计算之前先做预处理将表达式的的两侧加括号（不影响表达式的值），将操作数与操作符从字符串中分离，并将两种乘方符号统一用一个符号表示，用python中的fractions库来支持分数操作。

- #### 定义优先级


```python
def precedence(x):
    if x == "+":
        return 2
    if x == "-":
        return 2
    if x == "*":
        return 3 
    if x == "/":
        return 3
    if x == "^":
        return 4
    if x == "(":
        return 1
    if x == ")":
        return 1

```

- #### 预处理


```python
            line = line.split('\n')[0]
    
            expression = "(" + line + ")"

            exps_list = []

            # 将**替换为^
            tmp_ls = expression.split("**")
            expression = "^".join(tmp_ls)
            # 将数字与符号分离
            for i in expression:
                if i.isdigit():
                    exps_list[-1] = 10*exps_list[-1] + int(i)
                else:
                    exps_list.append(i)
                    exps_list.append(0)
            exps_list.pop()

```

- #### 采用调度场算法计算中缀表达式值


```python
def SYA(i, num_stack, ops_stack):                    
    if i == "(":
        ops_stack.append(i)
    elif i == ")" and ops_stack[-1] == "(":
        ops_stack.pop()
    elif precedence(i) <= precedence(ops_stack[-1]) and (i != "^" or ops_stack[-1] != "^"):
        b = num_stack.pop()
        a = num_stack.pop()
        op = ops_stack.pop()
        tmp = calculate(a, b, op)
        num_stack.append(tmp)
        SYA(i, num_stack, ops_stack)
    else:
        ops_stack.append(i)

```



```python
            # 操作数栈与符号栈
            num_stack = []
            ops_stack = []

            for i in exps_list:
                if type(i) == int:
                    num_stack.append(Fraction(i))
                else:
                    SYA(i, num_stack, ops_stack)                    

```

### 整合调用及输入输出部分

- #### 帮助信息

  ```python
  def help():
      print("-c  根据提示选择乘方符号，生成1000道不重复的四则运算题目\n"
              "-s  用户输入结果，判断对错，输入为空时退出并给出统计结果\n-h  显示当前信息")
  
  ```

- #### 调用部分

  ```python
  def main(argv):
      if len(argv) != 2:
          help()
      else:
          if argv[1] == "-c":
              while True:
                  powers = input("请输入乘方符号（**或^）：")
                  if powers == "**" or powers == "^":
                      break
                  else:
                      print("输入无效！！！")
              ob = Generate.Generate(powers)
              ob.generate_arithmetic()
          elif argv[1] == "-s":
              Calculate.main()
          else:
              help()
  
  ```




### 性能分析

使用VS对表达式计算部分进行性能分析，从中我们可以看到大部分时间都用在判断对错时的输出上了

![](/images/xingneng.png)

### 测试

对表达式计算部分的各个模块进行了测试

![](/images/ceshi.png)

### 扩展部分采用haskell实现

- #### 表达式计算部分

  [这里](https://gpn211314.github.io/2018/12/30/infix2rpn/)

- #### 表达式生成部分

  [这里](https://gpn211314.github.io/2019/01/14/arith-create/)

- #### 整合调用部分

  ##### 根据命令行参数调用不同的部分

  ```haskell
  main = do
       args <- getArgs
       x <- randomIO::IO Int
       case args of
         ["-c"] -> writeFile "question.txt" $ unlines.
           (map show).take 1000.sameFilter.
           (map (head.(foldl buildExp []).words)).
             creatExpLs $ mkStdGen x
         ["-s"] -> readFile "question.txt" >>= return.lines >>= tof
         _ -> putStrLn "同时支持两种乘方操作\n-c  生成1000道不重复的四则运算题目\n-s  用户输入结果，判断对错，输入为空时退出并给出统计结果\n-h  显示当前信息"
  
  ```

  ##### 对用户输入答案判断对错，并给出最终对错数量

  ```haskell
  --传统左折叠不能中断，为了实现当用户输入退出命令时退出，并统计结果，对左折叠进行了修改
  foldl'::(IO (Int,Int,Int) -> String -> IO (Int,Int,Int)) -> IO (Int,Int,Int) ->[String] -> IO (Int,Int,Int)
  foldl' _ zero [] = zero
  foldl' step zero (x:xs) = do
    (a,b,c) <- zero
    case c of
      1 -> return (a,b,c)
      _ -> foldl' step (step (return (a,b,c)) x) xs
  
  
  --判断用户输入的结果的正误，并给出最后统计结果
  tof expr = (foldl' func (return (0,0,0)) expr >>= (\(u, v, _) -> putStrLn ("正确"++ (show u) ++ "道，共" ++ (show v) ++ "道")))
    where func s t = do
           (a,b,c) <- s
           putStr (t ++ " = ") 
           hFlush stdout
           x <- getLine
           case x of
               "" -> s >>= (\(a,b,_) -> return (a,b,1))
               _ -> if let y=(toString.solveRpn.toRpn.preProcess2.preProcess1) t in (y `seq` x == y)
                      then putStrLn "正确！" >> return (a+1, b+1,0) 
                      else putStrLn "错误！" >> return (a, b+1,0) 
  
  ```


