Circuit Breaker
--------------

[Circuit Breaker](https://en.wikipedia.org/wiki/Circuit_breaker)，翻译一下就是，电路破坏器。翻译过来好像也没有什么卵用。这个东西的原始意思在wiki上比较清楚，主要就是对电路的一个过载保护。举个例子，当家里（寝室里）的电器开多了（比如你在寝室里打开了一个高功率的电饭煲来煮面），然后你就发现跳闸了。跳闸其实就是对电路的一个保护，原因是过载或者发生短路了。
Circuit Breaker在程序开发里的意思，可以参考Martin Fowler的[CircuitBreaker](http://martinfowler.com/bliki/CircuitBreaker.html)。简单来说，首先得明白这个东西的上下文：在对外部资源进行调用的时候，这里的外部资源可以是一个http api也可以是另外一个进程。如果外部资源不稳定，比如一会儿有返回，一会儿没返回。这时候有个Circuit
Breader就会来统计没有返回的次数，如果次数达到一定值，那么后面我们的程序再去调用这个外部资源的时候，就不去真正发请求了，就直接返回错误。
那么，写代码的时候怎么实现呢？马大叔说大概理念就是用一个Circuit Breaker的代码包裹一下你的原来的发请求的代码，然后统计一下请求返回失败的次数，如果次数达到一定数量，就直接返回错误了。
