---
layout: post
title: ForkJoinThreds,ForkJoinTasks,ForkJoinPool
---

>History

One of the best to ways understand how something works is to start with why that was build in the first place. This gives a very strong foundation for getting the intuition of how it works. Before we get into the details of how ForkJoinThreads work, lets try to build a story of why that was build in the first place.

<p>
    Lets take an example of adding all the elements of the array, I mean a really large array say 100000 elements.
</p>

<i>Note that the goal is not to build an optimised solution fo adding numbers but to get the intuition of the forkjoin concept.</i>

> Solution1 : Iterative Solution

A simple way of adding all the elements of the array is by iterating the array elements and adding one after the other.

{% highlight text %}

public class Iter {

	public void invoke() {

		ArrayList<Integer> list = new ArrayList<>();
		int i = 0;
		while (i < 10) {
			list.add(i);
			i++;
		}

		long stime = new Date().getTime();
		System.out.println("Start Time = " + stime);
		Integer sum = 0;
		i = 0;
		while (i < 10) {
			sum += list.get(i);
			i++;
		}

		long etime = new Date().getTime();
		System.out.println("SUM = " + sum + "\nEnd Time=" + etime);
		System.out.println("TotalTime taken = " + (etime - stime));

	}

	public static void main(String[] args) {
		Iter iter = new Iter();
		iter.invoke();
	}

}


{% endhighlight %}

An other way of solving this is by recursive appproach using divide and conquer technique.

{% highlight text %}
initialise array with 100000 elements
THRESHOLD=5000 (The smallest unit of work that can be solved and result can be returned)
function int add(startIndex,endIndex)
				int sum = 0;
				if ((endIndex - startIndex) <= THRESHOLD) {
					System.out.println("Computing Sum = " + sum);
					while (startIndex <= endIndex) {
						sum += array[startIndex++];
					}
				}
				else {
					int div = (startIndex + endIndex) / 2;
					sum+=add(startIndex, div);
					sum+=add(div + 1, endIndex);
				}
                return sum;
			}
end

{% endhighlight %}

Is this recursive approach efficient than the iterative one? The answer is unfortunately no. Yes its cool to write recursive functions! Unfortunately the above method is still single threaded and does not include any parallelism.

> Solution2: Adding a bit of parallelism

Lets say we have a pool of threads maintained by executorservice and a queue attached to add jobs and the executor service takes care of managing the threads (Sorry folks about the java terminologies.Please relate those to any programming languages you are aware of). For the sake of arguement lets assume that there is a way of returning values to the calling thread(Callables and Futures in java).But the calling thread has to wait untill the new thread finishes its job and returns the value.

{% highlight text %}

public class ExecutorServiceTest2 {

	public void invoke() {

		ArrayList<Integer> list = new ArrayList<>();
		final ExecutorService executor = Executors.newFixedThreadPool(10);
		int i = 0;
		while (i < 10) {
			list.add(i);
			i++;
		}

		class AddTask implements Callable<Integer> {
            int THRESHOLD = 2;
			int startIndex;
			int endIndex;
            int sum = 0;

			public AddTask(int start, int end) {
				this.startIndex = start;
				this.endIndex = end;
			}

			@Override
			public Integer call() {

				if ((endIndex - startIndex) <= THRESHOLD) {
					System.out.println("Computing Sum = " + sum);
					while (startIndex <= endIndex) {
						sum += list.get(startIndex++);
					}
				}
				else {
					int div = (startIndex + endIndex) / 2;
					AddTask firstHalf = new AddTask(startIndex, div);
					AddTask secondHalf = new AddTask(div + 1, endIndex);
					Future<Integer> future1 = executor.submit(firstHalf);
					Future<Integer> future2 = executor.submit(secondHalf);

					try {
						sum += future1.get();
						sum += future2.get();
					}
					catch (InterruptedException | ExecutionException e) {
						e.printStackTrace();
					}

				}
				System.out.println("Returning Sum = " + sum);
				return Integer.valueOf(sum);
			}
		}

		System.out.println("Starting Runnable Test");
		Future<Integer> future = executor.submit(new AddTask(0, list.size() - 1));
		try {
			Integer sum = future.get();
			System.out.println("SUM = " + sum);
		}
		catch (InterruptedException | ExecutionException e) {
			e.printStackTrace();
		}
	}

	public static void main(String[] args) {
		System.out.println("Starting Main");
		ExecutorServiceTest2 executorTest = new ExecutorServiceTest2();
		executorTest.invoke();
	}

}


{% endhighlight %}

Note that the executor service had enough threads in its pool to add 10 elements.If you just increase the array size to 1000 or reduce the thread pool size to 2, Then all the threads in the pool are waiting and we end up in thread starvation.There are no threads in the pool to take up new tasks from the queue.One solution is to have as many threads to take up new tasks from the queue.But this would not scale with increasing array size.</p>

<i>We wish the threads which are waiting for the result pause its current job state and start picking up new tasks and later come back and continue from the point where they had paused the tasks before.</i>

>Welcome the FORKJOINTHREDS

The above solution had 2 main issues.First, Huge number of threads are created.Second,for problems where there is a dependency among tasks,as the number of tasks increase so as the number of threads required.Lets look at the following approach where fork join threads are used.

{% highlight code %}

public class ForkJoinTest {

	public void invoke() {

		ArrayList<Integer> list = new ArrayList<>();
		int i = 0;
		while (i < 10) {
			list.add(i);
			i++;
		}

		class AddTask extends RecursiveTask<Integer> {
			int THRESHOLD = 2;
			int startIndex;
			int endIndex;
            int sum = 0;
			public AddTask(int start, int end) {
				this.startIndex = start;
				this.endIndex = end;
			}

			@Override
			protected Integer compute() {

				if ((endIndex - startIndex) <= THRESHOLD) {
					System.out.println("Computing Sum = " + sum);
					while (startIndex != endIndex) {
						sum += list.get(startIndex++);
					}
				}
				else {
					int div = (startIndex + endIndex) / 2;
					AddTask firstHalf = new AddTask(startIndex, div);
					AddTask secondHalf = new AddTask(div + 1, endIndex);
					System.out.println("Creating Task firstHalf:: " + firstHalf.startIndex + firstHalf.endIndex);
					System.out.println("Creating Task secondHalf:: " + secondHalf.startIndex + secondHalf.endIndex);
					firstHalf.fork();
					secondHalf.fork();
					sum += firstHalf.join();
					sum += secondHalf.join();
				}
				System.out.println("Returning Sum = " + sum);
				return Integer.valueOf(sum);
			}
		}

		final ForkJoinPool pool = new ForkJoinPool(1);
		try {
			System.out.println("Starting ForkJoin Test");
			Integer sum = pool.invoke(new AddTask(0, list.size() - 1));
			System.out.println("SUM = " + sum);
		}
		finally {
			pool.shutdown();
		}

	}

	public static void main(String[] args) {
		System.out.println("Starting Main");
		ForkJoinTest forkJoinTest = new ForkJoinTest();
		forkJoinTest.invoke();
	}

}

{% endhighlight %}

The idea here is to divide the array and fork new tasks untill the THRESHOLD is reached. On reaching the THRESHOLD return the result, and start joining results up the way untill the final sum is calculated. The previous Executorservice also did the same thing. The difference is in the previous method when a thread called get on the future (sum += future1.get();) the thread was blocked untill the result was returned by the future.That meant the calling thread cannot take new tasks in the executor pool. This is where forkjoin pool shines.

Before we dive into the way the above example works.Lets try to understand the algorithm behind the forkjoinpool.

>Stealing ain't bad: WorkSteal




>Stats

{% highlight code %}

Iterator:

Elements::10000          TimeTaken::3
Elements::100000         TimeTaken::5
Elements::1000000        TimeTaken::7

ForkJoin:

THRESHOLD = 2;

Elements::10000          TimeTaken::10   Threads::2
Elements::10000          TimeTaken::9    Threads::4
Elements::10000          TimeTaken::10   Threads::8

Elements::100000          TimeTaken::33   Threads::2
Elements::100000          TimeTaken::33   Threads::4
Elements::100000          TimeTaken::27   Threads::8

Elements::1000000         TimeTaken::300   Threads::2
Elements::1000000         TimeTaken::278   Threads::4
Elements::1000000         TimeTaken::268   Threads::8

THRESHOLD = 1000;

Elements::10000          TimeTaken::3    Threads::2
Elements::10000          TimeTaken::4    Threads::4
Elements::10000          TimeTaken::4    Threads::8

Elements::100000          TimeTaken::9   Threads::2
Elements::100000          TimeTaken::9   Threads::4
Elements::100000          TimeTaken::9   Threads::8

Elements::1000000         TimeTaken::25   Threads::2
Elements::1000000         TimeTaken::35   Threads::4
Elements::1000000         TimeTaken::55   Threads::8


THRESHOLD = 10000;

Elements::10000          TimeTaken::3    Threads::2
Elements::10000          TimeTaken::3    Threads::4
Elements::10000          TimeTaken::3    Threads::8

Elements::100000          TimeTaken::7   Threads::2
Elements::100000          TimeTaken::7   Threads::4
Elements::100000          TimeTaken::8   Threads::8

Elements::1000000         TimeTaken::27   Threads::2
Elements::1000000         TimeTaken::34   Threads::4
Elements::1000000         TimeTaken::40   Threads::8


{% endhighlight %}




