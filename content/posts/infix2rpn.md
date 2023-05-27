---
title: Haskell 中缀表达式转后缀表达式并计算
date: 2018-12-30 16:29:21
---

## 整数范围内的四则运算（包含乘方、分数运算,没做容错处理）

### 首先定义各运算符的优先级

```haskell
--定义各符号的优先级
--由于算法中已经设置了遇"("必定压栈，")"必定出栈，故只需将其优先级设置为最小与最大即可
precedence x = case x of
                 "+" -> 2 
                 "-" -> 2 
                 "*" -> 3
                 "/" -> 3
                 "^" -> 4 
                 "(" -> 1
                 ")" -> 5

```

### 预处理（将运算符与数字分开，并能同时处理**与^两种乘方运算符）

- #### 将表达式中的**全部用^替换

  ```haskell
  preProcess1 :: String -> String
  preProcess1 xs = foldr funDel [] $ zip ys (map ("*^" `isPrefixOf`) $ tails ys)
                where ys = foldr funInst [] $ zip xs (False:(init (map ("**" `isPrefixOf`) $ tails xs)))
                              where funInst (x, valBool) acc =
                                            case valBool of
                                                True -> '^':acc
                                                _    -> x:acc
                      funDel (x, valBool) acc = case valBool of
                                                  True -> acc
                                                  _    -> x:acc
  
  ```

- #### 将数字与符号分开，以减号开头的算式在减号前加上个0，并在整个算式加上括号（便于后边处理）

  ```haskell
  preProcess2 :: String -> [String]
  preProcess2 = let    addBracket zs = "(":(zs ++ [")"])
                       isAddSpace a b = (not.isDigit $ a) || (not.isDigit $ b)
                       addSpace s acc = case acc of
                                          (z:zs) -> if isAddSpace s z
                                                     then s:' ':acc
                                                     else s:acc
                                          _      -> s:acc
                       check ys | head ys == "-" = ("0":ys)
                                | otherwise      = ys
                  in addBracket.check.words.(foldr addSpace []) 
  
  ```

### 用fold实现[调度场算法](https://zh.wikipedia.org/wiki/%E8%B0%83%E5%BA%A6%E5%9C%BA%E7%AE%97%E6%B3%95)（Shunting Yard Algorithm）把中缀表达式转为后缀表达式

```haskell
toRpn :: [String] -> [String]
toRpn xs = reverse.fst $ foldl' infix2rpn ([],[]) xs--两个栈分别为操作数栈与运算符栈
              where infix2rpn (nums, ops) x = 
                     if isOp x then popStack nums ops else pushStack 
                        where popStack as bs = 
                                case bs of
                                  b:rest -> case x of
                                              "(" -> (as,(x:bs))
                                              ")" -> if b == "("
                                                        then (as, rest)
                                                        else popStack (b:as) rest--递归调用
                                              _ -> if (precedence x) <= (precedence b) && (x /= "^"||b /= "^")--由于^是右结合的
                                                      then popStack (b:as) rest
                                                      else (as ,(x:bs))
                                  _      -> (as ,(x:bs))

                              pushStack = ((x:nums), ops)
                              isOp = (`elem` ["+", "-", "*", "/", "^", "(", ")"])
```

### 计算后缀表达式的结果

```haskell
--计算后缀表达式，为了支持分数运算，所有结果均为分数
solveRpn :: (Read a, Show a, Integral a) => [String] -> Ratio a
solveRpn = head.foldl' func []
  where func :: (Read a, Integral a) => [Ratio a] -> String -> [Ratio a]
        func xs par=case xs of
                      (x:y:ys) -> case par of
                                    "*" -> (y * x):ys
                                    "-" -> (y - x):ys
                                    "/" -> (y / x):ys
                                    "^" -> (y ^^ (round x)):ys
                                    "+" -> (y + x):ys
                                    _   -> (fromIntegral.read) par:xs
                      _   -> (fromIntegral.read) par:xs

```

### 由于整个运算都用分数表示（为了支持分数运算），需要将分母为1的分数转换为整数

```haskell
toString :: (Integral a, Show a) => Ratio a -> String
toString x | denominator x == 1 = show.numerator $ x
           | otherwise          = (show.numerator) x ++ "/" ++ (show.denominator $ x)

```

### 由于getLine函数处理输入时有一些问题（误输入中文符号并退格后，终端显示与实际输入不相符），此处改为用haskeline库来实现输入

```haskell
main =  getArgs >>= (\x -> case x of
                              ["-c"] -> runInputT defaultSettings loop
                                          where loop = (getInputLine "" >>=
                                                              (\y -> case y of
                                                                       Nothing    -> return ()
                                                                       Just ""    -> return ()
                                                                       Just "exit"-> return ()
                                                                       Just "quit"-> return ()
                                                                       Just input -> (return.toString.solveRpn.toRpn.
                                                                         preProcess2.preProcess1) input >>= 
                                                                           outputStrLn >> loop
                                                              )
                                                       )
                              _    -> putStrLn "同时支持两种乘方操作\n-c  计算器交互\n-h  显示当前信息" 
                    )

```

### 用到时的一些库

```haskell
import System.Console.Haskeline--处理输入
import System.Environment (getArgs)--处理命令行参数
import Data.Ratio--处理分数
import Data.List--严格左折叠fold'用到
import Data.Char--用到isDigit

```

### [关于Haskell 后缀表达式转中缀表达式](https://rosettacode.org/wiki/Parsing/RPN_to_infix_conversion)