---
title: Ruby's Power Range
excerpt: Once in the evening I ran a couple of basic lines of code in the <b>irb</b> and received the results I couldn’t believe! But then I realized… 😏
category: Ruby
---

Once in the evening I ran a couple of basic lines of code in the <b>irb</b> and received the results I couldn’t believe! But then I realized… 😏

<h3> Boring Intro </h3>
<p>
Back in the days, when I wasn't familiar with Ruby but did climb the learning curve,
I'd already heard, that programs which were written in this language usually took more time to execute
comparing with any popular compiled language that had been taught in the university I studied.
Moreover, the pragmatism of the language was being killed by academical tasks, solutions of which contained loops not necessarily coming from <b> 0 </b> to the <b> end </b> of an array,
that made me end up with something like:

{% highlight ruby %}
  (1..x.length - 2).times { |i| ... }
{% endhighlight %}

Thus, the only reason to use it for such tasks was the desire of learning the language (not the best way, I agree).
</p>

<p>
That time <i> one company </i> successfully adopted Ruby and started actively evangelizing it. Somehow, I appeared on their lectures and witnessed something unexplainable.
One of the listeners joked about the performance and activated the <b><nobr> Search & Destroy </nobr></b> mode of the lector:
</p>

<blockquote>
<p>
  "What? Wait... stop... you're saying that Ruby is slow? That's bullshit! The only bottleneck is IO, other operations have the same performance as C"
</p>
</blockquote>

<p style="text-align: center;">
  <img src="/assets/images/posts/seriously.gif" style="width: 80%" />
</p>

<p>
Seems that I'll never understand the meaning of this phrase, no worries, but since I've been <b> triggered </b> by this already hazy recollection, why not just try it out and compare?
</p>

<h3> What about summing a range? </h3>
<p>
I opened <strong>irb</strong>, entered first lines of code I had in my head. What about calculating the sum of a range?

{% highlight ruby %}
  (0..10000000000).sum
  => 50000000005000000000 # immediately!
{% endhighlight %}

Damn!
</p>

<p style="text-align: center;">
  <img src="/assets/images/posts/epicbattle.gif" style="width: 80%" />
</p>

<p>
 I've received the result instantly!!1 <b> Shock content! </b> Excuse me not even placing a disclaimer with a warning!
</p>

<p>
But a few seconds later I added a couple more zeros to the number and realized that <i> practically constant </i> time needed to sum up a range.
</p>

<p>
The first thought I had: <i> ah, cool, seems like <a href="http://ruby-doc.org/core-2.5.0/Range.html" target="_blank">Range</a> class has this optimized method defined</i>.
However, turned out that it's just an <strong> if </strong> in the
<a href="https://github.com/ruby/ruby/blob/a60a1c031e7fc1302db3cc8e415383231fa5870b/enum.c#L3895" target="_blank"> Enumberable#sum </a>
<i> <nobr> yeah booooy</nobr></i>

<p>
It doesn't matter as long as it does precisely what we expect it to do:
</p>

1. Check if the object is a range
{% highlight c %}
if (RTEST(rb_range_values(obj, &beg, &end, &excl))) {
    if (!memo.block_given && !memo.float_value &&
            (FIXNUM_P(beg) || RB_TYPE_P(beg, T_BIGNUM)) &&
            (FIXNUM_P(end) || RB_TYPE_P(end, T_BIGNUM))) {
        return int_range_sum(beg, end, excl, memo.v);
    }
}
{% endhighlight %}

2. Use the formula that we all remember from school
{% highlight c %}
int_range_sum(VALUE beg, VALUE end, int excl, VALUE init)
{
    if (excl) {
        if (FIXNUM_P(end))
            end = LONG2FIX(FIX2LONG(end) - 1);
        else
            end = rb_big_minus(end, LONG2FIX(1));
    }

    if (rb_int_ge(end, beg)) {
        VALUE a;
        a = rb_int_plus(rb_int_minus(end, beg), LONG2FIX(1));
        a = rb_int_mul(a, rb_int_plus(end, beg));
        a = rb_int_idiv(a, LONG2FIX(2));
        return rb_int_plus(init, a);
    }

    return init;
}
{% endhighlight %}

3. Profit

</p>

<br/>

<blockquote>
<p>
  Previously I said that practically constant time is needed, but obviously, it takes more trivial operations to apply the formula of arithmetic progression to a bigger number.
  But it's nothing near as resource-intensive as iterating over a range with hundreds of billions of elements, as <b>reduce(:+)</b> would do.
</p>
</blockquote>

<h3> Dig a little deeper </h3>

Nevertheless, a bunch of methods that exists in <a href="https://ruby-doc.org/core-2.5.0/Enumerable.html" target="_blank">Enumerable</a> have its optimized definition in
<a href="http://ruby-doc.org/core-2.5.0/Range.html" target="_blank">Range</a>:

<ul>
  <li> <a href="http://ruby-doc.org/core-2.5.0/Range.html#method-i-first" target="_blank">first</a> </li>
  <li> <a href="http://ruby-doc.org/core-2.5.0/Range.html#method-i-include-3F" target="_blank">include?</a> </li>
  <li> <a href="http://ruby-doc.org/core-2.5.0/Range.html#method-i-max" target="_blank">max</a> </li>
  <li> <a href="http://ruby-doc.org/core-2.5.0/Range.html#method-i-member-3F" target="_blank">member?</a> </li>
  <li> <a href="http://ruby-doc.org/core-2.5.0/Range.html#method-i-min" target="_blank">min</a> </li>
</ul>

But the most tricky part is that some methods doesn't:

<h5> count </h5>

<p>
There is optimized <a href="http://ruby-doc.org/core-2.5.0/Range.html#method-i-size" target="_blank">Range#size</a> as an alternative.
</p>
<p>
<i> Range#count </i> is going to bubble up to <a href="https://ruby-doc.org/core-2.5.0/Enumerable.html#method-i-count" target="_blank">Enumerable#count</a> and instantiate an array from the range, the rest is history.
But <b><nobr> count → int </nobr> </b> and <b><nobr> count(item) → int </nobr></b> callings could be optimized and <b><nobr> count { |obj| block } → int </nobr></b> case could fallback to <a href="https://ruby-doc.org/core-2.5.0/Enumerable.html#method-i-count" target="_blank">Enumerable#count</a>.
<p>
For instance, <a href="http://ruby-doc.org/core-2.5.0/Range.html#method-i-last" target="_blank">Range#last</a>
has a similar implementation with fallback to <a href="http://ruby-doc.org/core-2.5.0/Array.html#method-i-last" target="_blank">Array#last</a>
(but could avoid it as <a href="http://ruby-doc.org/core-2.5.0/Range.html#method-i-first" target="_blank">Range#first</a> does).
</p>

<h5> minmax </h5>
<p>
Yea, there are <a href="http://ruby-doc.org/core-2.5.0/Range.html#method-i-min" target="_blank">Range#min</a> and
<a href="http://ruby-doc.org/core-2.5.0/Range.html#method-i-max" target="_blank">Range#max</a>, but <i> Range#minmax </i> is going to call the <a href="https://ruby-doc.org/core-2.5.0/Enumerable.html#method-i-minmax" target="_blank">Enumerable#minmax</a> implementation.
</p>

<hr />

<h3> Conclusion </h3>

<p>
Sure, there is a reason why a particular part of a library is implemented one way, but not another one.
Following the logic above, plenty of methods,
such as <a href="https://ruby-doc.org/core-2.5.0/Enumerable.html#method-i-drop" target="_blank">drop</a>, <a href="https://ruby-doc.org/core-2.5.0/Enumerable.html#method-i-sort" target="_blank">sort</a>, and <a href="https://ruby-doc.org/core-2.5.0/Enumerable.html#method-i-uniq" target="_blank">uniq</a>,
could have an optimized counterpart in the <a href="http://ruby-doc.org/core-2.5.0/Range.html" target="_blank">Range</a> class.
But most of them either useless for a range or can be replaced by a composition of other methods with a small overhead.
Therefore, such optimizations provide little or no benefit at the cost of a complication of the Standard Library.
</p>
<p>
But it's just useful to know the difference. For example, between <a href="http://ruby-doc.org/core-2.5.0/Range.html#method-i-size" target="_blank">Range#size</a>,
which executes instantaneously,
and <a href="https://ruby-doc.org/core-2.5.0/Enumerable.html#method-i-count" target="_blank">Range(Enumerable)#count</a>, which will take forever to finish on a ridiculously large range 😅.
</p>
