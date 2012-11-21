罗马数字
============================================

这次的题目是要你写一个普通数字和罗马数字的转换器。

这个脚本应该是个标准Unix过滤器，从命令行指定的文件或者`STDIN`读入并写到`STDOUT`。每行输入包括一个用阿拉伯或者罗马数字表示的整数(从1到3,999)。对应每行输入应该有一行输出，内容是原始数的另一种表达方式。

举例来说，对应下面的输入：

<pre>
III
29
38
CCXCI
1999
</pre>

正确的输出应该是这样：

<pre>
3
XXIX
XXXVIII
291
MCMXCIX
</pre>

如果你对罗马数字不太熟悉或者需要复习一下，很简单。首先，七个字母对应于七个值：

<pre>
I = 1 
V = 5
X = 10
L = 50
C = 100 
D = 500 
M = 1000
</pre>

其次，你可以将字母由大到小从左到右组合在一起来表示相加的值：

<pre>
II    is 2
VIII  is 8
XXXI  is 31
</pre>

然而，你最多只能将同一个字母连续使用三次。这样的话就需要一条特殊规则来表示像40和900这样的数。规则就是在一个较大的值前面加上一个较小的值来表示减少多少。这条规则用来表示先前的规则不能表示的数。像这些：

<pre>
IV is 4
IX is 9
XL is 40
XC is 90
CD is 400
CM is 900
</pre>

解答部分
---------------------------------------

解决这个问题很简单，到底多简单呢？这样说吧，问题已经给我们了一个转换表，可以轻松表示成一个`Hash`：

[roman_numerals/simple.rb](http://media.pragprog.com/titles/fr_quiz/code/roman_numerals/simple.rb)

```ruby
ROMAN_MAP = { 1     => "I",
              4     => "IV",
              5     => "V",
              9     => "IX",
              10    => "X",
              40    => "XL", 
              50    => "L", 
              90    => "XC", 
              100   => "C", 
              400   => "CD", 
              500   => "D", 
              900   => "CM", 
              1000  => "M" }
```

这就是我的代码，而大部分的解法都差不多如此。

这样的话我们就只需要`to_roman()`和`to_arabic()`方法了，对吧？对我这样的懒骨头听起来有好多工作吖，所以我还是作弊好了。如果你有了一个转换表，那么你就能单向转换了。

[roman_numerals/simple.rb](http://media.pragprog.com/titles/fr_quiz/code/roman_numerals/simple.rb)

```ruby
ROMAN_NUMERALS = Array.new(3999) do |index|
  target = index + 1
  ROMAN_MAP.keys.sort { |a, b| b <=> a }.inject("") do |roman, div|
    times, target = target.divmod(div)
    roman << ROMAN_MAP[div] * times
  end
end
```

这就是在许多解法中都差不多的`to_roman()`方法。我的办法就是填充一个数组。算法本身没啥难度。用每个罗马数字去除阿拉伯数得到的商，重复该罗马数字这么多次。Ruby的`divmod()`方法正好适合这个工作。

现在只需要加一层Unix接口了。但我还是喜欢先验证一下输入，于是就多写了一点正则工具：

[roman_numerals/simple.rb](http://media.pragprog.com/titles/fr_quiz/code/roman_numerals/simple.rb)

```ruby
IS_ROMAN = / ^ M{0,3}
               (?:CM|DC{0,3}|CD|C{0,3})
               (?:XC|LX{0,3}|XL|X{0,3})
               (?:IX|VI{0,3}|IV|I{0,3}) $ /ix
IS_ARABIC = /^(?:[123]\d{3}|[1-9]\d{0,2})$/
```
第一个正则是验证罗马数字的，以十的幂分开。第二个正则匹配1..3999，这是我们能够转换的区间。

现在，可以写Unix接口了：

[roman_numerals/simple.rb](http://media.pragprog.com/titles/fr_quiz/code/roman_numerals/simple.rb)

```ruby
if __FILE__ == $0
  ARGF.each_line() do |line|
    line.chomp!
    case line
    when IS_ROMAN  then puts ROMAN_NUMERALS.index(line) + 1
    when IS_ARABIC then puts ROMAN_NUMERALS[line.to_i - 1]
    else raise "Invalid input:  #{line}"
    end
  end
end
```
用普通话来表述就是，对于每行输入，如果能匹配`IS_ROMAN`，就在`Array`里查找。如果不能匹配`IS_ROMAN`而能匹配`IS_ARABI`，就在`Array`中直接索引该值。如果两项都不匹配，就报告输入错误。

####省点内存####

如果你不想遍历整个数组，只需要创建另外一套转换器，这也不难。J E Bailey的代码就做到了，我们来看看：

[roman_numerals/dual_conversions.rb](http://media.pragprog.com/titles/fr_quiz/code/roman_numerals/dual_conversions.rb)

```ruby
@data = [
["M"  , 1000],
["CM" , 900],
["D"  , 500],
["CD" , 400],
["C"  , 100],
["XC" ,  90],
["L"  ,  50],
["XL" ,  40],
["X"  ,  10],
["IX" ,   9],
["V"  ,   5],
["IV" ,   4],
["I"  ,   1]
]

@roman = %r{^[CDILMVX]*$}
@arabic = %r{^[0-9]*$}

def to_roman(num)
  reply = ""
  for key, value in @data
    count, num = num.divmod(value)
    reply << (key * count)
  end
  reply
end

def to_arabic(rom)
  reply = 0
  for key, value in @data
    while rom.index(key) == 0
      reply += value
      rom.slice!(key)
    end
  end
  reply
end

$stdin.each do |line|
  case line
  when @roman
    puts to_arabic(line)
  when @arabic
    puts to_roman(line.to_i)
  end
end
```
告诉过你我们总会用到类似我的`Hash`的东西。这里就是个一对一对的数组。

这下面可以看到J E验证数据的正则表达式。和我的不太一样，但显然更清楚明了。

接下来是`to_roman()`方法，看起来很眼熟。这个实现和我的几乎一模一样，但由于没有引入数组而更清楚一点。

再下来是个有趣的部分，`to_arabic()`。该方法上来就将一个叫`reply`的变量赋值为0。然后依次遍历`rom`字符串中的每个罗马数字，将`reply`增加对应值，并将该数字从字符串中移除。`@data`数组的顺序保证了XL或者IV将在X或者I之前处理。

最后，代码给出了quiz指定的Unix接口。这又和我的解法很像了，也有两种转换方式分别进行。

罗马ruby
-------------------------------

上面都是小儿科，我们还是看看Dave Burt的Ruby魔术。Dave的代码包含了一个模块，`RomanNumerals`，以及和我们之前看过的很相似的`to_integer()`和`from_integer()`。该模块还定义了`is_roman_numeral?()`方法来检查输入，另外还有些有用的常量，如`DIGITS`，`MAX`和`REGEXP`。

[roman_numerals/roman_numerals.rb](http://media.pragprog.com/titles/fr_quiz/code/roman_numerals/roman_numerals.rb)

```ruby
# Contains methods to convert integers to Roman numeral strings, and vice versa.
module RomanNumerals
  
  # Maps Roman numeral digits to their integer values
  DIGITS = {
    'I' => 1,
    'V' => 5,
    'X' => 10,
    'L' => 50,
    'C' => 100,
    'D' => 500,
    'M' => 1000
  }
  
  # The largest integer representable as a Roman numerable by this module
  MAX = 3999
  
  # Maps some integers to their Roman numeral values
  @@digits_lookup = DIGITS.inject({
    4 => 'IV',
    9 => 'IX',
    40 => 'XL',
    90 => 'XC',
    400 => 'CD',
    900 => 'CM'}) do |memo, pair|
    memo.update({pair.last => pair.first})
  end
  
  # Based on Regular Expression Grabbag in the O'Reilly Perl Cookbook, #6.23
  REGEXP = /^M*(D?C{0,3}|C[DM])(L?X{0,3}|X[LC])(V?I{0,3}|I[VX])$/i
  
  # Converts +int+ to a Roman numeral
  def self.from_integer(int)
    return nil if int < 0 || int > MAX
    remainder = int
    result = ''
    @@digits_lookup.keys.sort.reverse.each do |digit_value|
      while remainder >= digit_value
        remainder -= digit_value
        result += @@digits_lookup[digit_value]
      end
      break if remainder <= 0
    end
    result
  end
  
  # Converts +roman_string+, a Roman numeral, to an integer
  def self.to_integer(roman_string)
    return nil unless roman_string.is_roman_numeral?
    last = nil
    roman_string.to_s.upcase.split(//).reverse.inject(0) do |memo, digit|
      if digit_value = DIGITS[digit]
        if last && last > digit_value
          memo -= digit_value
        else
          memo += digit_value
        end
        last = digit_value
      end
      memo
    end
  end
  
  # Returns true if +string+ is a Roman numeral.
  def self.is_roman_numeral?(string)
    REGEXP =~ string
  end
end
```

我觉得我们应该再过一下这段代码，但我想先指出一个闪光点。 注意看Dave是如何以一种优雅的方式将IV这样的数从`DIGITS`中剥离的。其中用了个不寻常的写法`memo.update({pair.last=>pair.first})`，而非看起来更常见的`memo[pair.last]=pair.first`。是因为前者返回`Hash`本身，而`inject()`循环是需要不断更新这个变量的。

好了，这个模块只是Dave代码的一部分，剩下的也很有意思。来看看他是如何使用该模块的：

[roman_numerals/roman_numerals.rb](http://media.pragprog.com/titles/fr_quiz/code/roman_numerals/roman_numerals.rb)

```ruby
class String
  # Considers string a Roman numeral,
  # and converts it to the corresponding integer.
  def to_i_roman
    RomanNumerals.to_integer(self)
  end
  # Returns true if the subject is a Roman numeral.
  def is_roman_numeral?
    RomanNumerals.is_roman_numeral?(self)
  end
end
class Integer
  # Converts this integer to a Roman numeral.
  def to_s_roman
    RomanNumerals.from_integer(self) || ''
  end
end
```

首先，他给`String`和`Integer`类添加了转换方法。然后你就能编写像如下的代码了：

```ruby
puts "In the year #{1999.to_s_roman} ..."
```

有意思吧，更好玩的在后面。Dave的究极奥义是定义了这样一个类：

[roman_numerals/roman_numerals.rb](http://media.pragprog.com/titles/fr_quiz/code/roman_numerals/roman_numerals.rb)

```ruby
# Integers that look like Roman numerals
class RomanNumeral
  attr_reader :to_s, :to_i

  @@all_roman_numerals = []

  # May be initialized with either a string or an integer
  def initialize(value)
    case value
    when Integer
      @to_s = value.to_s_roman
      @to_i = value
    else
      @to_s = value.to_s
      @to_i = value.to_s.to_i_roman
    end
    @@all_roman_numerals[to_i] = self
  end

  # Factory method: returns an equivalent existing object if such exists,
  # or a new one
  def self.get(value)
    if value.is_a?(Integer)
      to_i = value
    else
      to_i = value.to_s.to_i_roman
    end
    @@all_roman_numerals[to_i] || RomanNumeral.new(to_i)
  end

  def inspect
    to_s
  end

  # Delegates missing methods to Integer, converting arguments to Integer,
  # and converting results back to RomanNumeral
  def method_missing(sym, *args)
    unless to_i.respond_to?(sym)
      raise NoMethodError.new(
        "undefined method '#{sym}' for #{self}:#{self.class}")
    end
    result = to_i.send(sym,
      *args.map {|arg| arg.is_a?(RomanNumeral) ? arg.to_i : arg })
    case result
    when Integer
      RomanNumeral.get(result)
    when Enumerable
      result.map do |element|
        element.is_a?(Integer) ? RomanNumeral.get(element) : element
      end
    else
      result
    end
  end
end



# Enables uppercase Roman numerals to be used interchangeably with integers.
# They are autovivified RomanNumeral constants
# Synopsis:
#   4 + IV           #=> VIII
#   VIII + 7         #=> XV
#   III ** III       #=> XXVII
#   VIII.divmod(III) #=> [II, II]
def Object.const_missing sym
  unless RomanNumerals::REGEXP === sym.to_s
    raise NameError.new("uninitialized constant: #{sym}")
  end
  const_set(sym, RomanNumeral.get(sym))
end



# Quiz solution: filter that swaps Roman and arabic numbers
if __FILE__ == $0
  ARGF.each do |line|
    line.chomp!
    if line.is_roman_numeral?
      puts line.to_i_roman
    else
      puts line.to_i.to_s_roman
    end
  end
end
```
用工厂方法`get()`来创建对象会很高效且可复用，它对于同样的数值总会给出同一个对象。

注意`method_missing()`方法最终会把处理委托给`Integer`类，所以你可以将这些对象当做`Integer`实例来看。这个类能让你写出如下的代码：

```ruby
IV = RomanNumeral.get(4)
IV + 5 # => IX
```

更神奇的是，多亏Dave，连第一步的代码都不需要了：

[roman_numerals/roman_numerals.rb](http://media.pragprog.com/titles/fr_quiz/code/roman_numerals/roman_numerals.rb)

```ruby
# Enables uppercase Roman numerals to be used interchangeably with integers.
# They are autovivified RomanNumeral constants
# Synopsis:
#   4 + IV           #=> VIII
#   VIII + 7         #=> XV
#   III ** III       #=> XXVII
#   VIII.divmod(III) #=> [II, II]
def Object.const_missing sym
  unless RomanNumerals::REGEXP === sym.to_s
    raise NameError.new("uninitialized constant: #{sym}")
  end
  const_set(sym, RomanNumeral.get(sym))
end
```

这样的话Ruby在必要时就会自动将`IX`这样的常量当做`RomanNumeral`对象来看待了，太爽了。

最后，（原书）下一页顶端是Dave应用上面这些工具对这个问题的解决过程。

[roman_numerals/roman_numerals.rb](http://media.pragprog.com/titles/fr_quiz/code/roman_numerals/roman_numerals.rb)

```ruby
# Quiz solution: filter that swaps Roman and arabic numbers
if __FILE__ == $0
  ARGF.each do |line|
    line.chomp!
    if line.is_roman_numeral?
      puts line.to_i_roman
    else
      puts line.to_i.to_s_roman
    end
  end
end
```

####补充练习####
1.修改你的代码，使之能扫描任何文本文档，将其中所有的合法罗马数替换成对应的阿拉伯数。

2.重新写一个类似于第一种解法的方案，但不能在内存中维护一个有4000个元素的数组。
