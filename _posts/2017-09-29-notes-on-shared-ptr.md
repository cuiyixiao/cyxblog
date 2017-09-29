---
layout: post
title: Notes on shared_ptr
date: 2017-09-29 18:49:00 +0800
categories: c++
---

It may be unnessary to introduce what `shared_ptr` is. Let's just figure out how its use_count increase and decrease, and some potential misuse we need to avoid.

### Share Ownership
It is described in detail on [c++ reference shared_ptr](http://www.cplusplus.com/reference/memory/shared_ptr/). And I think the following paragraph is quite important:
> **shared_ptr** objects can only share ownership by copying their value:
>  If two **shared_ptr** are constructed (or made) from the same (non-**shared_ptr**) pointer, they will both be owning the pointer without sharing it, causing potential access problems when one of them releases it (deleting its managed object) and leaving the other pointing to an invalid location.

This implies that we should only share ownership of objects by using copy constructor and assignment operator of shared_ptr, and after we initialize a shared_ptr from a non-shared_ptr, we should not do that again until the group of owners are all released.

Let's run a snippet of code for better understanding:
{% highlight c++ %}
#include <iostream>
#include <memory>

using namespace std;

int main()
{
    int *p = new int;
    shared_ptr<int> sp1(p);
    shared_ptr<int> sp2(p);
    cout << "sp1 use_count: " << sp1.use_count() << endl;
    cout << "sp2 use_count: " << sp2.use_count() << endl;

    shared_ptr<int> sp3(sp1);
    shared_ptr<int> sp4 = sp3;
    shared_ptr<int> sp5 = move(sp4);
    cout << "sp3 use_count: " << sp3.use_count() << endl;
    cout << "sp4 use_count: " << sp4.use_count() << endl;
    cout << "sp5 use_count: " << sp5.use_count() << endl;
}
{% endhighlight %}

And the output is as below:
> sp1 use_count: 1  
> sp2 use_count: 1  
> sp3 use_count: 3  
> sp4 use_count: 0  
> sp5 use_count: 3  
> *** Error in `./bin/smart_ptr': double free or corruption (fasttop): 0x095fd008 ***  

`sp1` and `sp2` both own `p` but their use_count() is 1 as described above. And further they cause segmentation fault because the releases of them will make `p` deleted twice.

`sp3` owns `p`, thus use_count add to 2. `sp4` again owns `p`, use_count is 3 now. `sp5` is constructed with the `move` semantics with `sp4`, so `sp4` is released, and the use_count remains unchanged. So we got the above result.

