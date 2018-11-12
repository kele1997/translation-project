# MODERN C++ FOR C PROGRAMMERS: PART 4
# 为 C 程序员准备的现代 C++: 第四部分

[原文链接](https://ds9a.nl/articles/posts/cpp-4/)



Welcome back! In [part 3](https://ds9a.nl/articles/posts/cpp-3/) I discussed classes, polymorphism, references and templates, and finally built a source indexer out of basic containers that achieves 60MB/s indexing speed.  
欢迎回来！在 [第三部分](https://ds9a.nl/articles/posts/cpp-3/) 我讨论了类，多态，引用和模板，并且最后在基本容器的基础上实现了源代码索引工具，实现了每秒 60 M 的索引速度。  

In this part we continue with further C++ features that you can use to spice up your code ‘line by line’, without immediately having to use all 1400 pages of ‘The C++ Programming Language’. There is frequent reference to the indexing example from part 3 so you may want to make sure you know what that is about.   
在这个部分我们继续深入那些你可以一点点用在你代码上的 C++ 特性，而不必等到读完厚厚的 1400 多行的《The C++ Programming Language》一书后，再去使用。在这部分我们会经常引用到第三部分的索引器样例，因此你可能需要确保对于那个索引器有一定的了解。  

Various code samples discussed here can be found on [GitHub](https://github.com/ahuPowerDNS/hello-cpp).    
文中讨论的所有代码可以在 [Github](https://github.com/ahuPowerDNS/hello-cpp) 中找到。  


If you have any favorite things you’d like to see discussed or questions, please hit me up on @PowerDNS_Bert or bert.hubert@powerdns.com    
如果你有特别喜欢的讨论主题或者问题，请联系我 @PowerDNS_Bert 或者发邮件给我 bert.hubert@powerdns.com   

## Lambdas  
## Lambdas 表达式
We have previously encountered these weird looking snippets, for example like this:  
我们之前已经遇到过这些看起来很奇怪的代码，例如：   

```c++
std::sort(vec.begin(), vec.end(),
          [](const auto& a, const auto& b) { return a < b; }
     );
```

Although lambdas are, in essence, syntactical sugar, their availability has made modern C++ a far more expressive language. In addition, as noted in [part 1](https://ds9a.nl/articles/posts/c++-1/), passing code snippets as function pointers severely restricts what the compiler can do to optimize code. So not only do lambdas lead to fewer lines of code, the resulting binaries can be faster too.    
虽然 lambda 表达式本质上是语法糖，但他们的可用性使现代 C++ 成为一种更具表达力的语言。此外，正如 [第一部分](https://ds9a.nl/articles/posts/c++-1/) 所说，将代码片段作为函数指针传递会严重限制编译器优化代码的能力。因此 lambda 表达式不仅可以减少代码的行数，而且生成的二进制文件运行速度也更快。  



C++ lambdas are first class citizens and are compiled just like normal code. Scripting languages can easily do lambdas since they come with a runtime interpreter, C++ has no such luxury. So how does it all work?  
C++ 中的 lambdas 表达式是一等公民，编译的时候就像是普通代码一样。脚本语言很容易实现 lambda ，因为它们带有运行时解释器，C++ 没有这样的奢侈的东西。所以 lambda 表达式是怎样实现的呢？


Here is the anatomy: [capture specification](parameters) { actual code }. The capture specification can be empty which means the code in the lambda only ‘sees’ global variables, and this is a very good default. Captures can be by value or by reference. In general, if a lambda needs to capture a lot of detail, ponder if it is still a lambda.  
下面是剖析：` [指定捕捉外部参数](参数) { 实际代码 }` 。捕捉的外部变量可以是空的，这表示 lambda 表达式中的代码只能访问全局变量，这是一个非常好的默认值。捕捉可以使传入值或者传入引用。一般来说，如果一个 lambda 表达式需要捕捉大量的细节，那么需要思考这是否还是一个 lambda 表达式。  


For the parameters, you will very frequently use auto there, but it is in no way mandatory.  
对于这些参数来说，你会经常使用 `auto` 类型，但它不是强制性的。

The actual code is then between { and }, where the only special thing is that the return type is derived automatically, but you can also override it if you know what you are doing. A worked example:  
实际代码在 `{` 和 `}` 之间，其中唯一需要特殊说明的是返回类型会自动派生，但是如果你足够了解的话，你也可以覆盖它。一个实例：  

```c++
vector<string> v{"9", "12", "13", "14", "18", "42", "75"};

string prefix("hi ");

for_each(v.begin(), v.end(), [&prefix](const auto& a) {
	cout << prefix + a << endl;
}); // outputs hi 9, hi 12, hi 13 etc
```


In general, lambdas are awesome but best used for small, inline, constructs. If you find yourself capturing lots of stuff, you may actually be better off using a [functor](https://www.cprogramming.com/tutorial/functors-function-objects-in-c++.html), which is a class instance you can call (because it has overloaded operator()).
一般来说，lambdas很棒，但最好用在小型的、内联的、构造中。 如果你发现自己捕获了很多东西，你最好还是使用 [仿函数(functor)]（https://www.cprogramming.com/tutorial/functors-function-objects-in-c++.html），这是一个类,你可以调用它的实例（因为它重载了operator（））。

## Expanding our indexer
## 扩展我们的索引器  
In the indexer from [part 3](https://ds9a.nl/articles/posts/cpp-3) we ended up with:   
在第三部分的索引器中我们结束的位置在下面：  

```c++
struct Location
{
	unsigned int fileno;
	size_t offset;
};

std::unordered_map<string, vector<Location>> allWords;
```

This contains an unordered list of all words found in the indexed files, plus per word a vector of Locations where the word was found. We used an unordered map since this is 40% faster than an ordered one.  
这包含了一个无序的列表，其中包含了在已索引文件中找到的所有单词，还有每个单词被找到的位置向量。我们使用一个无序 map 容器，因为它比有序 map 要快 40% 。  

However, if we want to perform lookups for things like “main*“, to match everything that begins with “main”, we also need an ordered list of words:
然而，如果我们想要实现查找类似 “main*” 这样以 "main" 开头的所有词，我们需要一个有序的单词列表：  

```c++
  std::vector<string> owords;

  owords.reserve(allWords.size()); // saves mallocs

  for(const auto& w : allWords)
    owords.push_back(w.first);

  sort(owords.begin(), owords.end());
```

Note how this uses the range-for construct to insert only the keys of the allWords map into a vector, as yet unsorted, which we remedy in the final line.  
注意这里是怎样使用 range-for 结构来将 `allWords` 中的键值插入到一个 `vector` 的，但是目前尚未进行排序，我们在最后一行中完成排序。  

Interestingly enough, we do not lose out on our 40% speedup since ‘sort once we are done’ is faster than keeping everything sorted all the time.  
非常有趣的是，我们并没有失去 40% 的速度提升，因为 “索引完成后进行排序” 比一直保持所有单词有序快的多。   

Should we be in the mood, we could attempt to be smarter. As written above, every word is now present in memory twice, once in allWords, once in owords.  
如果我们心情愉快，我们可能再聪明一些。正如上面所写，每一个单词现在都存在内存里面两份，一份在 `allWords`,另一个在 `owords`。

It is a pretty C-like thing to not do this and live on the edge for a bit:  
这一个很棒的 C 类似东西来防止这种情况，并且有点接近底层：  

```c++
  std::vector<const string*> optrwords;
  optrwords.reserve(allWords.size());

  for(const auto& w : allWords)
    optrwords.push_back(&w.first);

  sort(optrwords.begin(), optrwords.end(),
       [](auto a, auto b) { return *a < *b;}
       );
```


With this code, we store const pointers to the keys in the allWords unsorted map. Then we sort optrwords, containing pointers, using a lambda that dereferences these pointers.
使用这样的代码，我们保存一些 const 的指针，这些指针指向无序 map `allWords` 的键值。然后我们对 `optrwords` 进行排序，`optrwords` 中包含了全是指针，因此 lambda 表达式中我们进行了解引用。  

If we index the Linux source tree, which contains around 600,000 unique words, this does save us around 14 megabytes of memory, which is nice.  
如果我们建立了 Linux 源代码索引树，这棵树会包含大约 600,000 个互不相同的单词，这大概需要占用 14 兆字节的内存，这是很棒的。

The downside however is that we are now storing raw pointers straight into another container (allWords). As long we don’t modify allWords this is safe. And for some containers, it is even safe if we do make changes. This happens to be the case for [std::unordered_map](https://en.cppreference.com/w/cpp/container/unordered_map), as long as we don’t actually delete an entry we store a pointer to.    
接下来我们要把那些指针存储到另外一个容器（`allWords`）里面。只要我们不修改 `allWords` 这就是安全的。对于另外一些容器，如果我们进行修改，这甚至仍是安全的。以 [std::unordered_map](https://en.cppreference.com/w/cpp/container/unordered_map) 为例，只要我们没有真的删除那些指针指向的元素，这就是安全的。  


I think this illustrates a key point of using modern C++. You can shave 14 megabytes of memory ‘if you know what you are doing’, but I highly recommend that you only reach into this ‘C’ like bag of tricks if you really need to. But if that is the case, it is good to know it can be done.  
我认为这体现了现代 C++ 很重要的一点。如果你知道你在做什么的话，你可以节省到 14 Mb 的内存，但我强烈建议你只有在真正需要的时候才使用这个 ‘C’ 语言的技巧。但如果真是这种情况的话，最好知道是否能够使用这个技巧。


## Containers and algorithms
## 容器和算法

We have seen a variety of [containers](https://en.cppreference.com/w/cpp/container) so far (std::vector, std::unordered_map for example). In addition, there is a raft of algorithms that can operate on these containers. Crucially, through the use of templates, algorithms are actually completely decoupled from the containers they operate on.
我们到现在为止已经见过许多 [容器(containers)](https://en.cppreference.com/w/cpp/container) （例如 `std::vector`, `std::unordered_map`）。另外，有许多算法可以运行在这些容器上。关键是，通过使用模板，算法可以完全和它们操作的容器分离开来。  


This decoupling has enabled the standard to specify a larger than usual amount of generically useful algorithms. We’ve already encountered std::for_each and std::sort, but here’s a more exotic one: std::nth_element.  
这种分离使得标准库在实现了大量基础算法的基础上了添加了更多的算法。我们已经遇到过 `std::for_each` 和 `std::sort`，此外还有一个比较奇特的：`std::nth_element`。  

Going back to our indexer, we have a list of words and how often they occur. Let’s say we want to print the 20 most frequently occurring ones, we’d normally take the whole list of words, sort them in order of frequency and then print the top 20.    
回到我们的索引器，我们现在有一个单词列表和它们出现的频率。假设我们想要输出 20 个最常出现的单词，我们通常需要使用整个单词列表，按照出现的频率进行排序，然后输出前 20 个单词。  

With std::nth_element, we can actually get what we want. First, let’s gather the data to sort, and define the comparison function:  
使用 `std::nth_element` ,我们可以获得我们需要的数据。首先，让我们收集一些数据用来排序，并且定义一个比较函数：    

```c++
 vector<pair<string, size_t>> popcount;
  for(const auto& w : allWords)
    popcount.push_back({w.first, w.second.size()});

  auto cmp = [](const auto& a, const auto& b)
       {
         return b.second < a.second;
       };
```

We are defining a vector containing pairs. A pair is a convenient templated struct containing two members, called first and second. I find that pair inhabits a sweet spot that is quite useful, an ‘anonymous struct’ with well known names. Confusion occurs when pairs get nested into pairs, or when using std::tuple, which is std::pair on steroids. Beyond two simple members, create a struct with named members.   
我们定义了一个容纳 `pairs` 的容器。`pair` 是一个方便的模板结构可以容纳两个成员数据，分别称作 `first` 和 `second`。我发现 `pair` 是非常有用的众所周知的 “匿名结构体”。当嵌套使用 `pairs` 或者使用 `std::tuple` （加强版的 `std::pair` ）时，通常会发生混乱。

The range-for loop shows one new feature, ‘brace initialization’, which means w.first and w.second.size() (which is the number of occurrences of this word) are used to construct our pair. It saves a lot of typing.   
range-for 循环展示了一个新的特性，‘大括号初始化’(‘brace initialization’),这意味着 `w.first` 和 `w.second.size()` （这个单词出现的次数）被用来构造我们的 `pair`。这减少了许多代码。   

Finally, we define a comparison function and call it cmp so we can reuse it. Note that it compares in reverse order.  
最后，我们定义一个比较函数，并将其命名为 `cmp`，以便我们能够重复使用它。请注意它按照递减顺序进行比较。    

Next up, the actual sorting and printing:    
接下来，实际的排序和输出代码：  


```c++
  int top = std::min(popcount.size(), (size_t)20);
  nth_element(popcount.begin(), popcount.begin() + top, popcount.end(), cmp);
  sort(popcount.begin(), popcount.begin() + top, cmp);

  int count=0;
  for(const auto& e : popcount) {
    if(++count > top)
      break;
    cout << count << "\t" << e.first << "\t" << e.second << endl;
  }

```

This invocation of std::nth_element bears some explanation. As noted, iterators are places within a container. begin() is the first entry and, consistently, end() is one beyond the last entry. On an empty container, begin() == end().  
对于 `std::nth_element` 的调用需要一些解释。如上所述，`iterators`（迭代器）代表的是容器中的位置。`begin()` 是第一个条数据，这始终是不变的，`end()` 是最后一条数据后面的数据。对于空的容器，`begin() == end()`。   

We pass three iterators to nth_element: where to begin the sorting, what the cutoff is for our ‘top 20’ and finally the end of the container. What nth_element then does is make sure that the entire top-20 is in fact in the first 20 positions of the container. It does not however guarantee that the top-20 itself is sorted. For this reason we do a quick sort of the first 20 entries.    
我们将三个迭代器传入 `nth_element`: 排序开始的位置，我们放置前20个元素的终点位置，最后是容器的末尾。那么 `nth_element` 就可以确保前20个元素就位于容器的前20个位置上。但是它并不能保证前20个元素是排好序的。因此我们并且为前 20 个数据调用快速排序。   

The final 6 lines print the actual top-20, in the correct order.  
最后 6 行以正确的顺序输出前 20 个数据：  

这个std :: nth_element的调用有一些解释。 如上所述，迭代器是容器内的位置。 begin（）是第一个条目，始终如一，end（）是超出最后一个条目的条目。 在空容器上，begin（）== end（）。

我们将三个迭代器传递给nth_element：从哪里开始排序，我们的'前20'的截止点是什么，最后是容器的末尾。 那么nth_element确保整个top-20实际上位于容器的前20个位置。 但是，它并不能保证排名前20的排序。 出于这个原因，我们快速排序前20个条目。

最后的6行以正确的顺序打印实际的前20行。

C++ comes with [many useful algorithms](https://en.cppreference.com/w/cpp/algorithm) that allow you to compose powerful programs. For example, [std::set_difference](https://en.cppreference.com/w/cpp/algorithm/set_difference), [std::set_intersection](https://en.cppreference.com/w/cpp/algorithm/set_intersection) and [std::set_symmetric_difference](https://en.cppreference.com/w/cpp/algorithm/set_symmetric_difference) make it trivial to write ‘diff’ like tools or find out what changed from one state to another.  
C ++附带了 [许多有用的算法]（https://en.cppreference.com/w/cpp/algorithm），可以让你编写更强大的程序。 例如，[std :: set_difference]（https://en.cppreference.com/w/cpp/algorithm/set_difference），[std :: set_intersection](https://en.cppreference.com/w/cpp/algorithm/set_intersection)和 [std :: set_symmetric_difference](https://en.cppreference.com/w/cpp/algorithm/set_symmetric_difference) 使得编写 ‘diff’ 之类的工具或者找出从一个状态到另一个状态的变化变得非常简单。  



Meanwhile, algorithms like [std::inplace_merge](https://en.cppreference.com/w/cpp/algorithm/inplace_merge) and [std::next_permutation](https://en.cppreference.com/w/cpp/algorithm/next_permutation) may prevent you from having to whip out the Knuth books.  
同时，像是 [std :: inplace_merge](https://en.cppreference.com/w/cpp/algorithm/inplace_merge) 和 [std :: next_permutation](（)https://en.cppreference.com/w/cpp/algorithm/next_permutation) 这样的算法可以防止你不得不掏出 Knuth （唐纳德）的书。


Before doing any kind of data manipulation or analysis, I urge you to [go through the list](https://en.cppreference.com/w/cpp/algorithm) of existing algorithms, you will likely find things there that get you most of the way.  
在操作和分析任何类型的数据之前，我强烈建议您 [浏览这个列表](https://en.cppreference.com/w/cpp/algorithm) 了解已有算法，你可能会找到对你有帮助的东西。    

## Looking things up
## 查找
As an example, recall that we made a sorted list of words so we could do prefix lookups. All words ended up in std::vector<string> owords. We can interrogate this flat (and hence very efficient) container in several ways:  
举个例子，回想一下，我们生成了一个排好序的单词列表，以便我们可以进行前缀查找。 所有单词最后都存放在 `std :: vector <string> owords` 中。 我们可以通过以下几种方式询问这个线性（因此非常有效）的容器：

* std::binary_search(begin, end, value) will let you know if a value is in there.
* std::equal_range(begin, end, value) returns a pair of iterators which span all exactly matching entries.
* std::lower_bound(begin, end, value) returns an iterator that points to the first place value could be inserted without changing the sorting order. upper_bound returns the last iterator where that is true.

* `std::binary_search(begin, end, value)` 可以使你知道容器中是否有这个值
* `std::equal_range(begin, end, value)` 返回一对迭代器，分别位于满足条件的数据序列的开头和结尾  
* `std::lower_bound(begin, end, value)` 返回一个迭代器，指向不会改变当前排序且可以插入数据的第一个位置。`upper_bound` 返回最后一个迭代器，其余的和 `lower_bound` 一样。

As long as we don’t have multiple equivalent entries in our container, lower_bound and upper_bound are the same. To list all words starting with main from our sorted vector owords, we can do:  
只要我们的容器中没有多个相等的数据，`lower_bound` 和 `upper_bound` 结果是一样的。 要从我们排好序的 vector 容器中列出以 `main` 开头的所有单词，我们可以这样做：  

```c++
string val("main");

auto iter = lower_bound(owords.begin(), owords.end(), val);

for(; iter != owords.end() && !iter->compare(0, val.size(), val); ++iter)
	cout<<" "<<*iter<<endl;
```

std::lower_bound does the heavy lifting here, performing a binary search over our sorted std::vector. The for loop bears a bit of explaining. The first check iter != owords.end() will stop us if lower_bound did not find anything.      
这里我们使用 `std::lower_bound` 来完成这项繁重的工作，对我们已排好序的 `std::vector` 执行二分搜索。 `for` 循环需要一点解释。 如果 `lower_bound` 没有找到任何内容，第一个检查 `iter！= owords.end()` 将终止循环。

The second check using iter->compare performs a substring match on at most the first 4 characters of a candidate word. Once that no longer matches, we’ve iterated beyond the words that start with “main”.  
第二个检查使用 `iter->compare`来对候选词的前 4 个字符串进行子串匹配。一旦不再匹配，我们就会迭代其他以 `main` 开头的单词。   

## Some more containers
## 更多的容器

In the previous examples we’ve used the very basic std::vector, which is contiguous in memory and compatible with C, and std::unordered_map, which is a pretty fast key/value store, but has no ordering.     
在前面的例子中我们已经使用了最基础的 `std::vector` ,它在内存中是连续的，并且可以和 C 语言搭配使用，还使用了 `std::unordered_map` ,它是一个非常高效的键值对存储结构，但是它是无序的。

There are several other useful containers:  
这里还有其他几种有用的容器：   

* std::map an ordered map, where you can pass a comparison function if you want, for example to get case-insensitive ordering. Many many examples you will see needlessly use std::map. This is because pre-2011, C++ had no unordered containers. Ordered containers are wonderful when you need ordering, but present unnecessary overhead otherwise.
* std::set. This is like a std::map<string,void>, in other words, it is a key value store without values. Like std::map it is ordered, which you often do not need. Luckily there is also std::unordered_set.
* std::multimap and std::multiset. These work just like regular set and map, but then allowing multiple equivalent keys. This means these containers can’t be queried with [] since that only supports a single key.
* std::deque. A double-ended queue which is a great workhorse for implementing any kind of queue. Storage is not consecutive, but popping and pushing elements from either end is fast.   

* `std::map` 有序 map 容器，如果你需要的话，你可以传入一个比较函数，比如你需要得到大小写敏感的顺序。你遇到的许多许多例子都不必使用 `std::map`。这是因为 2011 年之前,C++ 还没有未排序容器。当你需要排序的时候，排序容器是非常完美的，但是目前来说是不必要的，因为会产生不必要的开销。   
* `std::set`。这个容器有点像 `std::map<string,void>` ,换句话说，这可以看做没有值的键对值存储。像 `std::map` 这样排序的容器，通常来说你不需要。幸运的是还有一个 `std::unordered_set` 。
* `std::multimap` 和 `std::multiset` 。它们和普通的 `set` 和 `map` ，但是允许多个相同的键值。这意味着这些容器不能使用 `[]` 运算符进行查询，因为这个运算符只支持一个键。
* `std::deque`。双端队列，这是实现任何一种队列的完美工具。存储位置不是连续的，但是从队列的任意一端出栈和入栈都是很快的。  

A full list of standard containers can be found [here](https://en.cppreference.com/w/cpp/container)  
标准容器的完整列表可以在[这里](https://en.cppreference.com/w/cpp/container) 找到   



## Boost containers
## Boost 容器

Although this series of posts focuses on ‘core’ C++, I would be remiss if I did not point out a few parts of Boost at this point. Boost is a large collection of C++ code, some of which is excellent (and tends to make it into the C++ standard, which is edited by some of the Boost authors), some of which is good and then there are some unfortunate parts.  
尽管这个系列文章侧重于 ‘核心’ C++,但如果我不讨论一点 Boost 的内容，那么我们会错过一部分内容。Boost 一个很大的 C++ 代码集合，其中一些非常棒（并且倾向于成为 C++ 标准库，这一部分由一些 Boost 的开发者编写的），其中一些也不错，但是还有一部分不太好的部分。   

But the good news is, most of Boost is pretty modular: it is not a framework kind of library where if you use one part, you use all parts. And in fact, many of the most interesting parts are include-only, with no need to link in libraries. Boost is universally available and freely licensed.  
但好消息是，Boost 的大部分代码都是模块化的：它并不像是类似于库的框架，如果你想使用一部分，你就必须使用全部。事实上，许多最有趣的部分仅需要 include 的(include-only)，不必再链接库。Boost 是通用的，并且许可免费。   

First up is the Boost Container library, which is not a library but a set of includes. This offers niche containers that are mostly completely compatible with standard library containers, but offer specific advantages if they match your usecase.  
首先是 Boost 容器库，它其实不是一个库，而是许多 include 的集合。它提供了一些容器，这些容器大多可以完全和标准库容器兼容，但是如果他们满足你的需要，它们还会提供许多特定的优势。    

For example, boost::container::flat_map (and set) are like std::map and std::set except they use contiguous slabs of memory for cache efficiency. This makes them slow on inserts, but lightning fast on lookups.  
例如， `boost::container::flat_map`(和 `set`) 就像是 `std::map` 和 `std::set`，不过 boost库中使用了连续的内存块来提高缓存效率。这使得它们在插入时变慢，但是在查找时却非常快。   

As another example, boost::container::small_vector is optimized for storing a small (templatized) number of elements, which can save a lot of malloc traffic.  
另一个例子， `boost::container::small_vector` 在存储少量（模板化）的元素是，可以节省大量 `malloc` 造成的开销。  

Lots more Boost containers can be found [here](https://www.boost.org/doc/libs/?view=category_containers).  
更多 Boost 容器可以在[这里](https://www.boost.org/doc/libs/?view=category_containers)找到。

### Boost.MultiIndex
### Boost.MultiIndex

Secondly, in part 1 of this series I promised I’d stay away from exotics and “template metaprogramming”. But there is one pearl I must share with you, something I regard as the golden standard by which I measure programming languages. Is the language powerful enough to implement Boost.MultiIndex?
其次，在这个系列的第一部分我许诺说我会远离那些奇异的东西和“模板元编程”。但有一点东西我必须和你分享，我认为这是衡量编程语言的黄金准则。该语言是否足够强大到可以实现 Boost.MultiIndex?    

In short, we frequently need objects to be findable in several ways. For example, if we have a container of open TCP sessions, we may want to find sessions based on the ‘full source IP, source port, destination IP, destination port’ tuple, but also on only source IP or destination IP. We might also like to have time ordering to harvest/close old sessions.  
简而言之，我们经常需要以多种方式查找对象。例如，如果我们现在有一个容纳了许多打开的 TCP 会话的容器，我们可能需要基于 ‘完整的源 IP,源端口，目的 IP,目的端口’ 元组来查找会话，但也有可能只依靠源IP或者目的IP来进行查找。我们可能还希望按照时间顺序来 获得/关闭 旧的会话。   

The ‘manual’ way of doing this is to have several containers in which objects live, and use all these to find objects using various keys:  
实现上述功能的 ‘手动’ 方式是用几个容器保存数据的位置，并使用各种键在容器中查找数据对象：   

```c++
map<pair<IPEndpoint,IPEndpoint>, TCPSession*> d_sessions;
map<IPEndpoint, TCPSession*> d_sessionsSourceIP;
map<IPEndpoint, TCPSession*> d_sessionsDestinationIP;
multimap<time_t, TCPSession*> d_timeIP;

auto tcps = new TCPSession;
d_sessions[{srcEndpoint, dstEndpoint}] = tcps;
d_sessionsSourceIP[srcEndpoint] = tcps;
d_sessionsDestinationIP[dstEndpoint] = tcps;
...
```

While this works, we suddenly have to do a lot of housekeeping. If we want to remove a TCPSession, we have to remember to erase it from all containers, for example, and then free the pointer.  
虽然这样实现了多重索引，但我们现在不得不处理大量的琐事。例如：如果你想要删除一个 TCPSession,我们必须记得从所有容器中删除它，然后释放这个指针。

Boost.MultiIndex is a work of art that not only offers containers that can be searched in several ways at the same time, it also offers (un)ordered, unique and non-unique indexes, plus the use of partial keys for lookups, as well as ‘alternate keys’, which enable you to find a std::string key using a char * (which saves the creation of temporaries).   
Boost.MultiIndex 就像是一件艺术品 ，它不仅提供了可以同时以多种方式搜索的容器，还提供了（无）有序、去重(unique)和不去重(non-unique)的索引，还提供了使用部分键（也叫作‘可选键’）进行查找的功能，它使你能够通过 `char* `（其中保存了临时变量）来查找 `std::string` 键。  

Here’s how we’d lookup TCP sessions. First let’s start with some groundwork ([full code](https://github.com/ahupowerdns/hello-cpp/blob/master/multi.cc)):   
下面是我们查找 TCP 会话的代码。首先让我们从一些简单的任务做起（[完整代码](https://github.com/ahupowerdns/hello-cpp/blob/master/multi.cc)）：  


```c++
struct AddressTupleTag{};
struct DestTag{};
struct TimeTag{};

struct Entry
{
  IPAddress srcIP;
  uint16_t srcPort;
  IPAddress dstIP;
  uint16_t dstPort;
  double time;
};
```

The three Tags provide types that identify three different indexes we will be defining on our container. We then define a struct that our Boost.MultiIndex container will contain. Note that the keys we want to search on are in the actual container itself - there is no separation here between key and value.    
这三个 `Tags` 提供了标识类型，用来区分我们在容器中定义的三种不同的索引。然后我们定义一个结构体，我们会向我们的 Boost.MultiIndex 容器中放置这种结构体。请注意，我们搜索用的键是位于实际的容器本身的 - 键和值之间没有分离。   

Next up the admittedly tricky definition of the container. You might literally spend an hour getting this right, but once it is right everything is easy:  
接下来是容器中公认比较棘手的定义。你可能花了一个小时才作对这一点，但一旦完成，剩下的事情就很简单了：  

```c++
typedef multi_index_container<
  Entry,
  indexed_by<
    ordered_unique<
      tag<AddressTupleTag>,
      composite_key<Entry,
                    member<Entry, IPAddress, &Entry::srcIP>,
                    member<Entry, uint16_t, &Entry::srcPort>,
                    member<Entry, IPAddress, &Entry::dstIP>,
                    member<Entry, uint16_t, &Entry::dstPort> >
      >,
    ordered_non_unique<
      tag<DestTag>,
      composite_key<Entry,
                    member<Entry, IPAddress, &Entry::dstIP>,
                    member<Entry, uint16_t, &Entry::dstPort> >
      >,
    ordered_non_unique<
      tag<TimeTag>,
      member<Entry, double, &Entry::time>
      >
    >
  > tcpsessions_t;  

```


This defines three indexes, one ordered & unique, and two ordered and non-unique. The first index is on the ‘4-tuple’ of the TCP session. The second one only on the destination of the session. The last one on the timestamp.  
这里定义了三个索引，一个是有序的并且去重的，还有两个是有序但未去重的。第一个索引位于 TCP 会话的 4元组。第二个索引位于会话的目的地。最后一个位于时间戳。   


It is important to note that this template definition creates the entire container code at compile time. All of this leads to code that, as usual for templated containers, is as efficient as if you had written it yourself. Most of the time in fact, a Boost.MultiIndex container will be faster than a std::map.
一定要注意这个模板在编译时创建了整个容器的代码。这样做的结果是：这样的代码和你亲自编写的模板化容器效率一样。大多数情况下， `Boost.MultiIndex` 容器将比 `std::map` 快的多。   

Let’s fill it with some data:  
让我们用一些数据填充整个容器：  

```c++
tcpsessions_t sessions;
  double now = time(0);

  Entry e{"1.2.3.4"_ipv4, 80, "4.3.2.1"_ipv4, 123, now};

  sessions.insert(e);
  sessions.insert({"1.2.3.4"_ipv4, 81, "4.3.2.5"_ipv4, 1323, now+1.0});
  sessions.insert({"1.2.3.5"_ipv4, 80, "4.3.2.2"_ipv4, 4215, now+2.0});

```

The first line uses our typedef to make an actual instance of our container, the second line takes the current time and puts it in a double.  
第一行使用我们 `typedef` 创建的类型来创建一个容器实例，第二行代码将当前时间作为参数，存入一个 `double` 类型变量中。   

Then some magic called a [user-defined literal](https://en.cppreference.com/w/cpp/language/user_literal) happens, which means that "1.2.3.4"_ipv4 gets converted to 0x01020304 - at compile time. To observe how this works, head over to [multi.cc](https://github.com/ahupowerdns/hello-cpp/blob/master/multi.cc) on GitHub. These party tricks are optional when using C++, but constexpr compile time code execution is pretty neat.  
然后一些被称为 user-define literal 的技巧出现了，这意味着 `"1.2.3.4"_ipv4` 在编译时会被转换成 0x01020304。为了观察它是怎样工作的，我们阅读一下 Github 上 [multi.cc](https://github.com/ahupowerdns/hello-cpp/blob/master/multi.cc) 的代码。使用 C++ 是，这些技巧是可选的，但常量表达式(constexpr)编译器出的执行代码会非常简洁。  


After this has run, our sessions container has 3 entries. Let’s list them all in time order:    
完成上面任务之后，我们的会话容器现在有三个数据条目。让我们以时间顺序把它们全部列出来：   

```c++
  auto& idx = sessions.get<TimeTag>();
  for(auto iter = idx.begin(); iter != idx.end(); ++iter)
    cout << iter->srcIP << ":" << iter->srcPort<< " -> "<< iter->dstIP <<":"<<iter->dstPort<<endl;
```

This prints:    
输出结果是：    

```
1.2.3.4:80 -> 4.3.2.1:123
1.2.3.4:81 -> 4.3.2.5:1323
1.2.3.5:80 -> 4.3.2.2:4215
```


In the first line, we request a reference to the TimeTag index, which we iterate over as usual in the second line.    
在第一行中，我们访问 `TimeTag` 索引的引用，然后我们像往常一样遍历它。    

Let’s do a partial lookup in the ‘main’ (first) index, which is on the full 4-tuple:  
让我们在 ‘主’索引（4 元组中第一个索引）进行部分单词的查找：   

```c++
  auto range = sessions.equal_range(std::make_tuple("1.2.3.4"_ipv4));

  for(auto iter = range.first; iter != range.second ; ++iter)
    // print
```

By creating a tuple with only one member, we indicate we want to do our lookup on only the first part of the 4-tuple. If we had added ‘, 80’ to std::make_tuple, we would have found only one matching TCP session instead of two. Note that this lookup used equal_range as described earlier in this page.  
通过创建一个只有一个数据成员的元组，这表示我们只想在 4 元组的第一部分进行查找。如果我们想把 ‘, 80’ 传递到 `std::make_tuple` 中，我们会发现只有一个匹配的 TCP 会话而不是两个。请注意，这次查找使用 了本文之前所说的 `equal_range` 函数。    

Finally to search on a TCP session’s destination:  
最后按照 TCP 会话的目的地进行搜索：   

```c++
 cout<<"Destination search for 4.3.2.1 port 123: "<<endl;
  auto range2 = sessions.get<DestTag>().
	equal_range(std::make_tuple("4.3.2.1"_ipv4, 123));

  for(auto iter = range2.first; iter != range2.second ; ++iter)
    // print
```

This requests the DestTag index, and then uses it to find sessions to 4.3.2.1:123.  
这次访问 `DestTag` 索引，然后使用它来查找目的地是 `4.3.2.1:123` 的会话。   

I hope you will forgive me this excursion outside of standard C++, but as Boost.MultiIndex is part of almost all code I write, I felt the need to share it.    
我希望你能够原谅我稍微偏离了标准 C++ 之外的内容，但是由于 `Boost.MultiIndex` 的代码几乎全是我编写的，所以我认为有必要分享它。

## Summary
## 总结  

In this long part 4, we have delved into some of the nitty-gritty of lambdas, and how they can be used for custom sorting, how they can be stored and when they are a good idea.
在第四部分里，我们深入研究了 lambdas 语法的细节，以及它们怎样用于自定义排序的，它们怎样存储的，以及什么时候用 lambda 表示式。

Secondly we explored the interaction between algorithms and containers by enhancing our code indexer with the ability to look up partial words, by sorting our unordered container of words into a flat vector. We also looked how some ‘C’ like tricks could be used to make this process both use less memory and be more dangerous.  
其次，我们改进代码索引器，把无序容器中的单词排序成为一个线性的 vector 容器，然后添加一个查找部分单词的功能，以便我们探索算法和容器之间的交互。我们还会研究怎样使用一些 ‘C’ 语言的技巧来使这个程序使用更少的内存，也更不安全。

We also looked into the rich array of algorithms provided by C++, enabled by the separation of code between containers and algorithms. Before doing any data manipulation, check the existing algorithms if there is something that does what you need already.  
我们还研究了 C++ 提供的大量的算法，容器和算法的代码是分离的。在进行任何数据操作之前，检查一下是否已经存在你需要的算法了。  

Finally we covered further containers found in Boost, including the most magical and powerful Boost.MultiIndex.  
最后我们介绍了 Boost 中创建的容器，如最神奇和最强大的 `Boost.MultiIndex`。  

In part 5 we will round off this series with a discussion of the ultimate smart pointer, std::unique_ptr and the associated concept of std::move. If you have any other favorite things you’d like to see discussed or questions, please do hit me up on @PowerDNS_Bert or bert.hubert@powerdns.com.   
在第五部分中我们会讨论终极的智能指针 `std::unique_ptr` 和 `std::move`的相关概念。如果你有任何你想讨论的喜欢的事情或者问题的话，请联系我 [@PowerDNS_Bert](https://twitter.com/PowerDNS_Bert) 或者使用邮箱 bert.hubert@powerdns.com 。
