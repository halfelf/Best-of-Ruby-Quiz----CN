填字游戏
=========================

这个Ruby测验是要写出一个您童年时候喜欢的游戏：填字游戏。没有玩过也不用担心，很容易学。填字游戏就是一句夹杂几个占位符的小故事，像这样：

<pre>
I had a ((an adjective)) sandwich for lunch today. It dripped all over my ((a body part)) and ((a noun)).
</pre>

出题人，作为唯一一个知道这个故事的人，将要求另外一个人想出每个占位符所对应的答案并记下来。在这个例子中，他们想要一个形容词，身体某个部分和一个名词。然后出题人把答案放进去念出来。这里可能会变成这个样子：

<pre>
I had a smelly sandwich for lunch today. It dripped all over my big toe and bathtub.
</pre>

有人笑了吧～

程序脚本将扮演读者的角色，要求用户给出一些单词，并把每个占位符用用户给出的答案替换掉。

这个故事的格式很简单，用一组((...))标记占位符。像这样：

<pre>
Our favorite language is ((a gemstone)).
</pre>

如果你的程序是按照这个模板写的，它还需要你给出“a gemstone”并显你编的故事：

<pre>
Our favorite language is Ruby.
</pre>

这样就涵盖了简单的情况，但有些时候我们可能需要多次使用某个答案。所以要引入一种标志符：

<pre>
Our favorite language is ((gem:a gemstone)). We think ((gem)) is better than ((a gemstone)).
</pre>

对于上面这个故事，你的程序将要求两块宝石（gemstone），并用一个来替换((gem:...))和((gem))。当((...))中出现一个冒号的时候，冒号前的部分将作为一个指针指向可重用的值，而冒号之后的部分就是值。这样就有了如下的结果：

<pre>
Our favorite language is Ruby. We think Ruby is better than Emerald.
</pre>

你可以选择任何你喜欢的界面，只要用户可以得到最终结果。你可以和the Ruby Quiz网站上一个基于CGI的实现来玩这个游戏。也可以找到我在the Ruby Quiz网站上所用的两个填字游戏文件。


解答部分
------------------------------------------

这是个有趣的小消遣吧？实际上，我惊讶的发现（当写这个测验的时候）这个挑战是多么的有实际意义。填字游戏确实是一个模板问题，而且涉及到编程的诸多方面。Ruby on Rails的视图部分就是一个现实的例子。

从这个角度来思考该问题让我想到，Ruby不是提供了一个模板引擎吗？是的，确实有。

Ruby包含了一个叫做ERB的标准库，ERB允许你将Ruby代码嵌入任何文本文档中。当用这个库处理该文本时，嵌入的代码会运行。这可以用来动态的构建文档内容。

对于这个测验，我们只需要ERB的一个特性。当我们把ERB应用到一个文件时，任何在看起来很好玩的<%=... %> 标记中的Ruby代码会被执行，并且该执行代码的返回值会插入该文档中。这可以被看作是一种延迟的改写（就像Ruby的#{...}语法，发生在被触发时而非字符串创建时）

我们用ERB试试：

[madlibs/erb_madlib.rb](http://media.pragprog.com/titles/fr_quiz/code/madlibs/erb_madlib.rb)

```ruby
# use Ruby' s standard template engine
require "erb"
# storage for keyed question reuse
$answers = Hash.new

# asks a madlib question and returns an answer
def q_to_a( question ) 
  question.gsub!(/\s+/, " ")      # normalize spacing
  
  if $answers.include? question   # keyed question
    $answers[question]            
  else                            # new question 
    key = if question.sub!(/^\s*(.+?)\s*:\s*/, "") then $1 else nil end

    print "Give me #{question}: " 
    answer = $stdin.gets.chomp

    $answers[key] = answer unless key.nil?

    answer
  end
end

# usage
unless ARGV.size == 1 and test(?e, ARGV[0])
  puts "Usage: #{File.basename($PROGRAM_NAME)} MADLIB_FILE"
  exit
end

# load Madlib, with title
madlib = "\n#{File.basename(ARGV.first, ' .madlib' ).tr(' _' , ' ' )}\n\n" +
          File.read(ARGV.first)
# convert ((...)) to <%= q_to_a(' ...' ) %> 
madlib.gsub!(/\(\(\s*(.+?)\s*\)\)/, "<%= q_to_a(' \\1' ) %>") 
# run template 
ERB.new(madlib).run
```

这里中心思想就是把((...))转换成<%= ... %>，然后我们就可以用ERB了。当然，<%= a noun %>不是一段合法的Ruby代码，这样我们就需要个辅助方法。这就是为什么有个q_to_a()了。它将填字游戏中的替换单词当作参数并返回用户的答案。为了使用它，我们需要把((...))转换成<%= q_to_a('...') %>。之后，ERB会接手其余的工作。


####自定模板####

对于简单的填字游戏，你其实不需要像ERB这样牛x的东西。很轻松就能写个自己的实现，而且很多人就是这么做的。我们来弄个自己的解析程序。

其实在我们的填字游戏中只有三种故事元素。普通的文字，对用户的问题，以及用来替换的值。

最后一部分是最容易识别的，我们从这里着手。如果问题中出现了了占位符((...))，那么这里就是一个可以替换的部分。编码这部分是很简单的：

[madlibs/parsed_madlib.rb](http://media.pragprog.com/titles/fr_quiz/code/madlibs/parsed_madlib.rb)

```ruby
# A placeholder in the story for a reused value.
class Replacement 
  # Only if we have a replacement for a given token is this class a match. 
  def self.parse?( token, replacements )
    if token[0..1] == "((" and replacements.include? token[2..-1]
      new(token[2..-1], replacements)
    else 
      false
    end 
  end

  def initialize( name, replacements )
    @name           = name 
    @replacements   = replacements
  end

  def to_s 
    @replacements[@name]
  end 
end
```

通过parse?()方法，你可以把文中要替换的值转换成要用来补全故事的编码元素。当参数token不是一个要替换的值的时候，parse?()的返回值为false，否则返回用来替换的对象。

在parse?()里，如果某个标志符以((开始并且是replacements这个Hash的键值，则该标志符被选中。这样就存储名字和该Hash以便需要的时候查找，即运行to_s()方法。

轮到Question对象了：

[madlibs/parsed_madlib.rb](http://media.pragprog.com/titles/fr_quiz/code/madlibs/parsed_madlib.rb)

```ruby
# A question for the user, to be replaced with their answer.
class Question 
  # If we see a ((, it's a prompt. Save their answer if a name is given. 
  def self.parse?( prompt, replacements )
    if prompt.sub!(/^\(\(/, "") 
      prompt, name = prompt.split(":").reverse
      replacements[name] = nil unless name.nil?
      new(prompt, name, replacements)
    else 
      false
    end 
  end

  def initialize( prompt, name, replacements )
    @prompt       = prompt
    @name         = name
    @replacements = replacements
  end

  def to_s 
    print "Enter #{@prompt}: " 
    answer = $stdin.gets.to_s.strip

    @replacements[@name] = answer unless @name.nil?

    answer
  end 
end
```

在故事中仍存留的以((开始的标志符被识别成一个Question对象，而非Replacement。对于prompt和name，如果有的话，和replacements分开存储以备使用。

当to_s方法被调用时，Question会问询用户并返回answer。在问题命名后也会设置@replacements中的值。

小故事中还有另外一个元素：散句。Ruby正好有个用来处理的对象，一个String。我们只需要调整String的接口就能用了。

[madlibs/parsed_madlib.rb](http://media.pragprog.com/titles/fr_quiz/code/madlibs/parsed_madlib.rb)

```ruby
# Ordinary prose.
class String 
  # Anything is acceptable. 
  def self.parse?( token, replacements )
    new(token)
  end 
end
```

没啥说的。故事中所有其他元素都是散句，parse?()会接受所有的东西并返回一个字符串。

下面是代码的其他部分：

[madlibs/parsed_madlib.rb](http://media.pragprog.com/titles/fr_quiz/code/madlibs/parsed_madlib.rb)

```ruby
# argument parsing
unless ARGV.size == 1 and test(?e, ARGV[0]) 
  puts "Usage: #{File.basename($PROGRAM_NAME)} MADLIB_FILE" 
  exit
end
madlib = << MADLIB

#{File.basename(ARGV.first, ".madlib").tr("_", " ")} 

#{File.read(ARGV.first)}
MADLIB

# tokenize input
tokens = madlib.split(/(\(\([^)]+)\)\)/).map do |token| 
  token[0..1] == "((" ? token.gsub(/\s+/, " ") : token
end

# identify each part of the story
answers = Hash.new 
story	= tokens.map do |token|
  [Replacement, Question, String].inject(false) do |element, kind| 
    element = kind.parse?(token, answers) and break element
  end 
end

# share the results
puts story.join
```

在一些熟悉的参数解析代码后，从输入到最终结果，我们的解决方案用共有三步。 首先输入文件被分解成标志符。这一步符号化只是简单的调用split()。请注意，split()使用的正则表达式捕获的所有东西都是返回集合的一部分。这样记号((...))也会被返回，尽管他们还是split()的分隔符。然而，捕获的括号会丢弃结尾的))。开头的((仍然会保留以待之后的识别。最后，所有的空白符都会统一转化为一个空格，避免他们占用多行。

在第二步中，每个标志符都根据我们先前的定义被转换成一个Replacement，Question或者String对象。别让那个花哨的inject()调用弄晕了你。也可以写成element or kind.parse?(token, answers)。但前者的好处是即便检查到了一个匹配，还会继续检查所有的类。 break则表示当我们找到能接受标志符的结果后从里面跳出。

最后一步则是再现该故事。为了理解者简单的一行代码，你需要知道在组合所有的对象前会调用to_s()，以保证join()所有的元素都是String对象。

尽管这个解析过程相对于其他我们看过或将要看到的程序来说有些复杂，但可能值得借鉴的是，如果我们需要重复某个故事，只需要重复代码最后一部分。而解析的部分是完全可重用的。

####迷你填字####

我们来看看Dominik Bathon写的超级短小精悍的实现。显然这段代码充满了奇技淫巧，并不是所有人都能很好的理解。这里面确实有些有意思的东西：

[madlibs/golfed_madlib.rb](http://media.pragprog.com/titles/fr_quiz/code/madlibs/golfed_madlib.rb)

```ruby
keys=Hash.new { |h, k| 
    puts "Give me #{k.sub(/\A([^:]+):/, "")}:" 
    h[$1]=$stdin.gets.chomp
} 
puts "", $*[0].split(".")[0].gsub("_", " "),
      IO.read($*[0]).gsub(/\(\(([^)]+)\)\)/) { keys[$1] }
```

为了理解这段代码，从后面那个puts开始看。这个用法不太常见，Ruby的puts方法接受一个需要打印的行的列表。这段代码就是这么用的。前三行代码在我们敲出故事前只是一个空String并产生一个空行。

puts打印出的第二行是填字游戏的名称，这是从文件名得到的。理解这段代码的关键是知道Perl风格变量$*是ARGV的一个同义词。这样你就能发现程序读取了头一个命令行参数，并用split去掉了扩展名，之后又弄漂亮了点（把下划线变成空格）。结果就是个适合阅读的题目。

最后一行实际上是整个填字游戏故事。你又看见了$*的头一个元素。gsub处理了提出的问题和替换过程，用一个简单的Hash。

我们仔细看看这个Hash。现在跳回程序开头。这个Hash使用了一个代码块在需要的时候变出默认的键值对。然后打印出一个问题，并使用sub()来替换其中的关键词。你会看到程序从用户那里读取答案并用$1作为键值将其加入到Hash中。那么这里的技巧就在于变量$1了。我们看到后面puts()中的gsub()将$1的实值替换到了整个填字游戏结果中。 而这里，Hash的代码块进行了另外一种替换，并覆写了$1变量。如果替换部分是有东西的，那么$1就是那部分东西。而如果sub()调用失败了，$1就不会改变。这样，由于我们使用了一个Hash，之后对同一个键值的访问将返回已经设置了的值，这段神奇的代码块就是这么工作的。

再多说一句，这段代码明显有几个坏习惯。。。但也用了一些Ruby中少见而有趣的符号，从而用很少的代码解决了大量的工作。

####补充练习####

1.扩展填字游戏的语法以支持条件变化

2.改进你的程序以适应新语法
