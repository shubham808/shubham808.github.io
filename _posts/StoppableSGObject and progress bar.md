# StoppableSGObject class

### Overview:
The code for premature stopping framework was written in ```CMachine```. We take all of its components together and put them in a seperate object type called ```CStoppableSGObject```. This makes life easier and removes copying of same code.

### Motivation:
The stopping framework is useful in classes that do not inherit from ```CMachine``` like ```CMachineEvaluation``` this makes the idea a lot scalable in terms of usage since all any class needs to do in order to include the whole thing is inherit from ```CStoppableSGObject```. Also, it makes introducing new features, like a callback member function, a lot more easier.

### Implementation details and Design choice:

```CStoppableSGObject``` inherits from ```CSGObject```. It has all the members that were introduced in the [premature stopping framework](https://github.com/shogun-toolbox/shogun/wiki/premature-stopping) last year. My mentor added a new feature that enables us to register a lambda function as callback whenever a new iteration starts in a loop. This is done by invoking an ```SG_BLOCK_COMP``` in the callback when a condition returns True.

##### Data members and Methods:
Apart from the already present components of premature stopping framework, we introduced a new way to cancel computation of a machine. This will make testing easier and understandable. 
- ```m_callback```: It is a ```std::function<bool>``` which can call ```cancel_computation()``` along with generating block signal from the ```global_signal_handler```.
An example of callback is:
```C++
		function<bool()> callback = [this]() 
        {
			// Stop if we did more than 5 steps
			if (m_last_iteration >= 5)
			{
				get_global_signal()->get_subscriber()->on_next(SG_BLOCK_COMP);
				return true;
			}
			m_last_iteration++;
			return false;
		};
```
- ```set_callback```: setter for ```m_callback```.

### Some Thoughts:
For ```CIterativeMachine``` this provides a base for a testing mechanism. The idea is to stop a model using callback and compare it with reference results to test concurrency. This enables us to simulate a user pressing ```CTRL+C``` which will help us to write good unit tests along with providing an alternative way to cancel computation.
For non-iterative classes this means the ```COMPUTATIONS_CONTROLLERS``` macro is still usable to support the signal handler and deal with them in a systematic way without having to write all the code again.

# Progress bar macro
### Overview and motivation:
The progress bar now has an informative prefix by default. This is more verbose and makes it easier to understand and diffrentiate. This was done by adding a new ```SG_PROGRESS``` macro that appends the ```function name::class name``` prefix to the progress bar. It is only possible to get the name of the current function being executed hence, this justifies the use of a macro to obtain the name of the caller.

### Examples and applying to more algorithms:
The new, smooth progress bar looks like:
>CustomKernel::get_kernel_matrix |██████████████████████████████████████████████████████| 100.00% 0.0 seconds

>KMeansMiniBatch::minibatch_KMeans |████████████████████████████████████████████████████|100.00% 0.0 seconds

to use it in a new algorithm we can make the following changes:
- Identify a candidate loop that will need the progress bar.
- Use the macro in the begining of the loop.
```C++
for (auto e : SG_PROGRESS(range(epochs)))
```
All ```CIterativeMachine``` will automatically have a progress bar since we apply it in ```continue_train()```

### Future Work:
- Use the progress bar anywhere it seems feasible.
- Expand the ```List of Iterative Algorithms``` while searching for suitable candidates.
