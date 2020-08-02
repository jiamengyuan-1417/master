<big>现场性能测试造包自动化(Java版)-分割PDF&&批量修改&&加密&&消息</big>

## 一、概述
现场性能测试方案要求：数量2000，节点种类（1101/1301/1302/1404/1406/1501）这六个节点，具体每个节点的大小和数量这里不详说，总共大小80多个G
目前是通过Java代码代替跑包过程中的部分手动操作过程，节约时间，提高效率，这块自动化也就是根据现场环境所产生的需求去实现的
造包过程：以现有的PDF模板分割成想要的PDF大小（每个PDF是根据方案要求数据包大小和具体数据包中的PDF数量算出）；批量修改数据包并替换PDF生成指定大小的数据包（偏差在十几分之一左右）；批量加密；批量生成消息

> 实现人：贾孟源

**前提**：需要修改的测试数据包和PDF模板已经准备好了
**步骤**：

a.分割PDF

1. 将PDF模板放到指定目录（如：G:\\参与项目）
2. 以key-value形式修改pdf-maven-web配置文件config.ini里的参数（格式：key=value；地址参数要用\\\）
3. 用部署好的Java环境（如eclipse、idea等）导入运行pdf-maven-web，从生成目录取出对应PDF（如：G:\\参与项目\\符合要求的PDF\\Doc2.pdf）
4. 注：配置文件中参数templatePath是指PDF模板路径、sourcePath是指生成PDF路径

b.修改替换加密上传

1. 将需要修改的测试包和要替换的PDF放在指定目录（如：E:\BJCMxml和E:\BJCMpdf）
2. 以key-value形式修改update_PDF配置文件config.properties（格式：key=value）
3. 这里需要注意的一点是update_PDF中UpdateXml类的第48行的properties常量值是要你的配置文件名要一致
4. 双击bat脚本(利用for循环控制生成数据包的数量)循环运行update_PDF，从生成修改包路径下（如：E:\BJCMimp\data）取走修改后的包
5. 将需要上传到服务器指定目录下的测试包放在uploadpackage2程序子目录下uploadzip
6. 现场是加密环境，所以跑加密包的话，需要将uploadpackage2程序子目录下的config.ini的if_encrypt改为true，然后将程序里主方法注释符号去掉，将clickBat方法里第37行参数改为jiamizippath，导入程序包运行环境就好了（之后加密会陆续优化，实现全自动）
7. 注意uploadpackage2程序子目录下的config.ini需要维护，提醒还是先备份，要是上传不上去，别着急，先看看config.ini里那六个参数还有没有

c.生成消息

1. 将符合规则的消息模板放在一个目录下（如：E:/BJCMmsg/），同时创建一个消息生成目录（如:E:\BJCMimp\msg\）。注：uploadpackage2加密上传的数据包备份在项目目录下（如：F:\eclipse\uploadpackage2\uploadzip\jiami）
2. 导入message-make程序包运行环境就批量生成了uploadpackage2加密上传数据包的专属消息

## 二、实现记录

### 相关工具

以下为几个程序实现所需工具和工具包说明

- **run_encrypt_upload.bat**

     功能：通过SSH连接，实现windows和linux之间文件的上传下载。  在windows系统通过命令运行pscp.exe即可，带上传的文件路径、目标服务器IP、用户名、密码、目标路径作为参数。文件：http://open.thunisoft.com/test-Team2/autotest-bjcm/tree/master/bat/相关工具

* **pscp.exe**  
  
  * 功能：通过SSH连接，实现windows和linux之间文件的上传下载。  
  * 在windows系统通过命令运行pscp.exe即可，带上传的文件路径、目标服务器IP、用户名、密码、目标路径作为参数。
* 文件：http://open.thunisoft.com/test-Team2/autotest-bjcm/tree/master/bat/相关工具
  
* **crypt**  
  * 开发实现的工具，可以对zip包加密、解密、修改包名md5值。  
  * 开发给的原版工具：http://open.thunisoft.com/test-Team2/autotest-bjcm/tree/master/bat/相关工具/crypt  
  * 局限性：加密、解密后的包存放路径，是写死在代码里的，在对应zip包同级目录下生成了“`jiami`”“`jiemi`”文件夹（可以让开发改，但这样目前也没有特别不方便的，暂时就用这个了）。

- **itextpdf-5.5.13.jar**
  - 通过Maven加载的处理PDF的工具包，用于分割时候引用其中的方法
  - 网上就有原版jar包和Maven引入的dependency

### 实现说明

经过时间和实践的检验，已经修改过好几版了（update_XML：修改XML某个节点值-修改测试包某个节点值（包括节点值为null的情况或者说非必填的）-通过读取配置文件来批量修改XML-支持测试包包名UUID和MD5的更新）-在原来的基础上增加替换PDF内容，按照节点规则随机生成PTTYBH和AJBH蜕变成update_PDF（uploadpackage2：上传到服务器对应目录-支持加密上传），有什么问题可以直接联系我，必有重谢

## 三、实现思路

### update_PDF

> 是基于update_XML进行的优化；

1. 最终整体思路：通过配置文件批量修改测试包里XML节点值和替换PDF内容
2. 思路细节（1）：将测试包解压到某一目录，当解压到PDF的时候，利用FileChannel将模板PDF写入加压对应目录下
3. 思路细节（2）：解压完，对其中YWXX.xml进行解析，将配置信息config.properties通过解析遍历到Map，通过遍历Map到已经解析的YWXX.xml找到节点，然后进行赋值；此外并对PTTYBH和AJBH根据规则有专门的函数生成，然后进行赋值
4. 思路细节（3）：保存XML到解压的目录，然后对YWXX.xml和其余剩下的文件进行压缩到另一目录下，对其赋予新的包名（YWLB+LCJDBH+FSDWBH+JSDWBH+SJBS++MD5），已对多个接收方的测试包进行处理
5. 缺点：YWXX.xml有例如R表嫌疑人信息表，会出现重复嫌疑人数据（嫌疑人主键不能一样），导致此数据包跑包报错，这里需要手动改一下

### uploadpackage2
1. 最终整体思路：将数据包按照发送方的不同分别上传到服务器的不同目录下，支持加密
2. 思路细节（1）：从原来已经有的上传shell脚本run_encrypt_upload.bat基础上进行优化（目前需要实现的是根据包名上传到对应目录和支持加密）
3. 思路细节（2）：run_encrypt_upload.bat是通过读取配置文件参数调用crypt和pscp.exe对数据包进行加密和上传，所以程序是对配置文件参数sendPath和zip_name不断地修改来实现不同发送方的包进入不同的目录，循环读入修改配置文件之后，每一个包上传都会运行run_encrypt_upload.bat，实现上传事件
4. 缺点：弹出的DOM窗口没有设置自动关闭，主要是因为设置自动关闭会出现有包还没有上传上去就关闭窗口了，之后会优化；加密有手动需要改的代码，这边也会实现自动化

### pdf-maven-web

1. 最终整体思路：将模板PDF按照设置的起始页进行分割，生成新的PDF
2. 思路细节：获取配置文件中的参数和参数值写入map中，然后传参数值到PDFReader和PDFCopy中，按照获取的起始页数进行读取和copy，最后生成新的PDF。一般页数和大小是成比例的，所以根据页数生成想要大小的PDF，这个方法也是可取的。

### message_make

1. 最终整体思路：将模板消息文件中的必填项（已和开发确认所需节点）批量更改为对应数据包专属消息值
2. 思路细节：获取模板目录下的消息XML，对消息文件进行解析，通过分析数据包包名将消息模板中的节点值替换成每个需要生成数据包消息的专属消息值，由于涉及到的节点比较少，所以不用遍历，直接用document对象去获取节点，然后进行赋值即可

### 其他注意事项

* update_XML内部传路径参数比较多，修改时请注意
* uploadpackage2目前只支持通州区单位上传,需要保持uploadpackage2目录下的config.ini、crypt、pscp.exe、run_encrypt_upload.bat完好无损的呆在那