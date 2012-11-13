Regexp Build
=====================================

这个quiz的目的是给Regexp添加一个叫做build()的类方法，该方法要接受不定数量的参数，包括整数(Integers)和整数的区间(Ranges of Integers)。build将返回一个Regexp对象，该对象仅匹配参数中的整数。

下面是几个用法示例：

```ruby
lucky = Regexp.build(3, 7)
"7"     =~ lucky    # => true
"13"    =~ lucky    # => false 
"3"     =~ lucky    # => true

month = Regexp.build(1..12)
"0"     =~ month    # => false
"1"     =~ month    # => true
"12"    =~ month    # => true

day = Regexp.build(1..31)
"6"     =~ day      # => true
"16"    =~ day      # => true
"Tues"  =~ day      # => false

year = Regexp.build(98, 99, 2000..2005)
"04"    =~ year     # => false
"2004"  =~ year     # => true
"99"    =~ year     # => true

num = Regexp.build(0..1_000_000) 
"-1"	=~ num  	# => false
```

你可以决定你的库所产生的表达式形式。但你或许需要考虑几个问题：
* 怎样处理以0开头的？比如，你怎么匹配24小时制中的整点数字(0,23)，这里的0到9可能会以一个0开头。
* 返回的Regexp需要捕获什么？
* 锚定是如何工作的？(How should anchoring work)
```ruby
"2004" =~ Regexp.build(4)   # => ???
```


解答部分
-------------------------------


这个quiz要考虑的第一个问题是Regexp要匹配的数字应该表示成什么样子。看看用来匹配1到12最基本的样子：


<pre>
1|2|3|4|5|6|7|8|9|10|11|12
</pre>

注意，你可能会考虑是不是要把它反过来，除非你觉得可以依靠你自己的anchoring来匹配正确。

显而易见，前面这个实现方法很土鳖。这里有一个Tanaka Akira提交的漂亮得多得方法：

[regex_build/limited.rb](http://media.pragprog.com/titles/fr_quiz/code/regex_build/limited.rb)

```ruby
def Regexp.build(*args)
  args = args.map {|arg| Array(arg) }.flatten.uniq.sort
  neg, pos = args.partition {|arg| arg < 0 }
  / \A (?: -0*#{Regexp.union(*neg.map {|arg| (-arg).to_s })} |
            0*#{Regexp.union(*pos.map {|arg| arg.to_s })} ) \z /x
end
```

方法的头一行很聪明，对所有的参数调用Array。这样Range对象会变成等价的数组，而普通整数会打包成只包含他们自己的数组。接下来的flatten将产生一个包含所有我们需要匹配的Array。

第二行将参数分成正负两组。最后第三行用帅气的Regexp.union从分组中建立起Regexp对象。

这个解法能处理负数，并允许以零开头。

这个quiz真的就这么简单么？对于某些数据集，显然可以如此。然而，Tanaka的解法有其不足。在我这里，只要运行Regexp.build(1..10_100)就能得到一个“⋯⋯正则表达式太大”的错误。诚然，如果你的数据集很大，就需要深入探讨一下。

####压缩Regexp####

我们回头看看最初的问题，但多了个限制条件：最简短的用Regexp匹配一个数字的方法是什么？对于我们的模式，最显而易见的优化方法是使用字符类型。回来看看我们的1..12的例子，大概可以写成这样：

<pre>
\d|1[0-2]
</pre>
这样看起来合理多了。对于更复杂的例子，1..1_000_000就看起来会是这样：

<pre>
[1-9]|[1-9]\d|[1-9]\d\d|[1-9]\d\d\d|[1-9]\d\d\d\d|[1-9]\d\d\d\d\d|1000000
</pre>
专业一点，我们继续改进并得到像这样的结果：

<pre>
[1-9]\d{0,5}|1000000
</pre>
然而，这些方法都不够好。

建立字符类型最主要的问题是如何分解传入的Range对象。你也可以把单独的Interger参数也一起放进来，但这么做没什么意思。一个办法是对Range类添加一个方法将它们转换成Regexp对象。

在这个基础上添加Regexp.build就简单了。下面是Mark Hubbart的一个杰作：

[regex_build/grouped.rb](http://media.pragprog.com/titles/fr_quiz/code/regex_build/grouped.rb)

```ruby 
def Regexp.build(*args)
  ranges, numbers = args.partition{|item| Range === item}
  re = ranges.map{|r| r.to_re } + numbers.map{|n| /0*#{n}/ }
  /^#{Regexp.union(*re)}$/
end

class Range
  def to_re
    # normalize the range format; we want end inclusive, integer ranges 
    # this part passes the load off to a newly built range if needed. 
    if exclude_end?
      return( (first.to_i..last.to_i - 1).to_re ) 
    elsif ! (first + last).kind_of?(Integer)
      return( (first.to_i .. last.to_i).to_re ) 
    end
    # Deal with ranges that are wholly or partially negative. If range is 
    # only partially negative, split in half and get two regexen. join them 
    # together for the finish. If the range is wholly negative, make it 
    # positive, and then add a negative sign to the regexp 
    if first < 0 and last < 0
      # return a negatized version of the Regexp
      return /-#{(-last..-first).to_re}/ 
    elsif first < 0
      neg = (first..-1).to_re 
      pos = (0..last).to_re 
      return /(?:#{neg}|#{pos})/
    end

    ### First, create an array of new ranges that are more 
    ### suited to regex conversion.

    # a and z will be the remainders of the endpoints of the range
    # as we slice it
    a, z = first, last

    # build the first part of the list of new ranges.
    list1 = [] 
    num = first 
    until num > z
      a = num # recycle the value 
      # get the first power of ten that leaves a remainder      
      v = 10
      v *= 10 while num % v == 0 and num != 0
      # compute the next value up
      num += v - num % v
      # store the value unless it's too high
      list1 << (a..num-1) unless num > z
    end

    # build the second part of the list; counting down.
    list2 = [] 
    num = last + 1 
    until num < a
      z = num - 1 # recycle the value
      # slice to the nearest power of ten
      v = 10
      v *= 10 while num % v == 0 and num != 0
      # compute the next value down
      num -= num % v
      # store the value if it fits
      list2 << (num..z) unless num < a
    end
    # get the chewy center part, if needed
    center = a < z ? [a..z] : []
    # our new list
    list = list1 + center + list2.reverse

    ### Next, convert each range to a Regexp.
    list.map! do |rng| 
      a, z = rng.first.to_s, rng.last.to_s 
      a.split(//).zip(z.split(//)).map do |(f,l)|
        case 
          when f == l then f 
          when f.to_i + 1 == l.to_i then "[%s%s]" % [f,l] 
          when f+l == "09" then "\\d" 
          else
            "[%s-%s]" % [f,l] 
        end
      end.join # returns the Regexp for *that* range
    end

    ### Last, return the final Regexp
    /0*#{ list.join("|") }/
  end
end
```

前三个to_re就用来对付普通的Range对象，这里注释得很详细。

中间三个将Range分割成便于Regexp处理的块，每块中的数字长度是一样的。举例来说，下面是to_re分割1..1_000得到的局部变量list：

<pre>
1..9
10..99
100..999
1000..1000
</pre>

当然，表达式并不总是分成整十个整十个。还有另外一个例子，1_234..56_789产生下面的list：

<pre>
 1234..1239
 1240..1299
 1300..1999
 2000..9999
10000..49999
50000..55999
56000..56699
56700..56789
</pre>
最后三个to_re从Range对象中建立字符类型。在list.map! do ... end中的代码块十分漂亮，我建议你在弄明白后学习这种方法。

这个解法没有考虑以零开头的对象，并且返回的表达式只能匹配整行(and it is anchored at the beginning and end of a line, 见Regex.build方法最后一行)，同时支持负数。

####加速生成####
我们来看最后一个解法。和Mark Hubbart的方法很像，但正如我们之后会看到的，这个解法非常快。下面是Thomas Leitner的代码：

[regex_build/fast.rb](http://media.pragprog.com/titles/fr_quiz/code/regex_build/fast.rb)

```ruby
class Integer 
  def to_rstr 
    "#{self}"
  end 
end

class Regexp 
  def self.build( *args )
    Regexp.new("^(?:" + args.collect {|a| a.to_rstr}.flatten.uniq.join('|') + ")$") 
  end
end

class Range 
  def get_regexps( a, b, negative = false )
    arr = [a]
    af = (a == 0 ? 1.0 : a.to_f) 
    bf = (b == 0 ? 1.0 : b.to_f) 
    1.upto( b.to_s.length-1 ) do |i|
      pot = 10**i
      num = (af/pot).ceil*(pot) # next higher number with i zeros
      arr.insert( i, num ) if num < b
      num = (bf/pot).floor*(pot) # next lower number with i zeros 
      arr.insert( -i, num )
    end
    arr.uniq! 
    arr.push( b+1 ) # +1 -> to handle it in the same way as the other elements

    result = []
    0.upto( arr.length - 2 ) do |i|
      first = arr[i].to_s 
      second = (arr[i+1] - 1).to_s 
      str = ' ' 
      0.upto( first.length-1 ) do |j|
        if first[j] == second[j] 
          str << first[j]
        else
          str << "[#{first[j].chr}-#{second[j].chr}]" end
        end
      end
    result << str
    end

    result = result.join(' /')
    result = "-(?:#{result})" if negative
    result
  end

  def to_rstr
    if first < 0 &amp;&amp; last < 0
      get_regexps( -last, -first, true ) 
    elsif first < 0
      get_regexps( 1, -first, true ) + "|" + get_regexps( 0, last ) 
    else
      get_regexps( first, last )
    end 
  end
end
```

再提一下，这个解法和Mark Hubbart的很像。主要的工作由get_regexps处理。其中头一个upto将Range中的以10的幂分开。举例来说，Range 1..1_000_000将产生这样的arr：

<pre>
[0, 10, 100, 1000, 10000, 100000, 1000000, 1000001]
</pre>

第二个upto将arr中的数字组成Regexp字符类型。这一段代码的结果和上一个解法差不多，但由于处理了较少的信息并有效利用了数学手段来确定边界，使得运行起来快很多。

上面的解法无法捕捉开头的零，因而不能处理。负数是允许的。返回的表达式没有包括行首行尾，所以就必须匹配整个数。

####测定时间####
应用这些解法的一个重要问题是，对于建立完一个Regexp你需要等待多长时间，以及这个结果找到一个匹配有多快。下面是建立用时的得分：

<pre>
                    user      system      total           real
Tanaka Akira    26.370000   0.150000    26.520000   ( 26.624490)
Mark Hubbart     9.890000   0.040000     9.930000   (  9.944374)
Thomas Leitner   5.270000   0.030000     5.300000   (  5.323440)
</pre>
这些数据是由以下的代码测量的：

[regex_build/build_times.rb](http://media.pragprog.com/titles/fr_quiz/code/regex_build/build_times.rb)

```ruby
#!/usr/local/bin/ruby
require "benchmark"

Benchmark.bm(16) do |stats| 
  { "Mark Hubbart"	  => "grouped",
    "Thomas Leitner"  => "fast", 
    "Tanaka Akira"    => "limited" }.each do |name, library| 
    require library 
    stats.report(name) do
      50_000.times { Regexp.build(1, 2, 5..100) }
    end 
  end
end
```

之后，下面是匹配的测试(不计算建立耗时)：

<pre>
                    user      system      total           real
Tanaka Akira    19.090000   0.040000    19.130000   ( 19.151167)
Mark Hubbart     0.010000   0.000000     0.100000   (  0.105672)
Thomas Leitner   0.010000   0.000000     0.100000   (  0.098846)
</pre>

以及代码：

[regex_build/match_times.rb](http://media.pragprog.com/titles/fr_quiz/code/regex_build/match_times.rb)

```ruby
#!/usr/local/bin/ruby
require "benchmark"

Benchmark.bm(16) do |stats| 
  { "Mark Hubbart"	  => "grouped",
    "Thomas Leitner"  => "fast", 
    "Tanaka Akira"    => "limited" }.each do |name, library| 
    require library regex = Regexp.build(1..5_000) 
    stats.report(name) do
      50_000.times { "4098" =~ regex } 
    end
  end 
end
```

####补充练习####
1.设计一个算法把Regexp变换成更小的表达形式，比如用[1-9]\d{0,5}|1000000表示1..1_000_000
2.把前面的测试用库代码应用到压缩表达式和你的原始解法上，还有quiz里的这些，比比看怎么样
3.让你的代码能够处理非数字的输入，这样你就调用像Regexp.build("cat", "bat", "rat", "dog"}这样的代码

