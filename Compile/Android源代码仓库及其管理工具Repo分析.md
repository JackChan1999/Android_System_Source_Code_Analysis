软件工程由于需要不断迭代开发，因此要对源代码进行版本管理。[Android](http://lib.csdn.net/base/android)源代码工程（AOSP）也不例外，它采用[Git](http://lib.csdn.net/base/git)来进行版本管理。AOSP作为一个大型开放源代码工程，由许许多多子项目组成，因此不能简单地用[git](http://lib.csdn.net/base/git)进行管理，它在Git的基础上建立了一套自己的代码仓库，并且使用工具Repo进行管理。工欲善其事，必先利其器。本文就对AOSP代码仓库及其管理工具repo进行分析，以便提高我们日常开发效率。

老罗的新浪微博：[http://weibo.com/shengyangluo](http://weibo.com/shengyangluo)，欢迎关注！

《[android](http://lib.csdn.net/base/android)系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

现代的代码版本管理工具，SVN和Git是最流行的。SVN是一种集中式的代码管理工具，需要有一个中心服务器，而Git是一种分布式的代码管理工具。不需要一个中心服务器。不需要中心服务器意味着在没有网络的情况下，Git也能进行版本管理。因此，单从这一点出发，Git就比SVN要方便很多。当然，Git和SVN相比，还有许多不同的理念设计，但是总的来说，Git越来越受到大家的青睐，尤其是在开源社区。君不见，[Linux](http://lib.csdn.net/base/linux)是采用Git进行版本管理，而越来越火的GitHub，提供也是Git代码管理服务。本文不打算分析Git与SVN的区别，以及Git的使用方法，不过强烈建议大家先去了解Git，然后再看下面的内容。这里推荐一本Git书籍《Pro Git》，它是GitHub的员工[Scott Chacon](http://scottchacon.com/about.html)撰写的，对Git的使用及其原理都介绍得非常详细和清晰。

前面提到，AOSP是由许许多项目组成的，例如，在Android 4.2中，就包含了329个项目，每一个项目都是一个独立的Git仓库。这意味着，如果我们要创建一个AOSP分支来做feature开发，那么就需要到每一个子项目去创建对应的分支。这显然不能手动地到每一个子项目里面去创建分支，必须要采用一种自动化的方式来处理。这些自动化处理工作就是由Repo工具来完成的。当然，Repo工具所负责的自动化工作不只是创建分支那么简单，查看分支状态、提交代码、更新代码等基础Git操作它都可以完成。

Repo工具实际上是由一系列的[Python](http://lib.csdn.net/base/python)脚本组成的，这些[python](http://lib.csdn.net/base/python)脚本通过调用Git命令来完成自己的功能。比较有意思的是，组成Repo工具的那些Python脚本本身也是一个Git仓库。这个Git仓库在AOSP里面就称为Repo仓库。我们每次执行Repo命令的时候，Repo仓库都会对自己进行一次更新。

上面我们讨论的是Repo仓库，但是实际上我们执行Repo命令想操作的是AOSP。这就要求Repo命令要知道AOSP都包含有哪些子项目，并且要知道这些子项目的名称、仓库地址是什么。换句话说，就是Repo命令要知道AOSP所有子项目的Git仓库元信息。我们知道，AOSP也是不断地迭代法变化的，例如，它的每一个版本所包含的子项目可能都是不一样的。这意味着需要通过另外一个Git仓库来管理AOSP所有的子项目的Git仓库元信息。这个Git仓库在AOSP里面就称为Manifest仓库。

到目前为止，我们提到了三种类型的Git仓库，分别是Repo仓库、Manifest仓库和AOSP子项目仓库。Repo仓库通过Manifest仓库可以获得所有AOSP子项目仓库的元信息。有了这些元信息之后，我们就可以通过Repo仓库里面的Python脚本来操作AOSP的子项目。那么，Repo仓库和Manifest仓库又是怎么来的呢？答案是通过一个独立的Repo脚本来获取，这个Repo脚本位于AOSP的一个官方网站上，我们可以通过HTTP协议来下载。

现在，我们就通过一个图来来勾勒一下整个AOSP的Picture，它由Repo脚本、Repo仓库、Manifest仓库和AOSP子项目仓库组成，如图1所示：

![img](http://img.blog.csdn.net/20140114004627265?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

图1 Repo脚本、Repo仓库、Manifest仓库和AOSP子项目仓库

接下来我们就看看上述脚本和仓库是怎么来的。

## Repo脚本

从[官方网站](http://source.android.com/source/downloading.html)可以知道，Repo脚本可以通过以下命令来获取：

```
$ curl http://commondatastorage.googleapis.com/git-repo-downloads/repo > ~/bin/repo  
$ chmod a+x ~/bin/repo  
```
也就是可以通过curl工具从http://commondatastorage.googleapis.com/git-repo-downloads/repo获得，并且保存在文件~/bin/repo中。由于~/bin/repo是一个python脚本，我们通过chmod命令赋予它可执行的权限，以便接下来我们可以通过repo命令来运行它。

## Repo仓库

我们下载好Repo脚本之后，要选通过以下命令来安装一个Repo仓库：

```
$ repo init -u https://android.googlesource.com/platform/manifest  
```
这个命令实际上是包含了两个操作：安装Repo仓库和Manifest仓库。其中，Manifest仓库的地址由-u后来带的参数给出。这一小节我们先分析Repo仓库的安装过程，在接下来的第3小节中，再分析Manifest仓库的安装过程。

我们看看Repo脚本是如何执行repo init命令的：

```python
def main(orig_args):  
repo_main, rel_repo_dir = _FindRepo()  
cmd, opt, args = _ParseArguments(orig_args)  

wrapper_path = os.path.abspath(__file__)  
my_main, my_git = _RunSelf(wrapper_path)  

if not repo_main:  
    if opt.help:  
      _Usage()  
    if cmd == 'help':  
      _Help(args)  
    if not cmd:  
      _NotInstalled()  
    if cmd == 'init':  
      if my_git:  
 _SetDefaultsTo(my_git)  
      try:  
 _Init(args)  
      except CloneFailure:  
 ......  
 sys.exit(1)  
      repo_main, rel_repo_dir = _FindRepo()  
    else:  
      _NoCommands(cmd)  

if my_main:  
    repo_main = my_main  

ver_str = '.'.join(map(str, VERSION))  
me = [repo_main,  
 '--repo-dir=%s' % rel_repo_dir,  
 '--wrapper-version=%s' % ver_str,  
 '--wrapper-path=%s' % wrapper_path,  
 '--']  
me.extend(orig_args)  
me.extend(extra_args)  
try:  
    os.execv(repo_main, me)  
except OSError as e:  
    ......  
    sys.exit(148)  

if __name__ == '__main__':  
main(sys.argv[1:])  
```
`_FindRepo`在从当前目录开始往上遍历直到根据目录。如果中间某一个目录下面存在一个.repo/repo目录，并且该.repo/repo存在一个main.py文件，那么就会认为当前是AOSP中执行Repo脚本，这时候它就会返回文件main.py的绝对路径和当前查找目录下的.repo子目录的绝对路径给调用者。在上述情况下，实际上是说明Repo仓库已经安装过了。

`_FindRepo`的实现如下所示：

```python
repodir = '.repo'               # name of repo's private directory  
S_repo = 'repo'                 # special repo repository  
REPO_MAIN = S_repo + '/main.py' # main script  

def _FindRepo():  
"""Look for a repo installation, starting at the current directory. 
"""  
curdir = os.getcwd()  
repo = None  

olddir = None  
while curdir != '/' \  
    and curdir != olddir \  
    and not repo:  
    repo = os.path.join(curdir, repodir, REPO_MAIN)  
    if not os.path.isfile(repo):  
      repo = None  
      olddir = curdir  
      curdir = os.path.dirname(curdir)  
return (repo, os.path.join(curdir, repodir))  
```
`_ParseArguments`无非是对Repo的参数进行解析，得到要执行的命令及其对应的参数。例如，当我们调用`repo init -u https://android.googlesource.com/platform/manifest`的时候，就表示要执行的命令是init，这个命令后面跟的参数是`-u https://android.googlesource.com/platform/manifest`。同时，如果我们调用“repo sync”，就表示要执行的命令是sync，这个命令没有参数。

`_RunSelf`检查Repo脚本所在目录是否存在一个Repo仓库，如果存在的话，就从该仓库克隆一个新的仓库到执行Repo脚本的目录来。实际上就是从本地克隆一个新的Repo仓库。如果不存在的话，那么就需要从远程地址克隆一个Repo仓库到本地来。这个远程地址是Hard Code在Repo脚本里面。

`_RunSelf`的实现如下所示：

```python
def _RunSelf(wrapper_path):  
my_dir = os.path.dirname(wrapper_path)  
my_main = os.path.join(my_dir, 'main.py')  
my_git = os.path.join(my_dir, '.git')  

if os.path.isfile(my_main) and os.path.isdir(my_git):  
    for name in ['git_config.py',  
          'project.py',  
          'subcmds']:  
      if not os.path.exists(os.path.join(my_dir, name)):  
 return None, None  
    return my_main, my_git  
return None, None  
```
从这里我们就可以看出，如果Repo脚本所在的目录存在一个Repo仓库，那么要满足以下条件：

(1). 存在一个.git目录

(2). 存在一个main.py文件

(3). 存在一个git_config.py文件

(4). 存在一个project.py文件

(5). 存在一个subcmds目录

上述目录和文件实际上都是一个标准的Repo仓库所具有的。

我们再回到main函数中。如果调用`_FindRepo`得到的repo`_main`的值等于空，那么就说明当前目录还没有安装Repo仓库，这时候Repo后面所跟的参数只能是help或者init，否则的话，就会显示错误信息。如果Repo后面跟的参数是help，就打印出Repo脚本的帮助文档。

我们关注Repo后面跟的参数是init的情况。这时候看一下调用`_RunSelf`的返回值`my_git`是否不等于空。如果不等于空的话，那么就说明Repo脚本所在目录存一个Repo仓库，这时候就调用`_SetDefaultsTo`设置等一会要克隆的Repo仓库源。

`_SetDefaultsTo`的实现如下所示：

```python
GIT = 'git'  

REPO_URL = 'https://gerrit.googlesource.com/git-repo'  
REPO_REV = 'stable'  

def _SetDefaultsTo(gitdir):  
global REPO_URL  
global REPO_REV  

REPO_URL = gitdir  
proc = subprocess.Popen([GIT,  
                    '--git-dir=%s' % gitdir,  
                    'symbolic-ref',  
                    'HEAD'],  
                   stdout = subprocess.PIPE,  
                   stderr = subprocess.PIPE)  
REPO_REV = proc.stdout.read().strip()  
proc.stdout.close()  

proc.stderr.read()  
proc.stderr.close()  

if proc.wait() != 0:  
    _print('fatal: %s has no current branch' % gitdir, file=sys.stderr)  
    sys.exit(1)  
```
如果Repo脚本所在目录不存在一个Repo仓库，那么默认就将https://gerrit.googlesource.com/git-repo设置为一会要克隆的Repo仓库源，并且要克隆的分支是stable。否则的话，就以该Repo仓库作为克隆源，并且以该Repo仓库的当前分支作为要克隆的分支。

从前面的分析就可以知道，当我们执行"repo init"命令的时候：

(1). 如果我们只是从网上下载了一个Repo脚本，那么在执行Repo命令的时候，就会从远程克隆一个Repo仓库到当前执行Repo脚本的目录来。

(2). 如果我们从网上下载的是一个带有Repo仓库的Repo脚本，那么在执行Repo命令的时候，就可以从本地克隆一个Repo仓库到当前执行Repo脚本的目录来。

我们再继续看main函数的实现，它接下来调用`_Init`在当前执行Repo脚本的目录下安装一个Repo仓库：

```python
def _Init(args):  
"""Installs repo by cloning it over the network. 
"""  
opt, args = init_optparse.parse_args(args)  
......  

url = opt.repo_url  
if not url:  
    url = REPO_URL  
    extra_args.append('--repo-url=%s' % url)  

branch = opt.repo_branch  
if not branch:  
    branch = REPO_REV  
    extra_args.append('--repo-branch=%s' % branch)  

......  

if not os.path.isdir(repodir):  
    try:  
      os.mkdir(repodir)  
    except OSError as e:  
      ......  
      sys.exit(1)  

_CheckGitVersion()  
try:  
    if NeedSetupGnuPG():  
      can_verify = SetupGnuPG(opt.quiet)  
    else:  
      can_verify = True  

    dst = os.path.abspath(os.path.join(repodir, S_repo))  
    _Clone(url, dst, opt.quiet)  

    if can_verify and not opt.no_repo_verify:  
      rev = _Verify(dst, branch, opt.quiet)  
    else:  
      rev = 'refs/remotes/origin/%s^0' % branch  

    _Checkout(dst, branch, rev, opt.quiet)  
except CloneFailure:  
    ......  
```
如果我们在执行Repo脚本的时候，没有通过--repo-url和--repo-branch来指定Repo仓库的源地址和分支，那么就使用由REPO_URL和REPO_REV所指定的源地址和分支。从前面的分析可以知道，REPO_URL和REPO_REV要么指向远程地址`https://gerrit.googlesource.com/git-repo`和分支stable，要么指向Repo脚本所在目录的Repo仓库和该仓库的当前分支。

`_Init`函数继续检查当前执行Repo脚本的目录是否存在一个.repo目录。如果不存在的话，那么就新创建一个。接着是否需要验证等一会克隆回来的Repo仓库的GPG。如果需要验证的话，那么就会在调用`_Clone`函数来克隆好Repo仓库之后，调用`_Verify`函数来验证该Repo仓库的GPG。

最关键的地方就在于函数`_Clone`函数和`_Checkout`函数的调用，前者用来从url指定的地方克隆一个仓库到dst指定的地方来，实际上就是克隆到当前目录下的./repo/repo目录来，后者在克隆回来的仓库中将branch分支checkout出来。

函数`_Clone`的实现如下所示：

```python
def _Clone(url, local, quiet):  
"""Clones a git repository to a new subdirectory of repodir 
"""  
try:  
    os.mkdir(local)  
except OSError as e:  
    _print('fatal: cannot make %s directory: %s' % (local, e.strerror),  
    file=sys.stderr)  
    raise CloneFailure()  

cmd = [GIT, 'init', '--quiet']  
try:  
    proc = subprocess.Popen(cmd, cwd = local)  
except OSError as e:  
    ......  

......  

_InitHttp()  
_SetConfig(local, 'remote.origin.url', url)  
_SetConfig(local, 'remote.origin.fetch',  
             '+refs/heads/*:refs/remotes/origin/*')  
if _DownloadBundle(url, local, quiet):  
    _ImportBundle(local)  
else:  
    _Fetch(url, local, 'origin', quiet)  
```
这个函数首先是调用"git init"在当前目录下的.repo/repo子目录初始化一个Git仓库，接着再调用`_SetConfig`函来设置该Git仓库的url信息等。再接着调用`_DownloadBundle`来检查指定的url是否存在一个clone.bundle文件。如果存在这个clone.bundle文件的话，就以它作为Repo仓库的克隆源。

函数`_DownloadBundle`的实现如下所示：

```python
def _DownloadBundle(url, local, quiet):  
if not url.endswith('/'):  
    url += '/'  
url += 'clone.bundle'  

......  

if not url.startswith('http:') and not url.startswith('https:'):  
    return False  

dest = open(os.path.join(local, '.git', 'clone.bundle'), 'w+b')  
try:  
    try:  
      r = urllib.request.urlopen(url)  
    except urllib.error.HTTPError as e:  
      if e.code in [403, 404]:  
 return False  
      ......  
      raise CloneFailure()  
    except urllib.error.URLError as e:  
      ......  
      raise CloneFailure()  
    try:  
      if not quiet:  
 _print('Get %s' % url, file=sys.stderr)  
      while True:  
 buf = r.read(8192)  
 if buf == '':  
   return True  
 dest.write(buf)  
    finally:  
      r.close()  
finally:  
    dest.close()  
```
Bundle文件是Git提供的一种机制，用来解决不能正常通过git、ssh和http等网络协议从远程地址克隆Git仓库的问题。简单来说，就是我们可以用“git bundle”命令来在一个Git仓库创建一个Bundle文件，这个Bundle文件就会包含Git仓库的提交历史。把这个Bundle文件通过其它方式拷贝到另一台机器上，就可以将它作为一个本地Git仓库来使用，而不用去访问远程网络。

回到函数`_Clone`中，如果存在对应的clone.bundle文件，并且能成功下载到该clone.bundle，接下来就调用函数`_ImportBundle`将它作为源仓库克隆为新的Repo仓库。函数`_ImportBundle`的实现如下所示：

```python
def _ImportBundle(local):  
path = os.path.join(local, '.git', 'clone.bundle')  
try:  
    _Fetch(local, local, path, True)  
finally:  
    os.remove(path)  
```
结合`_Clone`函数和`_ImportBundle`函数就可以看出，从clone.bundle文件克隆Repo仓库和从远程url克隆Repo仓库都是通过函数`_Fetch`来实现的。区别就在于调用函数`_Fetch`时指定的第三个参数，前者是已经下载到本地的一个clone.bundle文件路径，后者是origin（表示远程仓库名称）。

函数`_Fetch`的实现如下所示：

```python
def _Fetch(url, local, src, quiet):  
if not quiet:  
    _print('Get %s' % url, file=sys.stderr)  

cmd = [GIT, 'fetch']  
if quiet:  
    cmd.append('--quiet')  
    err = subprocess.PIPE  
else:  
    err = None  
cmd.append(src)  
cmd.append('+refs/heads/*:refs/remotes/origin/*')  
cmd.append('refs/tags/*:refs/tags/*')  

proc = subprocess.Popen(cmd, cwd = local, stderr = err)  
if err:  
    proc.stderr.read()  
    proc.stderr.close()  
if proc.wait() != 0:  
    raise CloneFailure()  
```
函数`_Fetch`实际上就是通过“git fetch”从指定的仓库源克隆一个新的Repo仓库到当前目录下的.repo/repo子目录来。

注意，以上只是克隆好了一个Repo仓库，接下来还需要从这个Repo仓库checkout出一个分支来，才能正常工作。从Repo仓库checkout出一个分支是通过调用函数`_Checkout`来实现的：

```python
def _Checkout(cwd, branch, rev, quiet):  
"""Checkout an upstream branch into the repository and track it. 
"""  
cmd = [GIT, 'update-ref', 'refs/heads/default', rev]  
if subprocess.Popen(cmd, cwd = cwd).wait() != 0:  
    raise CloneFailure()  

_SetConfig(cwd, 'branch.default.remote', 'origin')  
_SetConfig(cwd, 'branch.default.merge', 'refs/heads/%s' % branch)  

cmd = [GIT, 'symbolic-ref', 'HEAD', 'refs/heads/default']  
if subprocess.Popen(cmd, cwd = cwd).wait() != 0:  
    raise CloneFailure()  

cmd = [GIT, 'read-tree', '--reset', '-u']  
if not quiet:  
    cmd.append('-v')  
cmd.append('HEAD')  
if subprocess.Popen(cmd, cwd = cwd).wait() != 0:  
    raise CloneFailure()  
```
要checkout出来的分支由参数branch指定。从前面的分析可以知道，如果当前执行的Repo脚本所在目录存在一个Repo仓库，那么参数branch描述的就是该仓库当前checkout出来的分支。否则的话，参数branch描述的就是从远程仓库克隆回来的“stable”分支。

需要注意的是，这里从仓库checkout出分支不是使用“git checkout”命令来实现的，而是通过更底层的Git命令“git update-ref”来实现的。实际上，“git checkout”命令也是通过“git update-ref”命令来实现的，只不过它进行了更高层的封装，更方便使用。如果我们去分析组成Repo仓库的那些Python脚本命令，就会发现它们基本上都是通过底层的Git命令来完成Git功能的。

## Manifest仓库

我们接着再分析下面这个命令的执行：

```
repo init -u https://android.googlesource.com/platform/manifest  
```
如前所述，这个命令安装好Repo仓库之后，就会调用该Repo仓库下面的main.py脚本，对应的文件为.repo/repo/main.py，它的入口函数的实现如下所示：

```python
def _Main(argv):  
result = 0  

opt = optparse.OptionParser(usage="repo wrapperinfo -- ...")  
opt.add_option("--repo-dir", dest="repodir",  
          help="path to .repo/")  
......  

repo = _Repo(opt.repodir)  
try:  
    try:  
      init_ssh()  
      init_http()  
      result = repo._Run(argv) or 0  
    finally:  
      close_ssh()  
except KeyboardInterrupt:  
    ......  
    result = 1  
except ManifestParseError as mpe:  
    ......  
    result = 1  
except RepoChangedException as rce:  
    # If repo changed, re-exec ourselves.  
    #  
    argv = list(sys.argv)  
    argv.extend(rce.extra_args)  
    try:  
      os.execv(__file__, argv)  
    except OSError as e:  
      ......  
      result = 128  

sys.exit(result)  

if __name__ == '__main__':  
_Main(sys.argv[1:])  
```
从前面的分析可以知道，通过参数--repo-dir传进来的是AOSP根目录下的.repo目录，这是一个隐藏目录，里面保存的是Repo仓库、Manifest仓库，以及各个AOSP子项目仓库。函数`_Main`首先是调用init_ssh和init_http来初始化网络环境，接着再调用前面创建的一个`_Repo`对象的成员函数`_Run`来解析要执行的命令，并且执行这个命令。

`_Repo`类的成员函数`_Run`的实现如下所示：

```python
from subcmds import all_commands  

class _Repo(object):  
def __init__(self, repodir):  
    self.repodir = repodir  
    self.commands = all_commands  
    # add 'branch' as an alias for 'branches'  
    all_commands['branch'] = all_commands['branches']  

def _Run(self, argv):  
    result = 0  
    name = None  
    glob = []  

    for i in range(len(argv)):  
      if not argv[i].startswith('-'):  
 name = argv[i]  
 if i > 0:  
   glob = argv[:i]  
 argv = argv[i + 1:]  
 break  
    if not name:  
      glob = argv  
      name = 'help'  
      argv = []  
    gopts, _gargs = global_options.parse_args(glob)  

    ......  

    try:  
      cmd = self.commands[name]  
    except KeyError:  
      ......  
      return 1  

    cmd.repodir = self.repodir  
    cmd.manifest = XmlManifest(cmd.repodir)  

    ......  

    try:  
      result = cmd.Execute(copts, cargs)  
    except DownloadError as e:  
      ......  
      result = 1  
    except ManifestInvalidRevisionError as e:  
      ......  
      result = 1  
    except NoManifestException as e:  
      ......  
      result = 1  
    except NoSuchProjectError as e:  
      ......  
      result = 1  
    finally:  
      ......  

    return result  
```
Repo脚本能执行的命令都放在目录.repo/repo/subcmds中，该目录每一个python文件都对应一个Repo命令。例如，“repo init”表示要执行命令脚本是.repo/repo/subcmds/init.py。

`_Repo`类的成员函数`_Run`首先是在repo后面所带的参数中，不是以横线“-”开始的第一个选项，该选项就代表要执行的命令，该命令的名称就保存在变量name中。接着根据变量name的值在`_Repo`类的成员变量commands中找到对应的命令模块cmd，并且指定该命令模块cmd的成员变量repodir和manifest的值。命令模块cmd的成员变量repodir描述的就是AOSP的.repo目录，成员变量manifest指向的是一个XmlManifest对象，它描述的是AOSP的Repo仓库和Manifest仓库。

我们看看XmlManifest类的构造函数，它定义在文件.repo/repo/xml_manifest.py文件中：

```python
class XmlManifest(object):  
"""manages the repo configuration file"""  

def __init__(self, repodir):  
    self.repodir = os.path.abspath(repodir)  
    self.topdir = os.path.dirname(self.repodir)  
    self.manifestFile = os.path.join(self.repodir, MANIFEST_FILE_NAME)  
    ......  

    self.repoProject = MetaProject(self, 'repo',  
      gitdir   = os.path.join(repodir, 'repo/.git'),  
      worktree = os.path.join(repodir, 'repo'))  

    self.manifestProject = MetaProject(self, 'manifests',  
      gitdir   = os.path.join(repodir, 'manifests.git'),  
      worktree = os.path.join(repodir, 'manifests'))  

    ......  
```
XmlManifest作了描述了AOSP的Repo目录（repodir）、AOSP 根目录（topdir）和Manifest.xml文件（manifestFile）之外，还使用两个MetaProject对象描述了AOSP的Repo仓库（repoProject）和Manifest仓库（manifestProject）。

在AOSP中，每一个子项目（或者说仓库）都用一个Project对象来描述。Project类定义在文件.repo/repo/project.py文件中，用来封装对各个项目的基础Git操作，例如，对项目进行暂存、提交和更新等。它的构造函数如下所示：

```python
class Project(object):  
def __init__(self,  
        manifest,  
        name,  
        remote,  
        gitdir,  
        worktree,  
        relpath,  
        revisionExpr,  
        revisionId,  
        rebase = True,  
        groups = None,  
        sync_c = False,  
        sync_s = False,  
        clone_depth = None,  
        upstream = None,  
        parent = None,  
        is_derived = False,  
        dest_branch = None):  
    """Init a Project object. 
    Args: 
      manifest: The XmlManifest object. 
      name: The `name` attribute of manifest.xml's project element. 
      remote: RemoteSpec object specifying its remote's properties. 
      gitdir: Absolute path of git directory. 
      worktree: Absolute path of git working tree. 
      relpath: Relative path of git working tree to repo's top directory. 
      revisionExpr: The `revision` attribute of manifest.xml's project element. 
      revisionId: git commit id for checking out. 
      rebase: The `rebase` attribute of manifest.xml's project element. 
      groups: The `groups` attribute of manifest.xml's project element. 
      sync_c: The `sync-c` attribute of manifest.xml's project element. 
      sync_s: The `sync-s` attribute of manifest.xml's project element. 
      upstream: The `upstream` attribute of manifest.xml's project element. 
      parent: The parent Project object. 
      is_derived: False if the project was explicitly defined in the manifest; 
           True if the project is a discovered submodule. 
      dest_branch: The branch to which to push changes for review by default. 
    """  
    self.manifest = manifest  
    self.name = name  
    self.remote = remote  
    self.gitdir = gitdir.replace('\\', '/')  
    if worktree:  
      self.worktree = worktree.replace('\\', '/')  
    else:  
      self.worktree = None  
    self.relpath = relpath  
    self.revisionExpr = revisionExpr  

    if   revisionId is None \  
     and revisionExpr \  
     and IsId(revisionExpr):  
      self.revisionId = revisionExpr  
    else:  
      self.revisionId = revisionId  

    self.rebase = rebase  
    self.groups = groups  
    self.sync_c = sync_c  
    self.sync_s = sync_s  
    self.clone_depth = clone_depth  
    self.upstream = upstream  
    self.parent = parent  
    self.is_derived = is_derived  
    self.subprojects = []  

    self.snapshots = {}  
    self.copyfiles = []  
    self.annotations = []  
    self.config = GitConfig.ForRepository(  
             gitdir = self.gitdir,  
             defaults =  self.manifest.globalConfig)  

    if self.worktree:  
      self.work_git = self._GitGetByExec(self, bare=False)  
    else:  
      self.work_git = None  
    self.bare_git = self._GitGetByExec(self, bare=True)  
    self.bare_ref = GitRefs(gitdir)  
    self.dest_branch = dest_branch  

    # This will be filled in if a project is later identified to be the  
    # project containing repo hooks.  
    self.enabled_repo_hooks = []  
```
Project类构造函数的各个参数的含义见注释，这里为了方便描述，用中文描述一下：

| 参数          | 含义                                       |
| :---------- | :--------------------------------------- |
| manifest    | 指向一个XmlManifest对象，描述AOSP的Repo仓库和Manifest仓库元信息 |
| name        | 项目名称                                     |
| remote      | 描述项目对应的远程仓库元信息                           |
| gitdir      | 项目的Git仓库目录                               |
| worktree    | 项目的工作目录                                  |
| relpath     | 项目的相对于AOSP根目录的工作目录                       |
| parent      | 父项目                                      |
| is_derived  | 如果一个项目含有子模块（也是一个Git仓库），那么这些子模块也会用一个Project对象来描述，这些Project的is_derived属性会设置为true |
| dest_branch | 用来code review的分支                         |

**revisionExpr、revisionId、rebase、groups、sync_c、sync_s和upstream**：每一个项目在.repo/repo/manifest.xml文件中都有对应的描述，这几个属性的值就来自于该manifest.xml文件对自己的描述的，它们的含义可以参考.repo/repo/docs/manifest-format.txt文件

这里重点说一下项目的Git仓库目录和工作目录的概念。一般来说，一个项目的Git仓库目录（默认为.git目录）是位于工作目录下面的，但是Git支持将一个项目的Git仓库目录和工作目录分开来存放。在AOSP中，Repo仓库的Git目录（.git）位于工作目录（.repo/repo）下，Manifest仓库的Git目录有两份拷贝，一份（.git）位于工作目录（.repo/manifests）下，另外一份位于.repo/manifests.git目录，其余的AOSP子项目的工作目录和Git目录都是分开存放的，其中，工作目录位于AOSP根目录下，Git目录位于.repo/repo/projects目录下。

此外，每一个AOSP子项目的工作目录也有一个.git目录，不过这个.git目录是一个符号链接，链接到.repo/repo/projects对应的Git目录。这样，我们就既可以在AOSP子项目的工作目录下执行Git命令，也可以在其对应的Git目录下执行Git命令。一般来说，要访问到工作目录的命令（例如git status）需要在工作目录下执行，而不需要访问工作目录（例如git log）可以在Git目录下执行。

Project类有两个成员变量work_git和bare_git，它们指向的都是一个_GitGetByExec对象。用来封装对Git命令的执行。其中，前者在执行Git命令的时候，会将当前目录设置为项目的工作目录，而后者在执行的时候，不会设置当前目录，但是会将环境变量GIT_DIR的值设置为项目的Git目录，也就是.repo/projects目录下面的那些目录。通过这种方式，Project类就可以根据需要来在工作目录或者Git目录下执行Git命令。

回到XmlManifest类的构造函数中，由于Repo和Manifest也是属于Git仓库，所以我们也需要创建一个Project对象来描述它们。不过，由于它们是比较特殊的Git仓库（用来描述AOSP子项目元信息的Git仓库），所以我们就使用另外一个类型为MetaProject的对象来描述它们。MetaProject类是从Project类继承下来的，定义在project.py文件中，如下所示：

```python
class MetaProject(Project):  
"""A special project housed under .repo. 
"""  
def __init__(self, manifest, name, gitdir, worktree):  
    Project.__init__(self,  
              manifest = manifest,  
              name = name,  
              gitdir = gitdir,  
              worktree = worktree,  
              remote = RemoteSpec('origin'),  
              relpath = '.repo/%s' % name,  
              revisionExpr = 'refs/heads/master',  
              revisionId = None,  
              groups = None)  
```
既然MetaProject类是从Project类继承下来的，那么它们的Git操作几乎都可以通过Project类来完成的。实际上，MetaProject类和Project类目前的区别不是太大，可以认为是基本相同的。使用MetaProject类来描述Repo仓库和Manifest仓库，主要是为了强调它们是用来描述AOSP子项目仓库的元信息的。

回到`_Repo`类的成员函数`_Run`中，创建好用来描述Repo仓库和Manifest仓库的XmlManifest对象之后，就开始执行跟在repo脚本后面的不带横线“-”的选项所表示的命令。在我们这个场景中，这个命令就是init，它对应的Python模块为.repo/repo/subcmds/init.py，入口函数为定义在该模块的Init类的成员函数Execute，它的实现如下所示：

```python
class Init(InteractiveCommand, MirrorSafeCommand):  
    ......  

def Execute(self, opt, args):  
    ......  

    self._SyncManifest(opt)  
    self._LinkManifest(opt.manifest_name)  

    ......  
```
Init类的成员函数Execute主要就是调用另外两个成员函数_SyncManifest和_LinkManifest来完成克隆Manifest仓库的工作。

Init类的成员函数_SyncManifest的实现如下所示：

```python
class Init(InteractiveCommand, MirrorSafeCommand):  
......  

def _SyncManifest(self, opt):  
    m = self.manifest.manifestProject  
    is_new = not m.Exists  

    if is_new:  
 ......  

      m._InitGitDir(mirror_git=mirrored_manifest_git)  

      if opt.manifest_branch:  
 m.revisionExpr = opt.manifest_branch  
      else:  
 m.revisionExpr = 'refs/heads/master  
    else:  
      if opt.manifest_branch:  
 m.revisionExpr = opt.manifest_branch  
      else:  
 m.PreSync()  

    ......  

    if not m.Sync_NetworkHalf(is_new=is_new):  
      ......  
      sys.exit(1)  

    if opt.manifest_branch:  
      m.MetaBranchSwitch(opt.manifest_branch)  
    ......  

    m.Sync_LocalHalf(syncbuf)  
    ......  

    if is_new or m.CurrentBranch is None:  
      if not m.StartBranch('default'):  
 ......  
 sys.exit(1)  
```
Init类的成员函数 `_SyncManifest` 执行以下操作：

(1). 检查本地是否存在Manifest仓库，即检查用来描述Manifest仓库MetaProject对象m的成员变量mExists值是否等于true。如果不等于的话，那么就说明本地还没有安装过Manifest仓库。这时候就需要调用该MetaProject对象m的成员函数 `_InitGitDir` 来在.repo/manifests目录初始化一个Git仓库。

(2). 调用用来描述Manifest仓库MetaProject对象m的成员函数Sync_NetworkHalf来从远程仓库中克隆一个新的Manifest仓库到本地来，或者更新本地的Manifest仓库。这个远程仓库的地址即为在执行"repo init"命令时，通过-u指定的url，即`https://android.googlesource.com/platform/manifest`。

(3). 检查"repo init"命令后面是否通过-b指定要在Manifest仓库中checkout出来的分支。如果有的话，那么就调用用来描述Manifest仓库MetaProject对象m的成员函数MetaBranchSwitch做一些清理工作，以便接下来可以checkout到指定的分支。

(4). 调用用来描述Manifest仓库MetaProject对象m的成员函数Sync_LocaHalf来执行checkout分支的操作。注意，要切换的分支在前面已经记录在MetaProject对象m的成员变量revisionExpr中。

(5). 如果前面执行的是新安装Manifest仓库的操作，并且没有通过-b选项指定要checkout的分支，那么默认就checkout出一个default分支。

接下来，我们就主要分析MetaProject类的成员函数 `_InitGitDir` 、Sync_NetworkHalf和Sync_LocaHalf的实现。这几个函数实际上都是由MetaProject的父类Project来实现的，因此，下面我们就分析Project类的成员函数 `_InitGitDir` 、Sync_NetworkHalf和Sync_LocaHalf的实现。

Project类的成员函数 `_InitGitDir` 的成员函数的实现如下所示：

```python
class Project(object):  
......  

def _InitGitDir(self, mirror_git=None):  
    if not os.path.exists(self.gitdir):  
      os.makedirs(self.gitdir)  
      self.bare_git.init()  
      ......  
```
Project类的成员函数 `_InitGitDir` 首先是检查项目的Git目录是否已经存在。如果不存在，那么就会首先创建这个Git目录，然后再调用成员变量bare_git所描述的一个_GitGetByExec对象的成员函数init来在该目录下初始化一个Git仓库。

 `_GitGetByExec` 类的成员函数init是通过另外一个成员函数__getattr__来实现的，如下所示：

```python
class Project(object):  
......  

class _GitGetByExec(object):  
    ......  

    def __getattr__(self, name):  
      """Allow arbitrary git commands using pythonic syntax. 

      This allows you to do things like: 
 git_obj.rev_parse('HEAD') 

      Since we don't have a 'rev_parse' method defined, the __getattr__ will 
      run.  We'll replace the '_' with a '-' and try to run a git command. 
      Any other positional arguments will be passed to the git command, and the 
      following keyword arguments are supported: 
 config: An optional dict of git config options to be passed with '-c'. 

      Args: 
 name: The name of the git command to call.  Any '_' characters will 
     be replaced with '-'. 

      Returns: 
 A callable object that will try to call git with the named command. 
      """  
      name = name.replace('_', '-')  
      def runner(*args, **kwargs):  
 cmdv = []  
 config = kwargs.pop('config', None)  
 ......  
 if config is not None:  
   ......  
   for k, v in config.items():  
     cmdv.append('-c')  
     cmdv.append('%s=%s' % (k, v))  
 cmdv.append(name)  
 cmdv.extend(args)  
 p = GitCommand(self._project,  
                cmdv,  
                bare = self._bare,  
                capture_stdout = True,  
                capture_stderr = True)  
 if p.Wait() != 0:  
   ......  
 r = p.stdout  
 try:  
   r = r.decode('utf-8')  
 except AttributeError:  
   pass  
 if r.endswith('\n') and r.index('\n') == len(r) - 1:  
   return r[:-1]  
 return r  
      return runner  
```
从注释可以知道， `_GitGetByExec` 类的成员函数__getattr__使用了一个trick，将 `_GitGetByExec` 类没有实现的成员函数间接地以属性的形式来获得，并且将该没有实现的成员函数的名称作为git的一个参数来执行。也就是说，当执行_GitGetByExec.init()的时候，实际上是透过成员函数__getattr__执行了一个"git init"命令。这个命令就正好是用来初始化一个Git仓库。

我们再来看Project类的成员函数Sync_NetworkHalf的实现：

```python
class Project(object):  
......  

def Sync_NetworkHalf(self,  
      quiet=False,  
      is_new=None,  
      current_branch_only=False,  
      clone_bundle=True,  
      no_tags=False):  
    """Perform only the network IO portion of the sync process. 
Local working directory/branch state is not affected. 
    """  
    if is_new is None:  
      is_new = not self.Exists  
    if is_new:  
      self._InitGitDir()  

    ......  

    if not self._RemoteFetch(initial=is_new, quiet=quiet, alt_dir=alt_dir,  
                      current_branch_only=current_branch_only,  
                      no_tags=no_tags):  
      return False  

    ......  
```
Project类的成员函数Sync_NetworkHalf主要执行以下的操作：

(1). 检查本地是否已经存在对应的Git仓库。如果不存在，那么就先调用另外一个成员函数 `_InitGitDir` 来初始化该Git仓库。

(2). 调用另外一个成员函籹 `_RemoteFetch` 来从远程仓库更新本地仓库。

Project类的成员函数 `_RemoteFetch` 的实现如下所示：

```python
class Project(object):  
......  

def _RemoteFetch(self, name=None,  
            current_branch_only=False,  
            initial=False,  
            quiet=False,  
            alt_dir=None,  
            no_tags=False):  
    ......  

    cmd = ['fetch']  

    ......  

    ok = False  
    for _i in range(2):  
      ret = GitCommand(self, cmd, bare=True, ssh_proxy=ssh_proxy).Wait()  
      if ret == 0:  
 ok = True  
 break  
      elif current_branch_only and is_sha1 and ret == 128:  
 # Exit code 128 means "couldn't find the ref you asked for"; if we're in sha1  
 # mode, we just tried sync'ing from the upstream field; it doesn't exist, thus  
 # abort the optimization attempt and do a full sync.  
 break  
      time.sleep(random.randint(30, 45))  
      
    ......  
```
Project类的成员函数 `_RemoteFetch` 的核心操作就是调用“git fetch”命令来从远程仓库更新本地仓库。

接下来我们再看MetaProject类的成员函数Sync_LocaHalf的实现：

```python
class Project(object):  
......  

def Sync_LocalHalf(self, syncbuf):  
    """Perform only the local IO portion of the sync process. 
Network access is not required. 
    """  
    ......  

    revid = self.GetRevisionId(all_refs)  

    ......  

    self._InitWorkTree()  
    head = self.work_git.GetHead()  
    if head.startswith(R_HEADS):  
      branch = head[len(R_HEADS):]  
      try:  
 head = all_refs[head]  
      except KeyError:  
 head = None  
    else:  
      branch = None  

    ......  

    if head == revid:  
      # No changes; don't do anything further.  
      #  
      return  

    branch = self.GetBranch(branch)  

    ......  

    if not branch.LocalMerge:  
      # The current branch has no tracking configuration.  
      # Jump off it to a detached HEAD.  
      #  
      syncbuf.info(self,  
            "leaving %s; does not track upstream",  
            branch.name)  
      try:  
 self._Checkout(revid, quiet=True)  
      except GitError as e:  
 syncbuf.fail(self, e)  
 return  
      ......  
      return  

    ......  
```
这里我们只分析一种比较简单的情况，就是当前要checkout的分支是一个干净的分支，它没有做过修改，也没有设置跟踪远程分支。这时候Project类的成员函数 `_RemoteFetch` 的主要执行以下操作：

(1). 调用另外一个成员函数GetRevisionId获得即将要checkout的分支，保存在变量revid中。

(2). 调用成员变量work_git所描述的一个 `_GitGetByExec` 对象的成员函数GetHead获得项目当前checkout的分支，只存在变量head中。

(3). 如果即将要checkout的分支revid就是当前已经checkout分支，那么就什么也不用做。否则继续往下执行。

(4). 调用另外一个成员函数GetBranch获得用来描述当前分支的一个Branch对象。

(5). 如果上述Branch对象的属性LocalMerge的值等于None，也就是属于我们讨论的情况，那么就调用另外一个成员函数 `_Checkout` 真正执行checkout分支revid的操作。

如果要checkout的分支revid不是一个干净的分支，也就是它正在跳踪远程分支，并且在本地做过提交，这些提交又没有上传到远程分支去，那么就需要执行一些merge或者rebase的操作。不过无论如何，这些操作都是通过标准的Git命令来完成的。

我们接着再看Project类的成员函数 `_Checkout` 的实现：

```python
class Project(object):  
......  

def _Checkout(self, rev, quiet=False):  
    cmd = ['checkout']  
    if quiet:  
      cmd.append('-q')  
    cmd.append(rev)  
    cmd.append('--')  
    if GitCommand(self, cmd).Wait() != 0:  
      if self._allrefs:  
 raise GitError('%s checkout %s ' % (self.name, rev))  
```
Project类的成员函数_Checkout的实现很简单，它通过GitCommand类构造了一个“git checkout”命令，将参数rev描述的分支checkout出来。

至此，我们就将Manifest仓库从远程地址https://android.googlesource.com/platform/manifest克隆到本地来了，并且checkout出了指定的分支。回到Init类的成员函数Execute中，它接下来还要调用另外一个成员函数 `_LinkManifest` 来执行一个符号链接的操作。

Init类的成员函数 `_LinkManifest` 的实现如下所示：

```python
class Init(InteractiveCommand, MirrorSafeCommand):  
......  

def _LinkManifest(self, name):  
    if not name:  
      print('fatal: manifest name (-m) is required.', file=sys.stderr)  
      sys.exit(1)  

    try:  
      self.manifest.Link(name)  
    except ManifestParseError as e:  
      print("fatal: manifest '%s' not available" % name, file=sys.stderr)  
      print('fatal: %s' % str(e), file=sys.stderr)  
      sys.exit(1)  
```
参数name的值一般就等于“default.xml”，表示Manifest仓库中的default.xml文件，Init类的成员函数 `_LinkManifest` 通过调用成员变量manifest所描述的一个XmlManifest对象的成员函数Link来执行符号链接的操作，它定义在文件.repo/repo/xml_manifest.py文件，它的实现如下所示：

```python
class XmlManifest(object):  
"""manages the repo configuration file"""  
......  

def Link(self, name):  
    """Update the repo metadata to use a different manifest. 
    """  
    ......  

    try:  
      if os.path.lexists(self.manifestFile):  
 os.remove(self.manifestFile)  
      os.symlink('manifests/%s' % name, self.manifestFile)  
    except OSError as e:  
      raise ManifestParseError('cannot link manifest %s: %s' % (name, str(e)))
```

XmlManifest类的成员变量manifestFile的值等于$(AOSP)/.repo/manifest.xml，通过调用os.symlink就将它符号链接至$(AOSP)/.repo/manifests/<name>文件去。这样无论Manifest仓库中用来描述AOSP子项目的xml文件是什么名称，都可以统一通过$(AOSP)/.repo/manifest.xml文件来访问。

前面提到，Manifest仓库中用来描述AOSP子项目的xml文件名称默认就为default.xml，它的内容如下所示：

```xml
<?xml version="1.0" encoding="UTF-8"?>  
<manifest>  

<remote  name="aosp"  
    fetch=".."  
    review="https://android-review.googlesource.com/" />  
<default revision="refs/tags/android-4.2_r1"  
    remote="aosp"  
    sync-j="4" />  

<project path="build" name="platform/build" >  
    <copyfile src="core/root.mk" dest="Makefile" />  
</project>  
<project path="abi/cpp" name="platform/abi/cpp" />  
<project path="bionic" name="platform/bionic" />  
......  

</manifest>  
```
关于该xml文件的详细描述可以参考.repo/repo/docs/manifest-format.txt文件。一般来说，该xml包含有四种类型的标签：

**remote**：用来指定远程仓库信息。属性name描述的是一个远程仓库的名称，属性fetch用作项目名称的前缘，在构造项目仓库远程地址时使用到，属性review描述的是用作code review的server地址。

**default**：当project标签没有指定default标签的属性时，默认就使用在default标签列出的属性。属性revision描述的是项目默认检出的分支，属性remote描述的是默认使用的远程仓库名称，必须要对应的remote标签的name属性值，属性sync-j描述的是从远程仓库更新项目时使用的并行任务数。

**project**：每一个AOSP子项目在这里都对应有一个projec标签，用来描述项目的元信息。属性path描述的是项目相对于远程仓库URL的路径，属性name描述的是项目的名称，也是相对于 AOSP根目录的目录名称。例如，如果远程仓库URL为https://android.googlesource.com/platform，那么AOSP子项目bionic对应的远程仓库URL就为https://android.googlesource.com/platform/bionic，并且它的工作目录位于$(AOSP)/bionic。

**copyfile**：作为project的子标签，表示要将从远程仓库更新回来的文件拷贝到指定的另外一个文件去。

至些，我们就分析完成Manifest仓库的克隆过程了。在此基础上，我们再分析AOSP子项目仓库的克隆过程或者针对AOSP子项目的各种Repo命令就容易多了。

AOSP子项目仓库

执行完成repo init命令之后，我们就可以继续执行repo sync命令来克隆或者同步AOSP子项目了：

```
$ repo sync  
```
与repo init命令类似，repo sync命令的执行过程如下所示：

1. Repo脚本找到Repo仓库里面的main.py文件，并且执行它的入口函数 `_Main` ；

2. Repo仓库里面的main.py文件的入口函数 `_Main` 调用 `_Repo` 类的成员函数 `_Run` 对Repo脚本传递进来的参数进行解析；

3. `_Repo` 类的成员函数 `_Run` 解析参数发现要执行的命令是sync，于是就在subcmds目录中找到一个名称为sync.py的文件，并且调用定义在它里面的一个名称为Sync的类的成员函数Execute；

4. Sync类的成员函数Execute解析Manifest仓库的default.xml文件，并且克隆或者同步出在default.xml文件里面列出的每一个AOSP子项目。

在第3步中，Repo仓库的每一个Python文件是如何与一个Repo命令关联起来的呢？原来在Repo仓库的subcmds目录中，有一个__init__.py文件，每当subcmds被import时，定义在它里面的命令就会被执行，如下所示：

```python
all_commands = {}  

my_dir = os.path.dirname(__file__)  
for py in os.listdir(my_dir):  
if py == '__init__.py':  
    continue  

if py.endswith('.py'):  
    name = py[:-3]  
    clsn = name.capitalize()  
    while clsn.find('_') > 0:  
      h = clsn.index('_')  
      clsn = clsn[0:h] + clsn[h + 1:].capitalize()  

    mod = __import__(__name__,  
              globals(),  
              locals(),  
              ['%s' % name])  
    mod = getattr(mod, name)  
    try:  
      cmd = getattr(mod, clsn)()  
    except AttributeError:  
      raise SyntaxError('%s/%s does not define class %s' % (  
                  __name__, py, clsn))  

    name = name.replace('_', '-')  
    cmd.NAME = name  
    all_commands[name] = cmd  
```
__init__.py会列出subcmds目录中的所有Python文件（除了__init__.py），并且里面找到对应的类，然后再创建这个类的一个对象，并且以文件名为关键字将该对象保存在全局变量all_commands中。例如，对于sync.py文件，它的文件名称去掉后缀名后为sync，再将sync的首字母大写，得到Sync。也就是说，sync.py需要定义一个Sync类，并且这个类需要直接或者间接地从Command类继承下来。Command类有一个成员函数Execute，它的各个子类需要对它进行重写，以实现各自的功能。

_Repo类的成员函数_Run就是通过subcmds模块里面的全局变量all_commands，并且根据Repo脚本传进行来的第一个不带横线“-”的参数来找到对应的Command对象，然后调用它的成员函数Execute的。

Sync类的成员函数Execute的实现如下所示：

```python
class Sync(Command, MirrorSafeCommand):  
......  

def Execute(self, opt, args):  
    ......  

    mp = self.manifest.manifestProject  
    ......  

    if not opt.local_only:  
      mp.Sync_NetworkHalf(quiet=opt.quiet,  
                   current_branch_only=opt.current_branch_only,  
                   no_tags=opt.no_tags)  
    ......  

    if mp.HasChanges:  
      ......  
      mp.Sync_LocalHalf(syncbuf)  
      ......  

    all_projects = self.GetProjects(args,  
                             missing_ok=True,  
                             submodules_ok=opt.fetch_submodules)  
    ......  

    if not opt.local_only:  
      to_fetch = []  
      ......  
      to_fetch.extend(all_projects)  
      to_fetch.sort(key=self._fetch_times.Get, reverse=True)  

      fetched = self._Fetch(to_fetch, opt)  
      ......  

      if opt.network_only:  
 # bail out now; the rest touches the working tree  
 return  

      # Iteratively fetch missing and/or nested unregistered submodules  
      while True:  
 ......  
 all_projects = self.GetProjects(args,  
                                 missing_ok=True,  
                                 submodules_ok=opt.fetch_submodules)  
 missing = []  
 for project in all_projects:  
   if project.gitdir not in fetched:  
     missing.append(project)  
 if not missing:  
   break  
 ......  
 fetched.update(self._Fetch(missing, opt))  

    if self.UpdateProjectList():  
      sys.exit(1)  

    ......  

    for project in all_projects:  
      ......  
      if project.worktree:  
 project.Sync_LocalHalf(syncbuf)  

    ......  
```
Sync类的成员函数Execute的核以执行流程如下所示：

(1). 获得用来描述Manifest仓库的MetaProject对象mp。

(2). 如果在执行repo sync命令时，没有指定--local-only选项，那么就调用MetaProject对象mp的成员函数Sync_NetworkHalf从远程仓库下载更新本地Manifest仓库。

(3). 如果Mainifest仓库发生过更新，那么就调用MetaProject对象mp的成员函数Sync_LocalHalf来合并这些更新到本地的当前分支来。

(4). 调用Sync的父类Command的成员函数GetProjects获得由Manifest仓库的default.xml文件定义的所有AOSP子项目信息，或者由参数args所指定的AOSP子项目的信息。这些AOSP子项目信息都是通过Project对象来描述，并且保存在变量all_projects中。

(5). 如果在执行repo sync命令时，没有指定--local-only选项，那么就对保存在变量all `_projects` 中的AOSP子项目进行网络更新，也就是从远程仓库中下载更新到本地仓库来，这是通过调用Sync类的成员函数 `_Fetch` 来完成的。Sync类的成员函数 `_Fetch` 实际上又是通过调用Project类的成员函数Sync_NetworkHalf来将远程仓库的更新下载到本地仓库来的。

(6). 由于AOSP子项目可能会包含有子模块，因此当对它们进行了远程更新之后，需要检查它们是否包含有子模块。如果包含有子模块，并且执行repo sync脚本时指定有--fetch-submodules选项，那么就需要对AOSP子项目的子模块进行远程更新。调用Sync的父类Command的成员函数GetProjects的时候，如果将参数submodules_ok的值设置为true，那么得到的AOSP子项目列表就包含有子模块。将这个AOSP子项目列表与之前获得的AOSP子项目列表fetched进行一个比较，就可以知道有哪些子模块是需要更新的。需要更新的子模块都保存在变量missing中。由于子模块也是用Project类来描述的，因此，我们可以像远程更新AOSP子项目一样，调用Sync类的成员函数 `_Fetch` 来更新它们的子模块。

(7). 调用Sync类的成员函数UpdateProjectList更新$(AOSP)/.repo目录下的project.list文件。$(AOSP)/.repo/project.list记录的是上一次远程同步后所有的AOSP子项目名称。以后每一次远程同步之后，Sync类的成员函数UpdateProjectList就会通过该文件来检查是否存在某些AOSP子项目被删掉了。如果存在这样的AOSP子项目，并且这些AOSP子项目没有发生修改，那么就会将它们的工作目录删掉。

(8). 到目前为止，Sync类的成员函数对AOSP子项目所做的操作仅仅是下载远程仓库的更新到本地来，但是还没有将这些更新合并到本地的当前分支来，因此，这时候就需要调用Project类的成员函数Sync_LocalHalf来执行合并更新的操作。

从上面的步骤可以看出，init sync命令的核心操作就是收集每一个需要同步的AOSP子项目所对应的Project对象，然后再调用这些Project对象的成员函数Sync_NetwokHalft和Sync_LocalHalf进行同步。关于Project类的成员函数Sync_NetwokHalft和Sync_LocalHalf，我们在前面分析Manifest仓库的克隆过程时，已经分析过了，它们无非就是通过git fetch、git rebase或者git merge等基本Git命令来完成自己的功能。

以上我们分析的就是AOSP子项目仓库的克隆或者同步过程，为了更进一步加深对Repo仓库的理解，接下来我们再分析另外一个用来在AOSP上创建Topic的命令repo start。

## 在AOSP上创建Topic

在Git的世界里，分支（branch）是一个很核心的概念。Git鼓励你在修复Bug或者开发新的Feature时，都创建一个新的分支。创建Git分支的代价是很小的，而且速度很快，因此，不用担心创建Git分支是一件不讨好的事情，而应该尽可能多地使用分支。

同样的，我们下载好AOSP代码之后，如果需要在上面进行修改，或者增加新的功能，那么就要在新的分支上面进行。Repo仓库提供了一个repo start命令，用来在AOSP上创建分支，也称为Topic。这个命令的用法如下所示：

```
$ repo start BRANCH_NAME [PROJECT_LIST]  
```
参数BRANCH_NAME指定新的分支名称，后面的PROJECT_LIST是可选的。如果指定了PROJECT_LIST，就表示只对特定的AOSP子项目创建分支，否则的话，就对所有的AOSP子项目创建分支。

根据前面我们对repo sync命令的分析可以知道，当我们执行repo start命令的时候，最终定义在Repo仓库的subcmds/start.py文件里面的Start类的成员函数Execute会被调用，它的实现如下所示：

```python
class Start(Command):  
......  

def Execute(self, opt, args):  
    ......  

    nb = args[0]  
    if not git.check_ref_format('heads/%s' % nb):  
      print("error: '%s' is not a valid name" % nb, file=sys.stderr)  
      sys.exit(1)  

    err = []  
    projects = []  
    if not opt.all:  
      projects = args[1:]  
      if len(projects) < 1:  
 print("error: at least one project must be specified", file=sys.stderr)  
 sys.exit(1)  

    all_projects = self.GetProjects(projects)  

    pm = Progress('Starting %s' % nb, len(all_projects))  
    for project in all_projects:  
      pm.update()  
      ......  
      if not project.StartBranch(nb):  
 err.append(project)  
    pm.end()  

    ......  
```
参数args[0]保存的是要创建的分支的名称，参数args[1:]保存的是要创建分支的AOSP子项目名称列表，Start类的成员函数Execute分别将它们保存变量nb和projects中。

Start类的成员函数Execute接下来调用父类Command的成员函数GetProjects，并且以变量projects为参数，就可以获得所有需要创建新分支nb的AOSP子项目列表all_projects。在all_projects中，每一个AOSP子项目都用一个Project对象来描述。

最后，Start类的成员函数Execute就遍历all_projects里面的每一个Project对象，并且调用它们的成员函数StartBranch来执行创建新分支的操作。

Project类的成员函数StartBranch的实现如下所示：

```python
class Project(object):  
......  

def StartBranch(self, name):  
    """Create a new branch off the manifest's revision. 
    """  
    head = self.work_git.GetHead()  
    if head == (R_HEADS + name):  
      return True  

    all_refs = self.bare_ref.all  
    if (R_HEADS + name) in all_refs:  
      return GitCommand(self,  
                 ['checkout', name, '--'],  
                 capture_stdout = True,  
                 capture_stderr = True).Wait() == 0  

    branch = self.GetBranch(name)  
    branch.remote = self.GetRemote(self.remote.name)  
    branch.merge = self.revisionExpr  
    revid = self.GetRevisionId(all_refs)  

    if head.startswith(R_HEADS):  
      try:  
 head = all_refs[head]  
      except KeyError:  
 head = None  

    if revid and head and revid == head:  
      ref = os.path.join(self.gitdir, R_HEADS + name)  
      try:  
 os.makedirs(os.path.dirname(ref))  
      except OSError:  
 pass  
      _lwrite(ref, '%s\n' % revid)  
      _lwrite(os.path.join(self.worktree, '.git', HEAD),  
       'ref: %s%s\n' % (R_HEADS, name))  
      branch.Save()  
      return True  

    if GitCommand(self,  
           ['checkout', '-b', branch.name, revid],  
           capture_stdout = True,  
           capture_stderr = True).Wait() == 0:  
      branch.Save()  
      return True  
    return False  
```
Project类的成员函数StartBranch的执行过程如下所示：

(1). 获得项目的当前分支head，这是通过调用Project类的成员函数GetHead来实现的。

(2). 项目当前的所有分支保存在Project类的成员变量bare_ref所描述的一个GitRefs对象的成员变量all中。如果要创建的分支name已经项目的一个分支，那么就直接通过GitCommand类调用git checkout命令来将该分支检出即可，而不用创建新的分支。否则继续往下执行。

(3). 创建一个Branch对象来描述即将要创建的分支。Branch类的成员变量remote描述的分支所要追踪的远程仓库，另外一个成员变量merge描述的是分支要追踪的远程仓库的分支。这个要追踪的远程仓库分支由Manifest仓库的default.xml文件描述，并且保存在Project类的成员变量revisionExpr中。

(4). 调用Project类的成员函数GetRevisionId获得项目要追踪的远程仓库分支的sha1值，并且保存在变量revid中。

(5). 由于新创建的分支name需要追踪的远程仓库分支为revid，因此如果项目的当前分支head刚好就是项目要追踪的远程仓库分支revid，那么创建新分支name就变得很简单，只要在项目的Git目录（位于.repo/projects目录下）下的refs/heads子目录以name名称创建一个文件，并且往这个文件写入写入revid的值，以表明新分支name是在要追踪的远程分支revid的基础上创建的。这样的一个简单的Git分支就创建完成了。不过我们还要修改项目工作目录下的.git/HEAD文件，将它的内容写为刚才创建的文件的路径名称，这样才能将项目的当前分支切换为刚才新创建的分支。从这个过程就可以看出，创建的一个Git分支，不过就是创建一个包含一个sha1值的文件，因此代价是非常小的。如果项目的当前分支head刚好不是项目要追踪的远程仓库分支revid，那么就继续往下执行。

(6). 执行到这里的时候，就表明我们要创建的分支不存在，并且我们需要在一个不是当前分支的分支的基础上创建该新分支，这时候就需要通过调用带-b选项的git checkout命令来完成创建新分支的操作了。选项-b后面的参数就表明要在哪一个分支的基础上创建分支。新的分支创建出来之后，还需要将它的文件拷贝到项目的工作目录去。

至此，我们就分析完成在AOSP上创建新分支的过程了，也就是repo start命令的执行过程。更多的repo命令，例如repo uplad、repo diff和repo status等，可以以参考官方文档[http://source.android.com/source/using-repo.html](http://source.android.com/source/using-repo.html)，它们的执行过程和我们前面分析repo sync、repo start都是类似，不同的是它们执行其它的Git命令。有兴趣的小伙伴自己尝试自己去分析一下。更多的知识分享，请关注老罗的新浪微博：[http://weibo.com/shengyangluo](http://weibo.com/shengyangluo)。