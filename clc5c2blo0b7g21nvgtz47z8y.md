# A case against the Hungarian Notation on Android


In this article we take a look at the Hungarian Notation and why it is in my opinion a bad idea. We take a look at possible causes why this habit started and dispel a few of the common myths about its usefulness. 
 
## What is Hungarian Notation 
First of all, let's take a look at what exactly we mean by Hungarian Notation on Android. Wikipedia, in it's usual dry manner states that: 
> Hungarian notation is an identifier naming convention in computer programming, in which the name of a variable or function indicates its intention or kind, and in some dialects its type. [...] 
> In Hungarian notation, a variable name starts with a group of lower-case letters which are mnemonics for the type or purpose of that variable, followed by whatever name the programmer has chosen; this last part is sometimes distinguished as the given name. The first character of the given name can be capitalized to separate it from the type indicators. 
 
In the Android word, this usually takes the form of prepending a lowercase `m` in front of member variables and a lowercase `s` in front of static variables. That is to say, instead of calling a variable `name` you would call it `mName` if it was a member or `sName` if it were a static. 
 
## Why it is used 
The question of why this habit is so widespread, despite some quite influential people and organizations taking quite strong stances against it (such as [Jake Wharton](http://jakewharton.com/just-say-no-to-Hungarian-notation/) or [Google](https://google.github.io/styleguide/javaguide.html#s5.1-identifier-names)) is a tricky one. I found that people in general and programmers in particular tend to sometimes follow blindly what more experienced members of their community will tell them without questioning *why*. I think that this is the only reason this plague of extra characters is spreading. 
 
I wanted to include a reference to the famous [5 monkeys and a ladder experiment](http://www.throwcase.com/2014/12/21/that-five-monkeys-and-a-banana-story-is-rubbish/) but it turns out that's an outright fabrication. Still, the point raised there is valid and outlines our tendency of blindly following the herd. As we will see below, in the day and age of IDEs there is no valid reason to continue using it and I will try to prove that in fact is it even slightly harmful. 
 
## Common reasons given for its use 
Let's now look at a few reasons that people give when asked why they use the Hungarian Notation: 
 
**It helps distinguish between local, member and static variables** - In the era before GUIs this used to be true, and it is in fact the reason why Hungarian Notation first came into being. Not only was the scope hard to ascertain, but also the type of the variable, hence you would see conventions such as ``dwLightYears`` (double-word) or ``szName`` (zero-terminated string). However, nowadays a simple glance at the variable will tell you if the variable is static or not, while hovering it will tell you the type. Command-clicking it will take you to the declaration, thus giving you full info: 
![](/content/images/2017/06/Screen-Shot-2017-06-06-at-23.24.55.png) 
 
**The Android style guide requires it** -  
There is no such thing as an "Android style guide". Oracle's Java style guide makes no mention of the Hungarian notation, while Google explicitly and actively forbids Hungarian notation in their [Java style guide](https://google.github.io/styleguide/javaguide.html#s5.1-identifier-names). The only Android style guide that requires it is the AOSP (Android Open Source Project). If you are not writing code for that project, you should not care about their code conventions. 
 
**It increases code readability** - This one is a matter of habit more than anything else. I find it quite the opposite to be honest: adding an extra `m` at the beginning of a variable only makes it harder for me to determine which variable we are talking about. 
 
## Why it is harmful 
As we saw, there doesn't seem to be any reason to use the Hungarian Notation, which in itself should be enough to forgo its use. However, let's have a look at why it is actually bad for you - requiring you to do more work or otherwise make your life harder. 
 
**GSON serialization/deserialization** - Normally, if you have a JSON files with fields such as `name`, `address`, `GPA` etc and create a POJO with those exact same fields, GSON can automagically convert one to the other. Not so if you are using the Hungarian notation, because GSON can't guess that `mName` is the same thing as `name`. Sure, there is a workaround in using the `@SerializedName` annotation for each and every field, but that is a lot of needless work and it reduces readability a lot: 

```java 
public class CartResponse { 
    @SerializedName("entry") 
    private CartEntry mEntry; 
 
    @SerializedName("quantityAdded") 
    private int mQuantityAdded; 
 
    @SerializedName("quantity") 
    private int mQuantity; 
 
    @SerializedName("statusCode") 
    private String mStatusCode; 
 
    @SerializedName("missingCSW") 
    private int mMissingCSW; 
} 
``` 
**Automatic getter and setter generation** - this will generate `getmEntry` instead of `getEntry`. Sure you can go and delete the `m` from all the getters and setters, but that rather defeats the purpose of the entire features, doesn't it? 
 
**IDE code completion shenanigans** - since 99% of your variables start with `m` (and the rest with `s`) you now have to type 2 letters (instead of just one) after command-space for the IDE to have a chance of knowing which variable you want. Might not seem like a lot, but it does reduce your typing speed for no reason whatsoever. 
 
## Conclusion 
So, should you go and refactor your entire codebase to remove the Hungarian Notation? Probably not. It's a minor nuisance and refactoring just for that is simply not worth it. However I do encourage all new projects to ditch this antiquated habit and just use the natural naming convention. 
 