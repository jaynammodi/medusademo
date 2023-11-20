[![Review Assignment Due Date](https://classroom.github.com/assets/deadline-readme-button-24ddc0f5d75046c5622901739e7c5dd533143b0c8e959d652212380cedb1ea36.svg)](https://classroom.github.com/a/OoX-Hq0F)
# Assignment 11: Synchronization

In this assignment, you will use the producer-consumer solution we discussed in class to manage
access to a bounded buffer storing candies. One group of threads will model candy factories which
generate candies one at a time and insert the candies into the bounded buffer. Another group of
threads will model kids who get one candy at a time from the bounded buffer. Your program, called
`candykids`, will accept three arguments:

```bash
./candykids <#factories> <#kids> <#seconds>
```

\# Factories:  Number of candy-factory threads to spawn.

\# Kids:       Number of kid threads to spawn.

\# Seconds:    Number of seconds to allow the factory threads to run for.

Example: `./candykids 3 1 10`

## Producer/Consumer Operations

### Main Function

Your main() function will start and control the application. Its steps are as follows:

```c
int main(int argc, char *argv[]) {
  // 1.  Extract arguments
  // 2.  Initialization of modules
  // 3.  Launch candy-factory threads
  // 4.  Launch kid threads
  // 5.  Wait for requested time
  // 6.  Stop candy-factory threads
  // 7.  Wait until there is no more candies
  // 8.  Stop kid threads
  // 9.  Print statistics
  // 10. Cleanup any allocated memory
}
```

1. **Extract Arguments**: Process the arguments passed on the command line. All arguments must be
   greater than 0. If any argument is 0 or less, display an error and exit the program.
2. **Initialization**: Do module initialization as necessary. There are at least two modules as
   explained below, the bounded buffer and statistics. If no initialization is required by your
   implementation, you may skip this step.
3. **Launch factory threads**: Spawn the requested number of candy-factory threads. To each thread,
   pass it its factory number: _0_ to _(number of factories - 1)_. _Hint:_ Store the thread IDs in
   an array because you'll need to join on them later. Also, do **not** pass each thread a reference
   to the same variable because as you change the variable's value for the next thread, there's no
   guarantee that the previous thread will have read the previous value yet. You can use an array to
   have a different variable for each thread.
4. **Launch kid threads**: Spawn the requested number of kid threads.
5. **Wait for requested time**: In a loop, call `sleep(1)`. Loop as many times as the `# Seconds`
   command line argument. Print the number of seconds running each time, exactly as `Time Xs` where
   X is the number of seconds. This shows time ticking away as your program executes.
6. **Stop factory threads**: Indicate to the factory threads that they are to finish, and then call
   join for each factory thread. See the section on [candy-factory threads](#candy-factory-thread)
   for details.
7. **Wait until there is no more candies**: While there is still candies in the bounded buffer
   (check by calling a method in your bounded buffer module), print (exactly) `Waiting for all
   candies to be consumed` and sleep for 1 second.
8. **Stop kid threads**: For each kid thread, cancel the thread and then join the thread. For
   example, if a thread ID were stored in a variable called `tid`, you would run:

   ```c
   pthread_cancel(tid);
   pthread_join(tid, NULL);
   ```

9. **Print statistics**: Call the statistics module to display the statistics. See statistics
   section below.
10. **Cleanup any allocated memory**: Free any dynamically allocated memory. You may need to call
    cleanup functions in your statistics and bounded buffer modules if they need to free any memory.

### File Structure

You must split your code up into modules by using multiple .h and .c files. You need to have the
following files under your `src/` (for `*.c`)and `include/` (for `*.h`) directories appropriately:

* `candykids.c`: Main application holding factory thread, kid thread, and `main()` function. Plus
  some other helper functions, and some `#defined` constants.
* `bbuff.h/.c`: Bounded buffer module (see [Bounded Buffer](#bounded-buffer)).
* `stats.h/stats.c`: Statistics module (see [Statistics](#statistics)).

**_Suggestions:_** The factory creates candies and the kids consume it. The candies will be stored
in a bounded buffer. To do this, you need a data type to represent a candy. The following struct is
an example:

```c
typedef struct  {
  int factory_number;
  double creation_ts_ms;
} candy_t;
```

* `factory_number` tracks which factory thread produced the candy item.
* `creation_ts_ms` tracks when the item was created. You can get the current number of milliseconds
  using the following function:

  ```c
  double current_time_in_ms(void) {
    struct timespec now;
    clock_gettime(CLOCK_REALTIME, &now);
    return now.tv_sec * 1000.0 + now.tv_nsec/1000000.0;
  }
  ```

  This code must be linked with the `-lrt` flag.

### Bounded Buffer

Create a bounded buffer module which encapsulates access to the bounded buffer. Your bounded buffer
must be implemented using the producer-consumer technique we discussed in class, taking care of
synchronization using semaphores. The public interface (the complete `bbuff.h` file) is shown below.
Note that it operates on `void *` pointers instead of directly with `candy_t` structures. This is
done so that the buffer need not know anything about the type of information it is storing. In this
case, make the buffer array (declared in the `.c` file) of type `void *` such as: `void
*da_data[DA_SIZE]`.

```c
#ifndef BBUFF_H
#define BBUFF_H

#define BUFFER_SIZE 10

void bbuff_init(void);
void bbuff_blocking_insert(void* item);
void *bbuff_blocking_extract(void);
_Bool bbuff_is_empty(void);

#endif
```

For example, an item can be inserted into the buffer with the following code which will dynamically
allocate one candy element (pointer stored in `candy`), set the fields of the `candy`, and then call
the bounded buffer function to insert it into the bounded buffer.

```c
void foo() {
  candy_t *candy = malloc(...);
  candy->factory_number = ...;
  candy->creation_ts_ms = ...;
  bbuff_blocking_insert(candy);
}
```

The `bbuff_init()` function is used to initialize the bounded buffer module if anything needs to be
initialized. Think of it like the constructor for your module: if this were object-oriented C++ then
the constructor would do any needed initialization. But, in C there are no constructors, so
initialization functions are often used. If you do not need to initialize anything in your module,
you can omit the `bbuff_init()` function entirely.

Note that a program with threads and synchronization exhibits non-deterministic behavior. Thus, even
if your program has synchronization problems, a single run may not reveal those problems. You need
to run many times and consistently show no problems.

### Candy-Factory Thread

Each candy-factory thread should do the following.

1. Loop until main() signals to exit (see [Thread Signaling](#thread-signaling)).
    * Pick a number of seconds which it will (later) wait. The number is randomly selected between 0
      and 3 (inclusive).
    * Print a message exactly as: `Factory 0 ships candy & waits 2s` (where 0 is the thread number
      and 2 is the number of seconds it will wait)
    * Dynamically allocate a new candy item and populate its fields.
    * Add the candy item to the bounded buffer.
    * Sleep for number of seconds identified in the first step.
2. When the thread finishes, print a message exactly as: `Candy-factory 0 done` (where 0 is the
   thread number).

#### Thread Signaling

The thread will end when signaled to do so by `main()`. You need to use thread synchronization
primitives in order to do this. Choose something appropriate for this purpose.

### Kid Thread

Each kid thread should loop forever and during each iteration, do the following:

* Extract a candy item from the bounded buffer; this will block until there is a candy item to
  extract.
* Process the item. Initially you may just want to `printf()` it to the screen; in the next section,
  you must add a statistics module that will track what candies have been eaten.
* Sleep for either 0 or 1 seconds (randomly selected).

The kid threads are canceled from `main()` using `pthread_cancel()`. When this occurs, it is likely
that the kid thread will be waiting on the lock for the bounded buffer. This should not cause
problems.

### Statistics

Create a statistics module that tracks the following:

* The number of candies each factory creates. Called from the candy-factory thread.
* The number of candies that were consumed from each factory.
* For each factory, the min, max, and average delays for how long it took from the moment the candy
  was produced (dynamically allocated) until consumed (eaten by the kid). This will be done by the
  factory thread calling the stats code when a candy is created, and the kid thread calling the
  stats code when an item is consumed.

`stats.h` should have the following:

```c
#ifndef STATS_H
#define STATS_H

void stats_init(int num_producers);
void stats_cleanup(void);
void stats_record_produced(int factory_number);
void stats_record_consumed(int factory_number, double delay_in_ms);
void stats_display(void);

#endif
```

Internally in `stats.c`, you will likely need to track a number of values for each candy-factory.
For this, you should create a `struct` with all required fields, and then build an array of such
`struct`s (one element for each candy-factory). The `stats_init()` function can initialize your data
storage and get it ready to process produced and consumed events (via the respective functions). The
`stats_cleanup()` function is used to free any dynamically allocated memory. This function should be
called just before `main()` terminates.

#### Displaying Stats Summary

When the program ends, you must display a table summarizing the statistics gathered by the program.
It should look exactly like the following:

```c
Statistics:
Factory#   #Made  #Eaten  Min Delay[ms]  Avg Delay[ms]  Max Delay[ms]
       0       5       5        0.60498     2602.81274     5004.28369
       1       5       5        0.40454     2202.97290     5005.06494
       2       7       7        0.60107     2287.86067     4004.16162
       3       8       8     1001.12012     2377.36115     5004.13159
       4       5       5        0.40186     2202.63008     5005.38330
       5       4       4     1003.22095     2503.94049     4006.16309
       6       5       5     1003.24487     2603.35894     4005.19873
       7       6       6     3002.30640     3836.61743     4005.16089
       8       4       4     3001.74048     3753.03259     5004.31177
       9       4       4     3002.76660     4253.44440     5005.13550
```

* Factory #: Candy factory number. In this example, there were 10 factories.
* \# Made: The number of candies that each factory reported making (as per the call from the
  candy-factory thread).
* \# Eaten: The number of candies which kids consumed (as per the call from the kid threads).
* Min Delay\[ms\]: Minimum time between when a candy was created and consumed over all candies
  created by this factory. Measured in milliseconds.
* Avg Delay\[ms\]: Average delay between this factory's candy being created and consumed.
* Max Delay\[ms\]: Maximum delay between this factory's candy being created and consumed.

**Requirements:** The table must be nicely formatted (as above).
For the header, use the following: `printf("%8s%10s%10s%20s%20s%20s\n", ...);`
For the data rows, use: `printf("%8d%10d%10d%20.5f%20.5f%20.5f\n", ...);`
If the #Made and #Eaten columns don't match, print an error: `ERROR: Mismatch between number made
and eaten.`

## Grading Distribution

This assignment does not have a tester. Instead, the grading will be done according to the following
criteria.

* [40] Program execution and threads
    * Launch the correct number of factory and kid threads.
    * Correct operation of factory and kid threads.
    * Correctly signal factories to exit.
    * Correctly wait on candy in buffer.
    * Correctly cancel kids.
* [15] Bounded buffer and synchronization
    * Correct operation and synchronization of bounded buffer.
* [15] Statistics
    * Correctly formatted table
    * Correctly accumulated data
* [10] Correct memory access
    * You should not have any memory problems. We will test this with Clang's memory sanitizer.
* [10] No data races
    * You should not have any data races. We will test this with Clang's thread sanitizer.
    * Due to potential non-deterministic behavior, we will run many times and check data races.
* [10] File structure
    * Clean interface to C modules
    * CMake builds correctly.

Code that does not compile gets a 0.

## Acknowledgment

Created by Mohamed Hefeeda, modified by Brian Fraser, Keval Vora, Tianzheng Wang, and Steve Ko.
