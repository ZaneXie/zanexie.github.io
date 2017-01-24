---
title: PHP7源码的编译与调试
date: 2015-12-09 19:56:20
tags:
---

PHP7源码的编译和调试很方便，此处简要介绍一下在Ubuntu下的调试方法（使用其他系统的话可以用虚拟机启一台Ubuntu）

1. 带界面的Ubuntu。首选官方Ubuntu Desktop，推荐LTS版本。如果用虚拟机性能不足的话，可以使用XUbuntu 。
2. 集成gdb界面的IDE。直接命令行gdb调试太辛苦，效率也不高，建议使用IDE调试。这里推荐使用eclipse cdt和QtCreator。
3. PHP7代码。从官网[http://php.net/](http://php.net/)下载打包好的源码，解压后进入目录，运行
    ```
    ./configure --enable-debug --prefix=/home/php/usr
    ```
    --enable-debug表示开启调试，编译时才会带上调试符号，要调试的话一定要带上。prefix指安装的目录，为了避免不小心装到系统目录里，建议自行选择一个目录。然后就是 
    ```
    make
    ```
    了。编译好后的可执行文件在sapi下，分为不同的版本cli、fpm、cgi等。调试简单脚本时，最方便的是sapi/cli/php，即命令行的php。
4. 配置IDE 
    * eclipse
    1. 记得选择for C/C++ developer的eclipse（集成CDT才能调试）。任选一个workspace之后，点击上方的File--Import，如下图选择Existing code as Autotools project（选as Makefile project也行）
{% asset_img JBSER5YCL8QZVVGAU6.png basic_hashtable %}
     2. 然后选择php源码的目录，一直下一步到完成，eclipse会自动索引整个代码，然后就能方便的看代码了。一般到这一步，eclipse会提示有些文件的代码有错，不用管它，因为有些定义没找到，只要上一步编译正常就ok，不影响调试。
     3. 配制debug。在上方选择Run/Debug Configurations... 新建一个C/C++ Application的调试配置。选择刚才编译好的php可执行文件，并在Arguments里配上待调试的php文件位置 
    {% asset_img 749V_5.png basic_hashtable %}
    {% asset_img ISQ489O32RBF_Y633VU.png basic_hashtable %}
     4. 再点击右下角的Debug，大功告成！默认情况下eclipse会停在main函数的入口处，然后就可以愉快的打断点调试了。在这个地方eclipse可能会提示代码有错误，是否继续调试？不用管它，强制开始就好。
    
    * Qt Creator
        1. 使用方法和eclipse类似，安装之后选File/New File or Project/Import Project/Import Existing Project，依次选目录，加进去即可。 
    {% asset_img qt.png basic_hashtable %}
        2. 待索引好后，依次点击上方的Debug/Start Debugging/Start Debugging Without Deployment， 然后和依次填写要调试的可执行文件和arguments（和eclipse差不多），然后就大功告成了！如果Qt Creator没有默认停住，在代码里打个断点重新开始就行了。

个人觉得eclipse使用更方便，但是在使用调试的时候，有些变量打不出，而使用Qt Creator就正常。貌似是eclipse的一个bug，结合两个IDE使用就ok了。 其他的调试工具配置方法都类似：先自己编译源码（记得加--enable-debug），编译好为调试工具设置可执行文件、代码目录、arguments即可。
