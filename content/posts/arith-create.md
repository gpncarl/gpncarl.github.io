---
title: haskell四则运算
date: 2019-01-14 17:38:53
---

### 表达式生成

##### 难点："+","*"等满足交换律的表达式在交换后，表达式不能相同

很容易想到用树形数据结构表示比较合适，即 [左子树 根结点 右子树] 的形式

在这里定义为

```haskell
--定义表达式类型	单独一个数    或	   左表达式	   运算符      右表达式  
data Expression = Const String | Exp Expression String Expression
```

只需将此类型加入Eq类型类的实例，并定义(==)运算符，就可以对此类型的表达式进行判同。

```haskell
instance Eq Expression where
  Const a == Const b = a == b
  Const _ == Exp _ _ _ = False
  Exp _ _ _ == Const _ = False
  Exp a1 b1 c1 == Exp a2 b2 c2
    | b1 == b2 = case b1 of
                   "+" -> ((a1 == a2)&&(c1 == c2))||((a1 == c2)&&(a2 == c1))
                   "*" -> ((a1 == a2)&&(c1 == c2))||((a1 == c2)&&(a2 == c1))
                   _   -> (a1 == a2)&&(c1 == c2)
    | otherwise = False

```



##### 表达式生成时为了避免括号的影响，首先生成后缀表达式(语言性质导致随机数生成相较其它语言比较麻烦)

- 随机生成2~11中的一个数x来表示表达式的操作数的个数

- 将x随机分为两个数a，b之和与一个随机的运算符，再对a，b递归生成表达式

- 当x为1时随机生成一个数返回

  ```haskell
  creatRpn::Int -> StdGen -> (String, StdGen)
  creatRpn 1 g = (show a, b)
    where (a, b) = randomR (1,20) g::(Int, StdGen)
  creatRpn x g = ((fst $ creatRpn a g') ++ " " ++ (fst $ creatRpn b g'') ++ " " ++ op, gen''')
    where (a, gen) = randomR (1,x-1) g::(Int, StdGen)
          b = x-a
          (u, gen') = random gen::(Int, StdGen)
          g' = mkStdGen(u)
          (v, gen'') = random gen'::(Int, StdGen)
          g'' = mkStdGen(v)
          (nu, gen''') = randomR (0,4) gen''::(Int, StdGen)
          op = ["+", "-", "*", "/", "^"]!!nu 
  
  -- 递归生成一个随机生成的表达式无限列表
  creatExpLs::StdGen -> [String]
  creatExpLs x = a:(creatExpLs b)
    where (y, gen) = randomR (2,11) x::(Int, StdGen)
          (a, b) = creatRpn y gen
  
  ```


至此，已经随机生成了一个后缀表达式，每生成一个后缀表达式就将其转换为表达式类型

```haskell
buildExp :: [Expression] -> String -> [Expression]
buildExp stack x
    | not.isOp $ x = Const x : stack
    | otherwise    = Exp l x r : rest
        where r:l:rest = stack
              isOp = (`elem` ["^","*","/","+","-"])

```



##### 查重

对生成的表达式进行查重，如果相同就舍弃，直到生成1000个不同的表达式类型的表达式

```haskell
--过滤掉表达式无限列表中的相同的表达式
sameFilter::[Expression] -> [Expression]
sameFilter (x:xs) = x : (sameFilter $ filter (/=x) xs)

```



##### 将表达式类型转换成中缀表达式并写入文件

只用将表达式类型加入 Show 类型类的实例，并定义方法 show 将表达式类型根据优先级及结合方向，在合适的位置加上括号，转换成中缀表达式形式的字符串

```haskell
precedence :: Expression -> Int
precedence (Const _) = 5
precedence (Exp _ op _)
    | op `elem` ["^"]     = 4
    | op `elem` ["*","/"] = 3
    | op `elem` ["+","-"] = 2
    | otherwise = 0

leftAssoc :: Expression -> Bool
leftAssoc (Const _) = False
leftAssoc (Exp _ op _) = op `notElem` ["^","*","+"]

rightAssoc :: Expression -> Bool
rightAssoc (Const _) = False
rightAssoc (Exp _ op _) = op `elem` ["^"]

instance Show Expression where
    show (Const x) = x
    show exp@(Exp l op r) = left++" "++op++" "++right
        where left  = if leftNeedParen then "( "++(show l)++" )" else show l
              right = if rightNeedParen then "( "++(show r)++" )" else show r
              leftNeedParen = (leftPrec < opPrec) || ((leftPrec == opPrec) && (rightAssoc exp))
              rightNeedParen = (rightPrec < opPrec) || ((rightPrec == opPrec) && (leftAssoc exp))
              leftPrec  = precedence l
              rightPrec = precedence r
              opPrec    = precedence exp

```

