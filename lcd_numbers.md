LCD 数字
=======

这个quiz的目标是写个程序来显示指定大小的LCD风格的数字。(译注：对于此题参见[poj第1102题](http://poj.org/problem?id=1102)，hoj此题测试数据格式有误)

需要显示的数字作为参数传递给程序。用带-s选项的正整数作为命令行参数来控制数字大小。默认的值-s值是2。

举例来说，如果你的程序这么调用：

```shell
$ lcd.rb 012345
```

正确的显示应该是这样：

<pre>
 --        --   --        --  
|  |    |    |    | |  | |    
|  |    |    |    | |  | |    
           --   --   --   --  
|  |    | |       |    |    | 
|  |    | |       |    |    | 
 --        --   --        --  
</pre>


对于这个调用：

```shell
$ lcd.rb -s 1 6789
```

你的程序应该这么输出：


<pre>
 -   -   -   -  
|     | | | | | 
 -       -   -  
| |   | | |   | 
 -       -   - 
</pre>


注意在两个数字间还有个空列。给-s其他的值，只需要扩展横线-和竖线|。


解答部分
---------------------



显而易见这个问题没什么难度。Hao (David) Tran 有个少于300字节的变态解法（不在这里展示）。不管他是不是真这么简单，这个经典的问题确实涵盖了像缩放组合多行数据这样的内容，这些内容可以在许多计算机编程领域遇到。

####使用模板####

对这个问题我见过三种主要的解决策略。其一是使用模板，将这些数字表示成每节一个短线的文本。数字2看起来差不多这样：

```ruby
[ " - ",
  "  |",
  " - ",
  "|  ",
  " - " ]
```

把数字放大到任意大小包含两个扩展过程。首先，你需要将它水平拉伸。简单的办法就是提取每行第二个字符（一个"-"或者空格）并将其重复-s指定的次数。

```ruby
digit.each { |row| row[1, 1] *= scale }
```

之后，需要把数字纵向放大。如果你高兴甚至可以在打印阶段处理。只需把任意包含竖线|的行打印-s指定的次数。

```ruby
digit.each do |row|
  if row.include? "|"
    scale.times { puts row }
  else
    puts row
  end
end
```
把上面这些想法结合在一起，就得到了下面的完整解法。

[lcd_numbers/template.rb](http://media.pragprog.com/titles/fr_quiz/code/lcd_numbers/template.rb)

```ruby
# template
DIGITS = << END_DIGITS.split("\n").map { |row| row.split(" # ") }.transpose
 -  #     #  -  #  -  #     #  -  #  -  #  -  #  -  #  - 
| | #   | #   | #   | # | | # |   # |   #   | # | | # | |
    #     #  -  #  -  #  -  #  -  #  -  #     #  -  #  - 
| | #   | # |   #   | #   | #   | # | | #   | # | | #   |
 -  #     #  -  #  -  #     #  -  #  -  #     #  -  #  - 
END_DIGITS

# number scaling (horizontally and vertically)
def scale( num, size )
  bigger = [ ] 
  num.each do |line|
    row = line.dup
    row[1, 1] *= size 
    if row.include? "|"
      size.times { bigger << row }
    else
      bigger << row
    end 
  end
  bigger
end

# option parsing
s=2 
if ARGV.size >= 2 and ARGV[0] == '-s' and ARGV[1] =~ /^[1-9]\d*$/
  ARGV.shift
  s = ARGV.shift.to_i
end

# digit parsing/usage
unless ARGV.size == 1 and ARGV[0] =~ /^\d+$/ 
  puts "Usage: #$0 [-s SIZE] DIGITS" 
  exit
end
n = ARGV.shift

# scaling
num = [ ] 
n.each_byte do |c|
  num << [" "] * (s * 2 + 3) if num.size > 0
  num << scale(DIGITS[c.chr.to_i], s)
end

# output
num = ([""] * (s * 2 + 3)).zip(*num) 
num.each { |l| puts l.join }
```

####开关位####

第二个策略是把每个数字看作可以调整为“开”或“关”的一些段。数字可以简单的分成七个部分：

```ruby
  6 
5   4
  3  
2   1
  0
```

使用这种表示法，我们可以把数字2的表示转化为二进制：

```ruby
0b1011101
```

这种表示法的处理方式和前面的方法差不多。下面是Florian Gro&beta的完整的解题代码：

[lcd_numbers/bits.rb](http://media.pragprog.com/titles/fr_quiz/code/lcd_numbers/bits.rb)

```ruby
module LCD
  extend self

  # Digits are represented by simple bit masks. Each bit identifies 
  # whether a line should be displayed. The following ASCII 
  # graphic shows the mapping from bit position to the belonging line. 
  # 
  #   =6 
  # 5    4 
  #   =3 
  # 2    1 
  #   =0

  Digits = [0b1110111, 0b0100100, 0b1011101, 0b1101101, 0b0101110,
            0b1101011, 0b1111011, 0b0100101, 0b1111111, 0b1101111,
            0b0001000, 0b1111000] # Minus, Dot
  Top, TopLeft, TopRight, Middle, BottomLeft, BottomRight, Bottom = *0 .. 6
  SpecialDigits = { "-" => 10, "." => 11 }

  private
  def line(digit, bit, char = "|") 
    (digit & 1 << bit).zero? ? " " : char
  end

  def horizontal(digit, size, bit)
    [" " + line(digit, bit, "-") * size + " "]
  end

  def vertical(digit, size, left_bit, right_bit)
    [line(digit, left_bit) + " " * size + line(digit, right_bit)] * size
  end

  def digit(digit, size)
    digit = Digits[digit.to_i]
    horizontal(digit, size, Top) + 
    vertical(digit, size, TopLeft, TopRight) + 
    horizontal(digit, size, Middle) + 
    vertical(digit, size, BottomLeft, BottomRight) + 
    horizontal(digit, size, Bottom)
  end

  public

  def render(number, size = 1) 
    number = number.to_s
    raise(ArgumentError, "size has to be > 0") unless size > 0 
    raise(ArgumentError, "Invalid number") unless number[/\A[\d.-]+\Z/]

    number.scan(/./).map do |digit| 
      digit(SpecialDigits[digit] || digit, size)
    end.transpose.map do |line| 
      line.join(" ")
    end.join("\n") 
  end
end

if __FILE__ == $0 
  require 'optparse' 
  options = { :size => 2 } 
  number = ARGV.pop

  ARGV.options do |opts| 
    script_name = File.basename($0) 
    opts.banner = "Usage: ruby #{script_name} [options] number"

    opts.separator ""

    opts.on("-s", "-size size", Numeric, 
          "Specify the size of line segments.", 
          "Default: 2"
    ) { |options[:size]| }

opts.separator ""
opts.on("-h", "-help", "Show this help message.") { puts opts; exit }
opts.parse!
end
```

无论哪种方法，都需要将缩放的数字组合在一起输出。不过是在做二维的join()操作。这样的事情可以简单调用Array.zip或者Array.transpose来做。

####使用状态机####

最后，一种独特的策略是引入状态机。我们来看看Dale Martenson解法的主要类构成。

[lcd_numbers/states.rb](http://media.pragprog.com/titles/fr_quiz/code/lcd_numbers/states.rb)

```ruby
class LCD

  # This hash defines the segment display for the given digit. Each 
  # entry in the array is associated with the following states:
  #
  # HORIZONTAL
  # VERTICAL
  # HORIZONTAL
  # VERTICAL
  # HORIZONTAL
  # DONE
  #
  # The HORIZONTAL state produces a single horizontal line. There
  # are two types:
  #
  # 0 - skip, no line necessary, just space fill
  # 1 - line required of given size
  #
  # The VERTICAL state produces either a single right side line,
  # a single left side line or both lines.
  #
  # 0 - skip, no line necessary, just space fill
  # 1 - single right side line
  # 2 - single left side line
  # 3 - both lines
  #
  # The DONE state terminates the state machine. This is not needed
  # as part of the data array.
  LCD_DISPLAY_DATA = {
    "0" => [ 1, 3, 0, 3, 1 ], 
    "1" => [ 0, 1, 0, 1, 0 ], 
    "2" => [ 1, 1, 1, 2, 1 ], 
    "3" => [ 1, 1, 1, 1, 1 ], 
    "4" => [ 0, 3, 1, 1, 0 ], 
    "5" => [ 1, 2, 1, 1, 1 ], 
    "6" => [ 1, 2, 1, 3, 1 ], 
    "7" => [ 1, 1, 0, 1, 0 ], 
    "8" => [ 1, 3, 1, 3, 1 ], 
    "9" => [ 1, 3, 1, 1, 1 ]
  }

  LCD_STATES = {
    "HORIZONTAL",
    "VERTICAL",
    "HORIZONTAL",
    "VERTICAL",
    "HORIZONTAL",
    "DONE"
  }

  attr_accessor :size, :spacing

  def initialize( size=1, spacing=1 )
    @size = size
    @spacing = spacing
  end

  def display( digits ) 
    states = LCD_STATES.reverse 
    0.upto(LCD_STATES.lengt;h) do |i|
      case states.pop 
      when "HORIZONTAL"
        line = "" 
        digits.each_byte do |b|
          line += horizontal_segment( LCD_DISPLAY_DATA[b.chr][i] )
        end
        print line + "\n" 
      when "VERTICAL"
        1.upto(@size) do |j| 
          line = "" 
          digits.each_byte do |b|
            line += vertical_segment( LCD_DISPLAY_DATA[b.chr][i] )
          end
          print line + "\n" 
        end
      when "DONE" 
        break
      end 
    end
  end

  def horizontal_segment( type ) 
    case type 
    when 1
      return " " + ("-" * @size) + " " + (" " * @spacing) 
    else
      return " " + (" " * @size) + " " + (" " * @spacing) 
    end
  end

  def vertical_segment( type ) 
    case type when 1
      return " " + (" " * @size) + "|" + (" " * @spacing) 
    when 2
      return "|" + (" " * @size) + " " + (" " * @spacing) 
    when 3
      return "|" + (" " * @size) + "|" + (" " * @spacing) 
    else
      return " " + (" " * @size) + " " + (" " * @spacing) 
    end
  end 
end
```

LCD类开头的注释能给你很大的帮助去理解该类的作用。这个类表示了一个状态机。对于某个需要的大小(在initialize中设定)，该类将在一系列的状态中变化(在LCD_STATES中定义)。在每个状态中，按照需求建立横向或纵向的短线(在horizontal_segment和vertical_segment中)。

这个方法的好处就是可以简单的一次处理一行输出，像在display中所示。所有数字最上面一行，由第一个HORIZONTAL状态产生，并立即输出，之后的行类似。这个资源友好(resource-friendly)的系统可以轻易的扩展到更大的输入。

Dale的代码的其他部分是命令行参数处理和对display的调用：

[lcd_numbers/states.rb](http://media.pragprog.com/titles/fr_quiz/code/lcd_numbers/states.rb)

```ruby
require 'getoptlong'

opts = GetoptLong.new(
  [ "-size", "-s", GetoptLong::REQUIRED_ARGUMENT ], 
  [ "-spacing", "-sp", "-p", GetoptLong::REQUIRED_ARGUMENT ]
)

lcd = LCD.new

opts.each do |opt, arg| 
  case opt 
  when "-size"	then lcd.size = arg.to_i 
  when "-spacing" then lcd.spacing = arg.to_i end
end

lcd.display( ARGV.shift )
```

####补充练习####

1.修改你的解法，使之在建立一行的时候就输出，而不是等到整个数字完成。

2.扩展Florian Groß的解法，增加十六进制数字A到F
