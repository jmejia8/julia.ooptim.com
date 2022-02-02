

+++
title = "Working with Threads: A simple application"
hascode = true
date = Date(2022, 2, 02)
rss = "Introduction to threads in Julia from a naive point of view."
+++
@def tags = ["syntax", "code"]


# Working with Threads: A simple application

\toc

Running multiple tasks simultaneously becomes convenient when you have powerful computer
able to launch at least four threads. This post will remember out to implement **mutex**
tasks because I wanted to plot stuff in different threads but I obtained a **segmentation fault**
due to `Plots` (on `GR` backend) cannot plot two or more figures at the same time in different threads.

## Using Multiple Threads


Let's run multiple tasks in different threads. 

### Initialize Julia

Start Julia indicating the number of threads available (**4** in my case) for the simulations:

```shell
$ julia --threads 4
```

### Code the expensive task

Here, `my_expensive_task` implements a very expensive task that will run a thread.

```julia
function my_expensive_task(μ, σ, n)
    sleep(3)
    return μ .+ σ^2 * randn(n, n)
end

```

### Saving the data
This function will save the data obtained from `my_expensive_task`.

```julia
function save_data(data, i)
    sleep(1)
    @show i
    display(data)
    return true
end

```

### Run the experiments

The `main` function will run `my_expensive_task` in 4 threads and save the obtained data.

```julia
function main()
    n_experiments = 16
    n = 5
    Threads.@threads for i in 1:n_experiments
        μ = rand()
        σ = rand()
        M = my_expensive_task(μ, σ, n)
        save_data(M, i)
    end
end

```

## Threads and Mutex

Here, [Mutex](https://rosettacode.org/wiki/Mutex#:~:text=Mutexes%20are%20typically%20used%20to,synchronization%20primitive%20exposed%20to%20deadlocking.)
are used to execute tasks only once via synchronization.

In Julia there exists a useful synchronization object called `ReentrantLock()` to lock
let only one execution of the desired task. The implementation of a mutex in Julia is very simple.


```julia-repl
julia>?ReentrantLock()

  ReentrantLock()

  Creates a re-entrant lock for synchronizing Tasks. The same task can acquire the lock as many times as required.
  Each lock must be matched with an unlock.

  Calling 'lock' will also inhibit running of finalizers on that thread until the corresponding 'unlock'. Use of the
  standard lock pattern illustrated below should naturally be supported, but beware of inverting the try/lock order
  or missing the try block entirely (e.g. attempting to return with the lock still held):

  lock(l)
  try
      <atomic work>
  finally
      unlock(l)
  end
```



### Implementing Mutex


Assume that you only can save data once at a time. To prevent that different threads
call `save_data` at the same time, let's `lock` that action. Return to the `main` function
and add the following lines to implement the mutex.

```julia
function main()
    n_experiments = 16

    # data size
    n = 5 
    mutex = ReentrantLock() # used to save data

    Threads.@threads for i in 1:n_experiments
        μ = rand()
        σ = rand()

        # do simulations
        M = my_expensive_task(μ, σ, n)

        # using mutex while saving data
        lock(() -> save_data(M, i), mutex)
        
    end
end
```


**Note** that you don not need to unlock the mutex due to `lock` does it automatically
when `save_data` returns a value.
