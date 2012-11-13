GEMCOM解析器
====================

GEDCOM是"GEnealogical Data COMmunication"文件格式。这是一种用来传递基因图谱数据的纯文本电子文档。这个问题的目标是建立一个简单的解析器来把GEDCOM转换成XML。

####GEDCOM格式####

GEDCOM文件的格式是很直观的。每行代表树中的一个节点。大致看起来是这样：

<pre>
0 @I1@ INDI
1 NAME Jamis Gordon /Buck/
2 SURN Buck
3 GIVN Jamis Gordon
1 SEX M
...
</pre>

大致来说每行都是如下格式：

<pre>
LEVEL TAG-OR-ID [DATA]
</pre>

LEVEL是一个整数，表示当前在树中的深度。如果下面行的级别比现在节点的大，表示他们是当前节点的子孙。

TAG-OR-ID是该节点的数据类型的一个标记，或者是一个特殊的单一标志符。标记是一个三个或四个大写字母组成的单词。单一标志符总是由@字符包含(比如@I54@)。如果给出一个标志符ID，那么DATA段就是子树的类型标记。

这样，我们看看刚才给出的例子。能看出来：

* 0 @I1@ INDI.这行表示一个类型为INDI("individual")的子树的开始。该个人的ID是@I1@。
* 1 NAME Jamis Gordon /Buck/. 从这行展开一个名为NAME的子树，其值为Jamis Gordon /Buck/
* 2 SURN Buck. 这是NAME子树的一个子元素，类型是SURN。(译注：意即surname姓)
* 2 GIVN Jamis Gordon. 和上面的SURN差不多，指明了该人的名(given name)
* 1 SEX M. 建立了一个新的INDI的子元素，类型为SEX (比如, "gender").

剩下差不多都这样。

在层级和标志之间允许有若干的空白。空行将被忽略。

####目标####

目标是创建一个接受GEDCOM文件作为输入并将其转化为XML的解析器。前面的GEDCOM片段将扩展成这样：

```xml
<gedcom> 
  <indi id="@I1@">
    <name>
      Jamis Gordon /Buck/ 
      <surn>Buck</surn> 
      <givn>Jamis Gordon</givn>
    </name> 
    <sex>M</sex>
    ...
  </indi>
  ... 
</gedcom>
```

####输入样例####

[这儿](http://www.rubyquiz.com/royal.ged)有一个比较大的在线GEDCOM文件，包含了欧洲诸多王室血系。该文件插入了很多空白来增强可读性。

解答部分
------------------


我们直接看看Hans Fugal提交的一个解法：

[gedcom_parser/simple.rb](http://media.pragprog.com/titles/fr_quiz/code/gedcom_parser/simple.rb)

```ruby
#! /usr/bin/ruby
require 'rexml/document'

doc = REXML::Document.new "<gedcom/%gt;"
stack = [doc.root]

ARGF.each_line do |line|
  next if line =~ /^\s*$/

  # parse line
  line =~ /^\s*([0-9]+)\s+(@\S+@|\S+)(\s(.*))?$/ or raise "Invalid GEDCOM"
  level = $1.to_i 
  tag = $2 
  data = $4

  # pop off the stack until we get the parent
  while (level+1) < stack.size 
    stack.pop
  end
  parent = stack.last

  # create XML tag
  if tag =~ /@.+@/ 
    el = parent.add_element data 
    el.attributes[' id' ] = tag
  else
    el = parent.add_element tag
    el.text = data
  end
  
  stack.push el
end
doc.write($stdout,0) 
puts
```

这段代码使用了REXML标准库。这是一组用来解析(此处是生成)XML文档的工具。这里的用法比较基本。首先创建一份文档，之后当元素生成的时候向里面添加。最后，把整个文档写入到$stdout。

前面的部分以创建一个REXML文档和一个管理父子关系的堆栈开始。堆栈就是个Array，Ruby的数组很全能。设置完成后，代码从$stdin或者命令行参数指定的文件逐行读入。也就是ARGF对象干的事情。

每一行分三步处理。首先解析该行。Hans使用了Regexp来分解该行并将提取出的东西赋值给level，tag和data。

第二步是回溯栈，直到我们找到了当前行元素的父元素。这将保证下面的代码能够将当前元素放置到XML文档的正确位置。

最后一步添加元素。在这里将检测行内数据属于本问题规定的两种格式中哪一个。REXML用来创建元素的正确形式并将其添加到相应父元素下。之后栈对新元素做出相应更新。

在所有的东西读取完后，整个XML将输出到$stdout。

####优化读写循环####

使用REXML的一个问题是在输出前需要建立整个文档。在处理较大的GEDCOM文件时REXML需要存储处理的所有信息，这在某些系统上可能耗尽内存。如果你需要改进这点，就得建立自己的XML行。这种方式的有点是只要你读入了某元素的所有子元素(当LEVEL那个数变小时)，就可以写该节点。这样能更有效率。我们来看看Jamis Buck使用这种方式的解法：

[gedcom_parser/efficient.rb](http://media.pragprog.com/titles/fr_quiz/code/gedcom_parser/efficient.rb)

```ruby
#!/usr/bin/env ruby
class GED2XML

  IS_ID = /^@.*@$/
  
  class Node < Struct.new( :level, :tag, :data, :refid ) 
    def initialize( line=nil )
      level, tag, data = line.chomp.split( /\s+/, 3 ) 
      level = level.to_i 
      tag, refid, data = data, tag, nil if tag =~ IS_ID 
      super level, tag.downcase, data, refid
    end 
  end

  def indent( level ) 
    print " "*(level+1)
  end

  def safe( text ) 
    text.
      gsub( /&/, "&amp;" ). 
      gsub( /</, "&lt;" ). 
      gsub( />/, "&gt;" ). 
      gsub( /"/, "&quot;" )
  end

  def process( io )
    node_stack = [] 

    puts "<gedcom>"
    wrote_newline = true

    io.each_line do |line| 
      next if line =~ /^\s*$/o 
      node = Node.new( line )
      
      while !node_stack.empty? &amp;&amp; node_stack.last.level >= node.level 
        prev = node_stack.pop 
        indent prev.level if wrote_newline 
        print "</#{prev.tag}>\n"
        wrote_newline = true 
      end

      indent node.level if wrote_newline 
      print "<#{node.tag}" 
      print " id=\"#{node.refid}\"" if node.refid

      if node.data 
        if node.data =~ IS_ID
          print " ref=\"#{node.data}\">" 
        else
          print ">#{safe(node.data)}" 
        end
        wrote_newline = false 
      else
        puts ">"
        wrote_newline = true 
      end
      
      node_stack << node
    end

    until node_stack.empty? 
      prev = node_stack.pop 
      indent prev.level if wrote_newline 
      print "</#{prev.tag}>\n" 
      wrote_newline = true
    end

    puts "</gedcom>" 
  end

end

GED2XML.new.process ARGF  
```

这段代码中最显眼的部分是内部类Node。Jamis想要在这里使用Struct，也需要这个类能够在指定行找到所需数据。这样，Node就直接从构造方法派生。Struct.new返回一个它创建的Class对象，并直接充当Node的父类。你可以把这个比作匿名继承。在Node的initialize方法中，数据被按需分解并传递给Struct的标准构造器。

接下来，GED2XML定义了一组辅助方法。indent方法是用来在每行开头缩进，保持XML格式美观的。safe方法是必需的，我们现在没有REXML来处理转义了。它返回一个传入串的XML转义结果。

我们接着来看process方法，这是程序的核心部分。这也和前面Hans的解法差不多。只有这么几个区别：

* 输出将直接打印出来，而不是用REXML来建立元素。
* 把元素弹出栈的时候，我们需要打印一个结束标记。
* 这也意味着我们需要在解析了所有的东西后再清空栈，以确保所有的元素都有结束标记。

这就是解法的全部内容了。正如你看到的那样，最后一行只是简单的把ARGF传给process来触发转换。

再一次提下，第二种解法更有效率，能在有限的硬件条件下处理大规模数据。相对来说，第一种更简单一些，也差不多能处理大多数状况。“完成够用的最简单的设计”(Use the simplest thing that possibly work)，像极限编程众所喜欢说的那样～

####补充练习####

1.比较我们讨论的两种方法以及你的解法的运行时和内存使用状况。注意其中的差别。

2.反转题目的过程。读入你生成的XML，并输出GEDCOM文件，用题目中例子的格式
