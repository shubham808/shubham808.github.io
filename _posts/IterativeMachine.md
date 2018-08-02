---
layout: post
title: Iterative Machine Guide
---
### Overview:
Iterative machine enables us to write iterative algorithms that are prematurely stoppable. This means users can cancel the training process any time. The model is still usable and concurrent. This model can then be applied to test data, compared with reference weights and, if needed, can *resume* training from where it left off earlier. Iterative Machine framework makes using cancelled state models more robust.
The idea here is to have iterative models implement only a single iteration of the main training loop instead. This will be called from a while loop in [CIterativeMachine](https://github.com/shogun-toolbox/shogun/tree/develop/src/shogun/machine/IterativeMachine.h#62) class now. 

### Motivation:
The previous idea was to have different callbacks like ```on_next```, ```on_pause``` which will be called based on the user choice obtained from the ```ShogunSignalHandler``` prompt. This proves a bit restrictive with respect to what the user can acutally do. Furthermore, it was not possible to define such behaivour from an interface like python. The user *has* to write some c++ code. Therefore, it is a better approach to allow the user to just cancel training whenever he wants and then also to resume training from where it was left off. This is more flexible to the user and developer as it removes the element of *guessing* what is meaningful for a user.

### Implementation details:

```CIterativeMachine``` is a mixin class. This means it can inherit from some other class which is passed to it through a template argument to its constructors. Iterative models will now inherit from ```CIterativeMachine<CMockMachine>``` instead of being a direct subclass of ```CMockMachine```.
##### data members:
- ```m_current_iteration```: The current iteration count.
- ```m_max_iteration```: Maximum number of iterations allowed.
- ```m_complete```: If the model has completed training and converged.
##### methods:
- ```init_model```: Virtual, must be written in subclass, this is called before training loop begins to initialize all members.
- ```continue_training```: Contains the main training loop which updates ```m_current_iteration``` and called the ```iteration``` method.
- ```iteration```: Virtual, This must be written in subclass and implements a single iteration of training loop.
- ```end_training```: An optional method called after training to clean member states or giving warnings etc.

### Example:
Below is a cpp example of a fake iterative model. 
```c++
#include <shogun/base/init.h>
#include <shogun/base/some.h>
#include <shogun/labels/BinaryLabels.h>
#include <shogun/machine/IterativeMachine.h>
#include <iostream>

using namespace shogun;
using namespace std;

// Mock Iterative Algorithm which implements fake methods
class MockModel : public CIterativeMachine<CMachine> 
{
public:
	MockModel() : CIterativeMachine<CMachine>() {}
	~MockModel() {}

protected:
	virtual void init_model(CFeatures * data) 
	{
	    // Initialize members
	    x = 0;
      m_max_iterations = 10;
	}
	virtual void iteration()
	{
	    // Single iteration of training loop
	    x = x + m_labels->get_value(0);
	}
    virtual void end_training()
    {
      // clean member variable states or give warnings and information
      cout<<x<<endl;
      x = 0;
    }
protected:
    float64_t x;
};

int main() 
{
	init_shogun_with_defaults();
  
	// Set up binary labels
	auto labels = some<CBinaryLabels>(SGVector<float64_t>({1, -1}));

	MockModel a;
	a.set_labels(labels);
    cout<<"Training Start..."<<endl;
	a.train();
    /* Press CTRL+C before training is complete. Another way to stop training 
    is to pass a callback that will trigger can trigger a signal.
    */
    // Here you can use the pre-trained model. For example we can apply on test data, serialize the model etc.
    cout<<"Resuming Training"<<endl;
    a.continue_train();

	return 0;
}
```

There are two ways to Prematurely stop an algorithm. The user can press ```CTRL+C``` or the user can write a callback method that will trigger a signal. For more details on second method see [this patch](https://github.com/shogun-toolbox/shogun/pull/4293). From python the code will look like:

```Python
from shogun import Perceptron
Perceptron.train(feats)
# Press CTRL+C and you will see something like
# [ShogunSignalHandler] Immediately return to prompt / Prematurely finish computations / Pause current computation / Do nothing (I/C/P/D)?
# Type "C"
# Perform operations like apply on test data, save current model etc
Perceptron.continue_train()
```
### Applying Iterative Machine to more Algorithms:

To use the features of ```CIterativeMachine``` with a new Algorithm we can make the following changes:

- Use existing machine members (For eg: ```m_w```, ```bias``` of ```CLinearMachine``` for weights and bias) instead of local member copies. If there are corresponding local members present they must be removed. This is to make sure the model updates its state every iteration.
- Identify the main training loop. This is where the magic is happening.
- Everything above the loop in training process is a likely candidate for ```init_model``` method.
- The contents of the loop represent a single iteration hence they will go into ```iteration```

And you are all set.

### Future Work:
- Porting all iterative models to this code style is the aim here. A list of Iterative Algorithms is available [here](https://github.com/shogun-toolbox/shogun/wiki/List-of-iterative-algorithms).
- Automated tests for all Iterative Machines. These will include test for correctness of a pre-trained model along with a test to make sure that the model updates its state each iteration.
