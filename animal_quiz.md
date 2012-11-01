猜动物
======

这个Ruby问题是写一个猜动物的程序。

该问题的流程是这样的：开始时程序告诉用户在脑子里想一种动物。之后就问一系列关于该动物的是否问题：它会游泳么？它有头发么？诸如此类⋯⋯ 最终，程序会缩小可能的范围直到某一种动物，并做出猜测：它是只老鼠么？

如果程序猜测正确，游戏结束并开始猜一种新动物。如果程序猜错了，它会询问用户他们所想的是什么动物，并要求用户给出一个问题，能用来区别该动物和程序错猜的动物。然后程序就会添加一个新问题和动物到它的“数据库”，并在之也能猜测该动物(如果适当的话)。不管运行多少次，你的程序都应该能记住它学过了什么。

下面是一个运行样例：

<pre>
⇒  Think of an animal... 
    Is it an elephant? (y or n)
⇐  n 
⇒  You win. 
    Help me learn from my mistake before you go...
    What animal were you thinking of?
⇐  a rabbit 
⇒  Give me a question to distinguish a rabbit from an elephant.
⇐  Is it a small animal?
⇒  For a rabbit, what is the answer to your question? (y or n) 
⇐  y 
⇒  Thanks.
    Play again? (y or n)
⇐  y 
⇒  Think of an animal...
    Is it a small animal? (y or n)
⇐  y 
⇒  Is it a rabbit? (y or n) 
⇐  n 
⇒  You win. Help me learn from my mistake before you go...
    What animal were you thinking of?
⇐  a Shih Tzu 
⇒  Give me a question to distinguish a Shih Tzu from a rabbit. 
⇐  Is it a kind of dog? 
⇒  For a Shih Tzu, what is the answer to your question? (y or n) 
⇐  y 
⇒  Thanks.
    Play again? (y or n)
⇐  y 
⇒  Think of an animal...
    Is it a small animal? (y or n)
⇐  y 
⇒  Is it a kind of dog? (y or n) 
⇐  y
⇒  Is it a Shih Tzu? (y or n) 
⇐  y 
⇒  I win.	Pretty smart, aren' t I?
    Play again? (y or n)
⇐  n
</pre>

解答部分
----------------------------------------------------------

基本上每个人都用差不多的方式解决这个问题。Jim Weirch阐述了这种思路：

一个简单的表示数据库的办法是使用二叉树，将问题表示成内节点，而动物作为叶子节点。每个问题节点有两个子节点对应答案"yes"或"no"。子节点的内容是进一步的问题(接下来要问的)或者一种动物(接下来要猜的)。

[animal_quiz/animal.rb](http://media.pragprog.com/titles/fr_quiz/code/animal_quiz/animals.rb)

```ruby
#!/usr/bin/env ruby
require 'yaml'
require 'ui'

def ui
  $ui ||= ConsoleUi.new
end

class Question
  def initialize(question, yes, no)
    @question = question
    @yes = yes
    @no = no
    @question << "?" unless @question =~ /\?$/
    @question.sub!(/^([a-z])/) { $1.upcase }
  end

  def walk
    if ui.ask_if @question
      @yes = @yes.walk
    else
      @no = @no.walk
    end
    self
  end
end

class Animal
  attr_reader :name
  def initialize(name)
    @name = name
  end

  def walk
    if ui.ask_if "Is it #{an name}?"
      ui.say "Yea! I win!\n\n"
      self
    else
      ui.say "Rats, I lose"
      ui.say "Help me play better next time."
      new_animal = ui.ask "What animal were you thingking of?"
      question = ui.ask "Give me a question " +
                        "to distinguish a #{an name} from #{an new_animal}"
      response = ui.ask_if  "For #{an new_animal}, " +
                            "the answer to your question would be?"
      ui.say "Thank you\n\n"
      if response
        Question.new(question, Animal.new(new_animal), self)
      else
        Question.new(question, self, Animal.new(new_animal))
      end
    end
  end

  def an(animal)
    ((animal =~ /^[aeiou]/) ? "an " : "a ") + animal
  end
end

if File.exist? "animals.yaml"
  current = open("animals.yaml") { |f| YAML.load(f.read) }
else
  current = Animal.new("mouse")
end

loop do 
  current = current.walk
  break unless ui.ask_if "Play again?"
  ui.say "\n\n"
end

open("animals.yaml", "w") do |f| f.puts current.to_yaml end
```

这是一种很直接的方法。在一开始，引入YAML来存储数据(许多人都这么做)，以及一个ui库来处理界面。还为ui库来处理界面定义了一个辅助方法，这样只需改变一个全局变量就能轻易调整整个界面。

现在跳过类定义，看看"main"部分。头一段在可能的情况下载入一个已有的动物树文件。否则就创建一个新的。

中间一段访问树上的节点，当有新节点建立的时候储存该结果。接下利用ui的辅助方法来询问用户是否还要玩这个游戏，

最后一句使用YAML来存储本次运行之后的树。

为了弄明白上面讨论的这个“树”，你要回头看看那两个类。像我们描述的思路那样，Question对象表示一个问题本身，以及到答案节点的连接"yes"和"no"。重要的方法是Question.walk(不要和我们接下来要说的Animal.walk搞混了)。walk方法通过ui来询问问题并根据答案递归进入@yes.walk或@no.walk。值得注意的技巧是，返回结果还可以是节点本身。这允许节点在学习到新动物后更自身。

剩下就是Animal了，这个更简单。一样的，重要的方法是Animal.walk。walk方法通过ui来猜测动物并判断是否正确。如果错了，就询问能分辨动物区别的问题，并把自身和一个新动物打包在一个Question对象里返回。通过Question.walk的自我更新，来保证该树的更新。

下来就只有神秘的ui库了。我们来看看：

[animal_quiz/ui.rb](http://media.pragprog.com/titles/fr_quiz/code/animal_quiz/ui.rb)

```ruby
#!/usr/bin/env ruby

class ConsoleUi
  def ask(prompt)
    print prompt + " "
    answer = gets
    answer ? answer.chomp : nil
  end

  def ask_if(prompt)
    answer = ask(prompt)
    answer =~ /^\s*[Yy]/
  end

  def say(*msg)
    puts msg
  end
end
```

当然，这就是界面了。ask处理输入，say处理输出，而ask_if是依据用户答案的"yes"或"no"来返回真假的辅助方法(能在if条件中处理，像名字所示那样)。这些方法也可以用等价的CGI，GUI，或其他什么来替代。这是个很棒的抽象机制。

####用数组替代自定对象####
上面是一个对该树的很漂亮的面向对象抽象，但你也可以用简单的数据结构来处理问题。我们来看看Markus König使用嵌套数组的一个解法。

[animal_quiz/array.rb](http://media.pragprog.com/titles/fr_quiz/code/animal_quiz/array.rb)

```ruby
#! /usr/bin/env ruby

def ask(prompt)
  loop do
    print prompt
    $stdout.flush
    s = gets
    exit if s == nil
    s.chomp!
    if s == 'y' or s == 'yes'
      return true
    elsif s == 'n' or s == 'no'
      return false
    else
      $stderr.puts "Please answer yes or no."
    end
  end
end

def my_readline
  s = gets
  exit if s == nil
  s.chomp!
  return s
end

class AnimalQuiz
  DEFAULT_ANSWERS = ['an elephant']
  
  def initialize(filename)
    if not filename
      @answers = DEFAULT_ANSWERS
    else
      begin
        File.open(filename) do |f|
          @answers = eval(f.read)
        end
      rescue Errno::ENOENT
        @answers = DEFAULT_ANSWERS
      end
    end
  
    @current = nil
  end

  def save(filename)
    File.open(filename, 'w') do |f|
      f.puts @answers.inspect
    end
  end

  def run_once
    unless @current
      @current = @answers
      puts 'Think of an animal...'
    end
  
    if @current.length == 1
      if ask("Is it #{@current[0]}?")
        puts ' I win! Pretty smart, aren\' t I?' 
      else
        print ' You win! Help me learn from my ' 
        puts ' mistake before you go...' 
        puts ' What animal were you thinking of?' 
        correct = my_readline
        incorrect = @current[0] 
        print ' Give me a question to distinguish ' 
        puts "#{correct} from #{incorrect}." 
        question = my_readline 
        if ask("For #{correct}, what is the" \
          + ' answer to your question?') 
          @current.push [correct] 
          @current.push [incorrect]
        else
          @current.push [incorrect]
          @current.push [correct]
        end
        @current[0] = question
      end
      if ask(' Play again?' )
        @current = nil
        puts
      else
        exit
      end 
    elsif @current.length == 3
      if ask(@current[0]) 
        @current = @current[1]
      else
        @current = @current[2]
      end 
    end
  end

  def run 
    loop {run_once}
  end 
end

filename = ENV[' HOME' ] + ' /.animal-quiz'

quiz = AnimalQuiz.new(filename)

begin
  quiz.run
ensure
  quiz.save filename
end      
```

这段代码以一组方法开始。首先是ask，从用户终端那里得到一个有效的"yes"或"no"答案。值得注意的是该方法如何循环直到给出一个有效的答案，并使用return来跳出循环。

另一个方法，my_readline，是个自动chomp版本的gets。使得之后询问用户问题更简单。

AnimalQuiz类包含了主要的解决办法。首先建立一个默认的答案树，即一个简单的Array。你能看出来initialize试图加载一个之前存储的树，当行不通时则建立一个默认的(很可能是首次运行时没有文件)。

在我们深入之前，先仔细看看initialize中的文件加载。除了漂亮的异常处理之外，它到底加载了什么？Ruby代码。很不起眼。如果你跳到save，你将看出来Array是怎样以代码形式存储到文件里的。这样initialize就能对其进行eval操作以重建它。

run_once方法是解法的核心部分。第一次调用时，它将@current设置为答案树Array，接着取决于该树有一个还是三个成员来展开分支。一个单元素Array就是种动物，用来猜测，用户被要求判断程序答案的正确性。如果对了，当前游戏终止并打印最后的信息。如果错了，用户会被要求给出正确的动物。注意这些答案是怎么存储的。"yes"和"no"分支将添加到@current，动物将被换成新的问题。这个动作将当前的单元素Array扩展到三个成员。现在加上去的分支都在Arrays里了，并保持树的继续成长。

当树的当前Array有三个成员时，我们知道这其实表示一个问题。程序将做出询问并根据回答选择下一个Array。

run方法让run_once可以重复执行。剩下的代码就是加载之前的文件并启动run方法。看这里对于ensure的漂亮运用，保证退出前能储存。这是十分必要的，因为exit可能在任何用户要做出输入的时候被调用。

####撇开树不谈####

现在，我必须得说使用树的方法很简单，但也不是没有缺点。我再一次引用Jim的话：

借助于树的办法有几个缺点。它对于添加动物的顺序和所用的问题类型十分敏感。树的办法在最初的问题能把可能的动物集合分成差不多的两部分的时候效果最好。这时能保证树有良好的平衡性，并且接下来问题到答案的路径都差不多短。不幸的是，在现实中，树倾向于变得非常的不平衡，不同问题会将某个特定的动物引入"yes"或者"no"分支中并弄出一个长长的问题列表。

对于树的另一个小问题是某些问题有歧义，或者在用户的知识范围内不能确定问题答案。比如某个问题可能是“它生活在水里么？”。一些人想的动物可能是海狸并觉得“当然是了，它非常喜欢游泳”。而有些人则会说“不，它生活在陆地上，只是喜欢游泳。”。在实际中，两种情况出现得差不多，你可能对某个问题同时得到"yes"和"no"的答案。每种分支会使用不同的问题来缩小选择范围。尽管不是个致命的缺陷，这也会在树中产生多余的答案，而且问题本身也被浪费了，本来可以好好利用的。

我也确实的思考了相当多的替代方案。不幸的是，每次我拆开树型结构后，添加新动物就变得很棘手。基本上我需要用户回答接近关于该动物的所有问题中的一半，而这样做的结果是我似乎要解决更多不相干的东西。总之，我最终放弃了这个方法。如果谁知道或者弄出了另一种思路，一定要记得分享！

####补充练习####

1.弄来尽可能多的人来试你的解法。比如把它带到你家小子的学校里去展示，做个web界面并让全世界都能使用。或者就把所有你家的客人拉到键盘前猜几把。在很多人用过后检查一下你的树。

2.加入一个历史功能，使得程序能够告诉你他一共猜对了多少次。

3.修改你的程序，使得它能告诉你它知道的某个动物，还有尽可能多的细节。比如：

<pre>
  ⇐  describe mouse 
  ⇒  A mouse is small.
     A mouse does not fly. 
     A mouse has fur.
  为了支持这个功能，你可能需要重新对动物结构的细节进行调整。
</pre>
