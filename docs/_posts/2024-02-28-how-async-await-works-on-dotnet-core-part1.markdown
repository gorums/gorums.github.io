---
layout: post
title:  "How async/await works on dotnet core. Part 1"
date:   2024-02-28 09:56:51 -0500
categories: dotnet
tags: dotnet
author: "Alejandro Ferrandiz"
---
> Note: The way async/await is handled internally by the runtime is an implementation detail, meaning it can change in the future. You should avoid basing your code on the specific internal implementation. However, this doesn't limit how you can use it.

Ok, Let’s talk about the subject

While treating asynchronous calls with **async/await** as a 'black box' has served me well for 10 years, that is the power of the abstraction. Understanding the mechanics within can empower you to make more informed decisions about when and how to use this powerful tool.

That bring me to the question. How **async/await** is implemented?

While it may seem magical, magic does not exist, could be complex inside, hard to understand, but it is not magic at the end.

When you use an **async/await** method, the compiler actually transforms it behind the scenes. Basically the compiler transform our method to a struct with a state machine pattern (a class in debug mode) with a *MoveNext* method like an iterator, this struct implement the *IAsyncStateMachine* interface.

Let's check a simple example of a delay task helper and the compiler transformation.

{% highlight csharp %}
// Our delay helper class
public static class DelayHelper
{
    // Delay 10 seconds
    public static async Task DelayBy10Sec()
    {
        int tenSec = 10 * 1000;
        
        await Task.Delay(tenSec);
    }
}
{% endhighlight %}

Compiler transformation:

{% highlight csharp %}
// this is what the compiler transformation generate from our code
public static class DelayHelper
{
    [CompilerGenerated]
    private sealed class <DelayBy10Sec>d__0 : IAsyncStateMachine
    {
        public int <>1__state;

        public AsyncTaskMethodBuilder <>t__builder;

        private int <tenSec>5__1;

        private TaskAwaiter <>u__1;

        private void MoveNext()
        {
            int num = <>1__state;
            try
            {
                TaskAwaiter awaiter;
                if (num != 0)
                {
                    <tenSec>5__1 = 10000;
                    awaiter = Task.Delay(<tenSec>5__1).GetAwaiter();
                    if (!awaiter.IsCompleted)
                    {
                        num = (<>1__state = 0);
                        <>u__1 = awaiter;
                        <DelayBy10Sec>d__0 stateMachine = this;
                        <>t__builder.AwaitUnsafeOnCompleted(ref awaiter, ref stateMachine);
                        return;
                    }
                }
                else
                {
                    awaiter = <>u__1;
                    <>u__1 = default(TaskAwaiter);
                    num = (<>1__state = -1);
                }
                awaiter.GetResult();
            }
            catch (Exception exception)
            {
                <>1__state = -2;
                <>t__builder.SetException(exception);
                return;
            }
            <>1__state = -2;
            <>t__builder.SetResult();
        }

        void IAsyncStateMachine.MoveNext()
        {
            //ILSpy generated this explicit interface implementation from .override directive in MoveNext
            this.MoveNext();
        }

        [DebuggerHidden]
        private void SetStateMachine([Nullable(1)] IAsyncStateMachine stateMachine)
        {
        }

        void IAsyncStateMachine.SetStateMachine([Nullable(1)] IAsyncStateMachine stateMachine)
        {
            //ILSpy generated this explicit interface implementation from .override directive in SetStateMachine
            this.SetStateMachine(stateMachine);
        }
    }

    [NullableContext(1)]
    [AsyncStateMachine(typeof(<DelayBy10Sec>d__0))]
    [DebuggerStepThrough]
    public static Task DelayBy10Sec()
    {
        <DelayBy10Sec>d__0 stateMachine = new <DelayBy10Sec>d__0();
        stateMachine.<>t__builder = AsyncTaskMethodBuilder.Create();
        stateMachine.<>1__state = -1;
        stateMachine.<>t__builder.Start(ref stateMachine);
        return stateMachine.<>t__builder.Task;
    }
{% endhighlight %}

You can verify and play with the compiler transformation using [shaplab.io](https://sharplab.io/#v2:CYLg1APgAgDABFAjAVgNwFgBQWoGYGIBsCATHACICmANgIYCeAEjQA6UBOWA3lnHwviTEoADgTEqdegCF6iGAGVKAYwAUASl78emfnrgBLAHYAXOCcpGlyuAF448uACoHMN6jgB6T67gBnFQB7I2A/LX04cP0oAE5xADpJBlULKxV1DF1+AF8sbKA===).

Lets take a look to the firs part:

{% highlight csharp %}
public static class DelayHelper
{
    [CompilerGenerated]
    private sealed class <DelayBy10Sec>d__0 : IAsyncStateMachine
    {
        public int <>1__state;

        public AsyncTaskMethodBuilder <>t__builder;

        private int <tenSec>5__1;

        private TaskAwaiter <>u__1;
        
        private void MoveNext()
        {
          ...
        }
        ...
    }
}
{% endhighlight %}

We have a private sealed class <*DelayBy10Sec>d__0* that is used internally. This class acts as a state machine, but in *non-debug* mode, it's optimized as a struct to improve performance. This allows the state machine to reside on the stack, avoiding heap allocation and reducing memory usage.

This state machine implements the *IAsyncStateMachine* interface, ensuring it adheres to a specific set of rules. Notice its unusual name, *<DelayBy10Sec>d__0*, and field names starting with *<>*. These intentional choices are made to prevent conflicts with existing valid names within your code, ensuring a smooth operation.

Within this state machine, a field named *<>1__state* plays a crucial role. It acts as a guide, indicating which code block to execute next within the *MoveNext* method. Think of it as a bookmark, keeping track of the operation's progress and ensuring each step is executed in the correct sequence.

The state machine also stores **TaskAwaiter** fields, one for each await keyword used in your code. In this case, since your code only employs a single await, there's just one such field.

{% highlight csharp %}
await Task.Delay(tenSec);
{% endhighlight %}

The **TaskAwaiter** plays a key role in managing asynchronous operations. It essentially holds the awaited task and schedules a callback to our state machine's *MoveNext* method once the task completes. This ensures the state machine resumes correctly at the appropriate point using the state as a checkpoint.

We’ll talk about **TaskAwaiter** in another entry in more details.

Another crucial field is the **AsyncTaskMethodBuilder** *<>t__builder*. This struct acts as an abstraction, simplifying task management, for example is going to keep the synchronization context for the ambient data that we need to flow between thread, We’ll talk about it more in details in other entry.

But where is our code?

Our code is essentially transformed into a method that initializes the state machine, creates the **AsyncTaskMethodBuilder**, and starts the state machine using the builder. This builder then calls the *MoveNext* method, which contains the remaining logic of our original method.

Les take a look at this method

{% highlight csharp %}
[NullableContext(1)]
[AsyncStateMachine(typeof(<DelayBy10Sec>d__0))]
[DebuggerStepThrough]
public static Task DelayBy10Sec()
{
    <DelayBy10Sec>d__0 stateMachine = new <DelayBy10Sec>d__0();
    stateMachine.<>t__builder = AsyncTaskMethodBuilder.Create();
    stateMachine.<>1__state = -1;
    stateMachine.<>t__builder.Start(ref stateMachine);
    return stateMachine.<>t__builder.Task;
}
{% endhighlight %}

This generated method shares the same signature as your original method (excluding the **async** keyword) and includes an attribute specifying the state machine it uses: *<DelayBy10Sec>d__0*.

Within the method's body, the private state machine is instantiated, its fields are initialized to -1 representing the initial state, and finally, the builder is created and started.

Finally, the method returns the **Task** created by the builder. This **Task** acts as a placeholder for the ongoing operation and can be awaited by the code that called our *DelayBy10Sec* method.

Let's now dive into the *MoveNext* method implementation. While the code itself is fairly straightforward, I've added comments to better understanding. Remember, the first call to MoveNext happens when the builder invokes the *Start* method within *DelayBy10Sec*.

{% highlight csharp %}
stateMachine.<>t__builder.Start(ref stateMachine);
{% endhighlight %}

{% highlight csharp %}
private void MoveNext()
{
    int num = <>1__state;
    try
    {
        TaskAwaiter awaiter;
        // state can be -1, 0 and -2 based on this example. state 0 means the awaiter was called,
        // -1 if is the initial state or -2 if our task is done, in this case we can have only -1 and 0
        if (num != 0)
        {
            // set the 10 sec
            <tenSec>5__1 = 10000;
            // we call the Task and obtain the awaiter
            awaiter = Task.Delay(<tenSec>5__1).GetAwaiter();
            // we validate if the awaiter is completed here, if that is the case
            // means that we can treate this call as a sync one and we dont need
            // to allocate space in the heap calling AwaitUnsafeOnCompleted method for the builder
            if (!awaiter.IsCompleted)
            {
                // if is not completed the awaiter change state to 0
                num = (<>1__state = 0);
                <>u__1 = awaiter;
                <DelayBy10Sec>d__0 stateMachine = this;
                // schedule a continuation callback to the MoveNext() when the awaiter is done
                <>t__builder.AwaitUnsafeOnCompleted(ref awaiter, ref stateMachine);
                // here we return, the awaiter is going to call again when it finished
                return;
            }
        }
        else
        {
            // if state is 0 means that the awaiter is done, and it called MoveNext as a callback
            awaiter = <>u__1;
            <>u__1 = default(TaskAwaiter);
            num = (<>1__state = -1);
        }
        // in this point the awaiter is completed because was done sync or async
        // if there were and exception inside the awaiter GetResult() is going to
        // re-trowing here that is why we have a try catch
        awaiter.GetResult();
    }
    catch (Exception exception)
    {
        // we set the state to completed and save the exception inside the task
        // that the builder is handle for us
        <>1__state = -2;
        <>t__builder.SetException(exception);
        return;
    }
    // in this point the awaiter is completed without exception and we can mark our task
    // completed calling the builder method SetResult()
    <>1__state = -2;
    <>t__builder.SetResult();
}
{% endhighlight %}

A potentially confusing aspect is the validation of the awaiter to check if its task has completed. Here's why: when you call a task and retrieve its awaiter...

{% highlight csharp %}
awaiter = Task.Delay(<tenSec>5__1).GetAwaiter();
{% endhighlight %}

The task might complete before the validation finishes. In such scenarios, scheduling a continuation callback to *MoveNext* would be unnecessary. This allows the task to complete synchronously

What I’m missing still here:

- What is a TaskAwaiter?
- Why do we need a builder and what is it role?
- What is the compiler transformation for a ValueTask?
- If the awaiter throw an exception how we can handle it?
- What is the compiler transformation for a async void and why is problematic if the awaiter throw an exception?

I’ll create at least another entry to explain a bit about those questions trying to avoid to many implementation details and keeping it simple.

In conclusion, async/await in .NET Core is a powerful feature which simplifies asynchronous programming. It relies on compiler transformations, creating a state machine that manages the flow of control in the asynchronous method. Key components include TaskAwaiter and the builder(AsyncTaskMethodBuilder).