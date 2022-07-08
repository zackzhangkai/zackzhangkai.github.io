---
layout: post
published: true
title:  linux之export eval source
categories: [linux]
tags: [export,eval]
---
* content
{:toc}

### eval
eval的作用是再次执行命令行处理，也就是说，对一个命令行，执行两次命令行处理
```
qy_access_key_id="IWXJIAAQLAEFGAERHDWD"
qy_secret_access_key="LfD3gkGm3KjsWeElWHYmPdcsSe3w1OgkW9iE1zY0"
echo "id:" $qy_access_key_id  "key:" $qy_secret_access_key
modify=(qy_access_key_id qy_secret_access_key)
for i in ${modify[@]};do
tmp=`eval echo '$'"$i"`
sed -i "s/^$i:\ .*/$i:\ '$tmp'/g" $1
sed -n "/$i/p" $1
done
```

### export
两个典型场景：
1. 环境变量时，加上这个执行后立即生效
2. 在重定向时有应用，输入输出

用户登录到Linux系统后，系统将启动一个用户shell。在这个shell中，可以使用shell命令

或声明变量，也可以创建并运行shell脚本程序。运行shell脚本程序时，系统将创建一个子shell。

此时，系统中将有两个shell，一个是登录时系统启动的shell，另一个是系统为运行脚本程序创建

的shell。当一个脚本程序运行完毕，脚本shell将终止，返回到执行该脚本之前的shell。



从这种意义上来说，用户可以有许多 shell，每个shell都是由某个shell（称为父shell）派生的。

在子shell中定义的变量只在该子shell内有效。如果在一个shell脚本程序中定义了一个变量，

当该脚本程序运行时，这个定义的变量只是该脚本程序内的一个局部变量，其他的shell不能引用它，

要使某个变量的值可以在其他shell中被改变，可以使用export命令对已定义的变量进行输出。

export命令将使系统在创建每一个新的shell时，定义这个变量的一个拷贝。

这个过程称之为变量输出。



expect 导入的变量在当前shell和子shell中都生效； source 变量只在当前shell生效，在新的shell不生效；

1. 每执行一个脚本都是新开一个shell并在新的shell中执行的过程，脚本执行完毕，变量消失；
2. 如果用expect，子shell会继承父shell的所有变量；export命令将使系统在创建每一个新的shell时，定义这个变量的一个拷贝。

```
export 功能说明：设置或显示环境变量。
语　　法：export [-fnp][变量名称]=[变量设置值]
补充说明：在shell中执行程序时，shell会提供一组环境变量。export可新增，修改或删除环境变量，供后续执行的程序使用。export的效力仅限于该次登陆操作。
参　　数：
　-f 　代表[变量名称]中为函数名称。
　-n 　删除指定的变量。变量实际上并未删除，只是不会输出到后续指令的执行环境中。
　-p 　列出所有的shell赋予程序的环境变量。
```

用source执行后，在shell下是能看到这个变量，但再执行bash开一个子shell时，test是不会

被复制到子shell中的，因为执行脚本文件其实也是在一个子shell中运行，所以我再建另一个脚本

文件执行时，是不会输入任何东西的，内容如：echo $test。所以这点特别注意了，明明在提示符

下可以用echo $test输出变量值，为什么把它放进脚本文件就不行了呢？


　　所以得出的结论是：

1、执行脚本时是在一个子shell环境运行的，脚本执行完后该子shell自动退出；

2、用export定义的变量，一个shell中的系统环境变量会被复制到子shell中；

3、一个shell中的系统环境变量只对该shell或者它的子shell有效，该shell结束时变量消失
（并不能返回到父shell中）。

3、不用export定义的变量只对该shell有效，对子shell也是无效的。


> source导入一个变量文件后，然后在执行的，一定要分清楚是脚本还是仅仅是个命令；如果是个命令变量是可以直接用的；如果是脚本，变量是不可以用的。那现在就有个疑问，既然expect变量可以输出到子shell，而source的变量只在当前shell生效，我们经常的做法是吧expect arg1=value1等语句写进source.sh文件，然后要更换变量的时候直接source这个文件；根据刚才以上的说法，这里是不是有问题？即expect让子shell继承、source不让子shell继承？

其实source的强大在于，它source 一个文件后，这个文件执行不会新起一个子shell的，就在当前Shell里执行完成，这样就会把这个文件里面定义的变量传递到当前shell中。

引申：sh file.sh , ./file.sh 与 source file.sh的区别？

在有执行权限时sh file.sh 与 ./file.sh没有区别；都是起一个新的Shell并来执行  
而source file.sh则是顺序读取脚本内容，在当前shell执行。
