# Linux

## Shell

- 要想运行一个shell文件，也就是以`.sh`结尾的文件，可以用`sh /**/**/**/**.sh`
  - ![image-20200825201014358](C:\Users\asus\AppData\Roaming\Typora\typora-user-images\image-20200825201014358.png)
- 命令后面可以带参数

### Shell 预定义变量

- $0	当前执行shell程序的名字
- $1-9,${10},${11}.....   参数的位置变量，可以看上图
- $#   一共有多少个参数
- $*   所有位置变量的值
- $?  判断上一条命令是否执行成功，0是成功，非0是失败

### 条件表达式

- -d	测试是否为目录
- -e    测试文件或目录是否存在
- -f     判断是否为文件
- -r    测试当前按用户是否有权限读取
- -w   测试当前用户是否有权限写入
- -x    测试当前用户是否有权限执行
- ![image-20200825212007462](C:\Users\asus\AppData\Roaming\Typora\typora-user-images\image-20200825212007462.png)
  - 注意：[]中的命令与两边的[],有一个空格

### 逻辑表达式

- &&	而且的意思
- ||	   或
- !	    逻辑的否

### 数值比较

- -eq    判断是否等于
- -ne    判断是否不等于
- -gt     判断是否大于
- -lt      判断是否小于
- -le     判断是否等于或着小于
- -ge    判断是否大于或者等于

### 字符串比较

- =    比较字符串是否相等
- !=   比较字符串是否不等
- -z   判断字符串内容是否为空

### 条件测试语句

- 示例
  - ![image-20200826224615420](C:\Users\asus\AppData\Roaming\Typora\typora-user-images\image-20200826224615420.png)
  - 上图的作用是判断是否存在`/media/cdroom`这个目录，如果没有就创建`/media/cdroom`这个目录，**注意，if和[]之间一定要有空格，[]和里面的条件也要有空格**
- ![imag e-20200826230921589](C:\Users\asus\AppData\Roaming\Typora\typora-user-images\image-20200826230921589.png)
- ![image-20200826232207835](C:\Users\asus\AppData\Roaming\Typora\typora-user-images\image-20200826232207835.png)
- ![image-20200827205737605](C:\Users\asus\AppData\Roaming\Typora\typora-user-images\image-20200827205737605.png)
  - 这个有错误，错误原因是 “cat user.text” 改为$(cat user.tex)

### 切换命令行

- `init 3`