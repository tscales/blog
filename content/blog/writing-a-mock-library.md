---
title: "Writing a Mock Library in Go"
date: 2023-08-29T13:34:16-07:00
---

This post will walk through the core components that make the mock package from [Testify](https://github.com/stretchr/testify) work. It consists of 3 parts:

- a `mock` object that keeps track of expected calls
- a `call` object that keeps track of expected arguments and return values
- an algorithm to fetch the return values when called.


`Mock` is a struct with an Array of `Calls` and a mutual exclusion lock to ensure only one goroutine accesses the resource at a time. 

```Go
type Mock struct {
    ExpectedCalls []*Call
    mutex sync.Mutex
}

type Call struct {
    Parent *Mock
    Method string
    Arguments []interface{}
    ReturnArguments []interface{}
}
```

`Mock` needs the ability to add Calls, and Calls needs the ability to add return values. This is done by defining a method `On` that takes a method name and a variable number of arguments. A `Call` is returned, which has its return values set using `Return`. Finally, Mock can be embedded in any struct that implements an interface to make it a mock, and provide access to creating calls. The basic example looks like this

```Go
type Adder interface {
    AddOne(n int) int 
}

type mockAdder struct {
    Mock
}

func (m *mockAdder) AddOne(n int) int {
    //TODO get the return values from the mock
}

mockAdder := &mockAdder
mockAdder.On("AddOne", 1).Return(2)

```

`On` simply defines a new call and appends it to the expected arguments.
`Return`, likewise, sets the `ReturnArguments` of the Call

```Go
func newCall(parent *Mock, methodName string, methodArguments ...interface{}) *Call {
    return &Call{
        Parent: parent,
        Method: methodName,
        Arguments: methodArguments,
        ReturnArguments: make([]interface{},0),
    }
}
func (m *Mock) On(methodName string, arguments ...interface{}) *Call {
    m.mutex.Lock()
    defer m.mutex.Unlock()
    c := NewCall(m, methodName, arguments...)
    m.ExpectedCalls = append(m.ExpectedCalls, c)
}
func (c *Call) Return(returnArguments ...interface{}) *Call {
    c.lock()
    defer c.unlock()
    
    c.ReturnArguments = returnArguments
    return c
}
```

now we need to actually implement the method of the interface.This is where the majority of our code will be written. To get the arguments from the mock, we need to

- Get the name of the method called
- Match the name against the calls that have been defined
- Match the provided arguments against the calls arguments
- if everything matches, returned the call's `ReturnArguments` field.

First, we define a `Called` Method to take in the arguments and get the name of the function that was called

```Go
// get the name of the method that called this function, 
// search for it in the array of calls in this mock
func (m *Mock) Called(arguments ...interface{}) []interface{} {
    pc, _, _, ok := runtime.Caller(1)
    if !ok {
        panic("Couldn't get the caller information")
    }
    functionPath := runtime.FuncForPC(pc).Name()
    parts := strings.Split(functionPath, ".")
    functionName := parts[len(parts)-1]

    return m.MethodCalled(functionName, arguments...)
}
```

now we'll need a function to find the expected call, and when we find the call that matches the method name, a function to match the arguments. 

```Go
func (m *Mock) findExpectedCall(method string, arguments ...interface{}) *Call {
    var expectedCall *Call

    for i, call := range m.ExpectedCalls {
        if call.Method == method {
            _, diffCount := argumentDiff(call.Arguments, arguments)
            if diffCount == 0 {
                expectedCall = call
            }
        }
    }
    return expectedCall
}
```
`argumentDiff` iterates through each argument of the call looking for any differences. With the exception of some special cases, this is mostly done using `reflect.DeepEqual`

```Go
func argumentDiff(callArgs ...interface{}, arguments ...interface{}) int {
    var differences int
    maxArgCount := len(callArgs)
    if len(arguments) > maxArgCount {
        maxArgCount = len(arguments)
    }
    for i := 0; i < maxArgCount; i++ {
        var actual, expected interface{}
        if len(arguments) <= i {
            actual = "(Missing)"
        } else {
            actual = arguments[i]
        }

        if len(callArgs) <= i {
            expected = "(Missing)"
        } else {
            expected = callArgs[i]
        }
        if !objectsAreEqual(actual, expected) {
            differences++
        }     
    }
    return differences
}  
func objectsAreEqual(expected, actual interface{}) bool {
 	switch exp := expected.(type) {
	case []byte:
		act, ok := actual.([]byte)
		if !ok {
			return false
		}
		if exp == nil || act == nil {
			return exp == nil && act == nil
		}
		return bytes.Equal(exp, act)
	default:
		return reflect.DeepEqual(expected, actual)  
}
```

we can now complete the `AddOne` call of our mock by getting the arguments with `Called` and returning them. 

```Go
type Adder interface {
	AddOne(n int) int
}

type mockAdder struct {
	Mock
}

func (m *mockAdder) AddOne(n int) int {
	args := m.Called(n)
	if len(args) < 1 {
		panic("error: expected 1 return argument for mocked function AddOne")
	}
	return args[0].(int)
}

func TestMock(t *testing.T) {
	m := &mockAdder{}
	m.On("AddOne", 1).Return(2)

	res := m.AddOne(1)
	if res != 2 {
		t.Fatalf("expected 2, got %d", res)
	}
}
```

The test passes! from here, you can extend out the `argumentDiff` function to handle more special cases. For Example, if you want a placeholder that always matches, you could export a constant `Anything` 

```Go
const Anything = "mock.Anything"

func ArgumentDiff(callArgs ...interface{}, arguments ...interface{}) int {
    ...
    for i := 0; i < maxArgCount; i++ {
        ...
        if !objectsAreEqual(actual, expected) && !objectsAreEqual(expected, Anything) {
            differences++
        }     
    }

}
```

Now we can accept any value for a mock and get a passing test. This is especially useful for functions that take in a context that don't need exact matches. 

```Go
type Adder interface {
    AddOne(ctx context.Context, n int) int 
}

type mockAdder struct {
    Mock
}

func TestMock(t *testing.T) {
	m := &mockAdder{}
	m.On("AddOne", mock.Anything, 1).Return(2)

	res := m.AddOne(context.Background(), 1)
	if res != 2 {
		t.Fatalf("expected 2, got %d", res)
	}
}
```

The full code is available on [Github](https://github.com/tscales/methodMock)