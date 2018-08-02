# Feature Dispatching Guide

### Overview:
Most algorithms in shogun do not behave in a generic manner in the sense that they are *type dependent*. The ```train``` method can accept any type of features as a ```CFeatures*``` pointer however it is later assumed that the features provided are of a particular type. We intorduced feature dispatching to enable this feature in a more automated way. Some algorithms (like ```CLDA```) already try to take care of types and implement a templated ```train_machine```. We take that idea and give its own space in shogun [here](https://github.com/shogun-toolbox/shogun/tree/develop/src/shogun/machine/FeatureDispatcherCRTP.h).  

### Motivation:
Least Angle Regression is an Iterative Algorithm that tries to stay type independent. This is a good idea in cases where small feature matrix can be scaled down to ```float32_t``` type. Also to implement our Iterative Machine code style here was a problem since it meant having to perform type dispatching in *every iteration*. This is obviously redundant, even if its cheap it is not a good code style. The idea here is to dispatch feature type in base class (```CMachine```) so that when we start training loop, types are already taken care of. 

### Implementation details and Design choice:

```CDenseRealDispatch``` is a class to dispatch dense feature types in ```FeatureDispatchCRTP.h```. It is a mixin class that takes two template arguments. The first argument is the class itself and the second argument is the orignal base class. This is possible due to the concept of Curiously Recursive Template Pattern(```CRTP```). ```C++``` is lazy, this means a pointer for a class is available to use even *before*  it is declared. In other words, a call to a member method of such a class does not need to be instantiated until the function is actually *called*.
Classes(like ```CLDA```, ```CLeastAngleRegression```) which support dynamic dispatching via the mixin will inherit from 
```CDenseRealDispatch<CMockModel, CBaseMachine>``` instead of directly inheriting from ```CBaseMachine```.

##### Methods:
- ```train_dense```: Virtual, the method is written in ```CDenseRealDispatch``` called if the feature class of data pointer is ```C_DENSE```. In the dispatcher this calls ```train_machine_templated``` of model with appropriate type.
```C++
		virtual bool train_dense(CFeatures* data)
		{
			auto this_casted = this->template as<P>();
			switch (data->get_feature_type())
			{
			case F_DREAL:
				return this_casted->template train_machine_templated<float64_t>(
				    data->as<CDenseFeatures<float64_t>>());
			case F_SHORTREAL:
				return this_casted->template train_machine_templated<float32_t>(
				    data->as<CDenseFeatures<float32_t>>());
			case F_LONGREAL:
				return this_casted
				    ->template train_machine_templated<floatmax_t>(
				        data->as<CDenseFeatures<floatmax_t>>());
			default:
				SG_SERROR(
				    "Training with %s of provided type %s is not "
				    "possible!",
				    data->get_name(),
				    feature_type(data->get_feature_type()).c_str());
			}
			return false;
		}
```
- ```train_string```: Virtual, this is similar to ```train_dense``` but it dispatches string types like ```uint8_t```, ```char```.
- ```train_machine_templated```: This is a templated version of ```train_machine``` written in subclass. It is called with appropriate parameter by the dispatcher.

These methods keep feature class check in the base class and perform feature type checks in mixin. This keeps dense and string features seperate.

There is also an added detail that any class that implements feature type dispatching needs to pass features while calling ```train()```. This is something we want to enforce all over shogun and the mixin seemed a good place to start.

### Example and Tests:
A cookbook of how to use a class that supports dispatching is [here]().
The tests for dynamic dispatch use a fake model that returns *true* when a particular feature type is passed. The feature type is provided in constructor.
```C++
class CDenseRealMockMachine
    : public CDenseRealDispatch<CDenseRealMockMachine, CMachine>
{
public:
	CDenseRealMockMachine(EFeatureType f)
	    : CDenseRealDispatch<CDenseRealMockMachine, CMachine>()
	{
		m_expected_feature_type = f;
	}
	~CDenseRealMockMachine()
	{
	}
	template <typename T>
	bool train_machine_templated(CDenseFeatures<T>* data)
	{
		if (data->get_feature_type() == m_expected_feature_type)
			return true;
		return false;
	}
	virtual const char* get_name() const
	{
		return "CDenseRealMockMachine";
	}

	EFeatureType m_expected_feature_type;
};
```
This is then tested with a few feature types for each dispatcher. 
### Applying Dispatchers to more Classes:
To implement dense dispatching in more algorithm we can make the following changes:
- Port the ```train_machine``` call to its templated version ```train_machine_templated```. This can be a bit tricky and involves making the implementation fully templated and making sure the dispatched types are respected.
- Inherit from the mixin instead of directly inheriting from base class.
```diff
- class CMockModel : public CMockMachine
+ class CMockModel : public CDenseRealDispatch<CMockModel, CMockMachine>
```
- Make the class a friend of the Dispatcher. This is done to bring ```train_machine_templated``` into the dispatcher's scope.
```C++
friend class CMockModel : public CDenseRealDispatch<CMockModel, CMockMachine>
```
The idea is similar for ```CStringFeaturesDispatch```.
And you are all set with a fully templated model.

Writing a dispatcher for a new feature class F can be done by:
- Identify what feature types will make sense to dispatch. This is highly dependent on the feature class we pick.
- Add a new method ```train_F``` or some other suitable name to CMachine and update ```CMachine::train()``` with new feature class.
- Add a new dispatcher class to ```FeatureDispatcherCRTP.h``` in a similar way as is already done.
- Implement it in an algorithm or write a unit test.

### Future Work:
- Add more dispatchers with tests along with implementing the dispatchers all over shogun.
- A nice design improvement would be an automated way to create a new dispatcher class or maybe a workaround so that we don't need to have as many dispatchers as we have feature types. 
