---
layout: post
title: Error handling in C
---

C, a language created in 1972, does not provide an easy way to handle errors like modern programming languages. We cannot "try" some code and "catch" an error. So how do we handle errors in C?


Almost all of the top 10 results in Google suggest the same approach:

~~~~C
int err = someFunction();
if(err != ERROR_SUCCESS){
	//fail and do something
}
else{
	//do this
}
~~~~

In large code bases this is no way elegant or practical (think of all the nested ifs!). So is there a better way to do this? (hint: macros)

Our team in VMware is responsible for writing platform code, and the code base is huge. This is how error handling is implemented in our code base.

Instead of repeated if, else blocks navigating the flow of execution around the errors, the following macro is used:

~~~~C
int err = someFunction();
BAIL_ON_ERROR(err);
~~~~

The macro is defined:

~~~~C
#define BAIL_ON_ERROR(x) if(x){LogError(x); goto error;}
~~~~

LogError is a function call to log this error into the log file - make this function threadsafe. And the next part - oh the blasphemy - we use goto.

Gotos get a bad rep because of the famous Dijkstra's [letter](http://www.cs.utexas.edu/users/EWD/ewd02xx/EWD215.PDF), and we are taught in college that gotos are bad and harmful. But with well thought out code, gotos can be very useful in writing structured and readable code. One obvious rule that must be followed is "no backward gotos".

So this is how the code looks including the error section:

~~~~C
int err = someFunction();
BAIL_ON_ERROR(err);
int err = anotherFunction();
BAIL_ON_ERROR(err);
...
...
...
error:
	//clean up all the allocated resources
	return err;
~~~~


Moving the deallocation part to the error section gives two advantages:

* As gotos are fall through, there need not be a special case if the code doesn't error out. The error section frees up used resources, which is required anyway.
* In case of an error, first deallocating all the used resources will reduce the chances of possible memory leaks.

Thoughts on nested BAIL_ON_ERRORs: In situations where repeatedly we have to check for errors in nested sections, it is better to wrap it into a different function call. This will also improve modularity of the code. It must be obvious that we are defining error to be anything other than a 0 exit code. This calls for a strict error code defines, where 0 is ERROR_SUCCESS and the other errors are non-zero.


A large part of our codebase is now open source - you can check it out on [GitHub](https://github.com/vmware/lightwave). I am curious how the error handling is implemented in your team, and if there is a better approach.
