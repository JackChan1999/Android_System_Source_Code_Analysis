### 本文配套视频：

[http://www.toutiao.com/i6432823051523457538/](http://www.toutiao.com/i6432823051523457538/)
[http://www.toutiao.com/i6432826510821818881/](http://www.toutiao.com/i6432826510821818881/)

### 常见Linux命令

Linux命令是对Linux系统进行管理的命令。对于Linux系统来说，无论是中央处理器、内存、磁盘驱动器、键盘、鼠标，还是用户等都是文件。命令就是去操作文件，以达到管理系统的目的。

Linux包含系统内置shell命令（比如cd）及安装的其他Linux命令（比如git）。
相比Window系统的强项-图形化，Linux的命令则是它的强项。熟练掌握常见Linux命令，是做Linux环境下开发的基本技能。

PS：Linux命令大部分在Mac OS 下也是通用的，它们都属于类Unix系统。

#### 3.1目录操作

```
pwd ：获取当前目录路径
ls ：查看文件列表 eg:    ls dirname
    -a:查看所有文件，包括隐藏文件
    -l:查看文件详细信息
    (.):代表当前目录
    (..):代表上一级目录
cd ：进入某一个目录 eg: cd dirname
    cd .. 进入上一级目录
    cd - 进入上次所在目录

mkdir ：创建目录
    -p :创建多级目录  
cp ：拷贝文件或者目录 eg:  cp src1 src2... dest
    -r:拷贝目录     eg: cp -r src dest
mv ：移动文件或者目录 eg:mv src dest
    (如果srcfile和destfile在同一目录下，就是重命名的效果)
rm ：删除文件  eg: rm filename1 filename2 ... 
    -r:删除目录     eg: rm -r dirname
```

#### 3.2文件查找

```
find ：查找某一个文件
    find [path] -name <filename>
grep ：从某一个文件/目录下所有文件 中匹配字符串
    grep [-r -n ]<string>  path
    -r:遍历目录查找
    -n:显示行号
```

#### 3.3系统操作

```
su ：切换到某一个用户(需要输入要切换的用户的密码) eg: su username
sudo :非root用户强制执行一些需要root权限的操作(需要输入当前用户的密码) eg: sudo rm filename  

apt-get : Ubuntu的软件管家，进行软件的更新，卸载与维护。
    sudo apt-get update :更新到最新软件列表
    sudo apt-get install vim :安装VIM
    sudo apt-get remove vim :卸载VIM
vim :终端下的文本编辑器。
    i:进入编辑模式
    ESC：跳到命令模式
    dd:删除一行
    (:wq):保存退出
    (:q!):不保存退出
cat :查看文件,将文本内容打印到控制台。
chmod ：修改文件权限。eg：chmod a+x xxx.sh chmod 777 xxx.sh 
权限设定可以使用字串[ugoa][+-=][rwx]或者数字 (r=4,w=2,x=1,-=0)
u 拥有者，g 用户组，o 其他用户，a 所有人。 
+ 表示增加权限、- 表示取消权限、= 表示直接指定权限
```