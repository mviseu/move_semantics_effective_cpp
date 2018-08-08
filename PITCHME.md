# Move Semantics #

@color[gray](Scott Meyers: Effective Modern C++)

@color[gray](Maria Viseu)

---

## Content
* @size[0.9em](lvalues and rvalues)
* @size[0.9em](Understand std::move and std::forward)
* @size[0.9em](Distinguish universal references from rvalue references)
* @size[0.9em](Use std::move on rvalue references, std::forward on universal references)
---

## @color[orange](lvalues) and @color[orange](rvalues)
---

### lvalues and rvalues
* Distinguishing expressions that are lvalues from those that are rvalues is the foundation of move semantics
* rvalues indicate objects that are eligible for moving
---

### lvalues and rvalues
@color[grey](Examples)
```cpp

void someFunc(Widget w); // someFunc's parameter passed by value

Widget wid;

someFun(wid); // w is a copy of widget that's created via copy constructor

someFun(Widget()); // w is a copy of widget that's created via move constructor

```
---

### lvalues and rvalues
@size[0.7em](The type of an expression is independent of whether it is an rvalue or lvalue)

```cpp
class Widget {
public:
	Widget(Widget&& rhs); // rhs is an lvalue, though it has an rvalue reference type
	...
};
```
---

@size[1.5em](Understand @color[orange](std::move) and @color[orange](std::forward))
---
@size[1.5em](@color[orange](std::move))
* Replace expensive copy operations with less expensive moves using move constructors, move assignment operators, etc.

* Enable the creation of move-only types (std::unique_ptr)
---

@size[1.5em](@color[orange](std::forward))
* Used with function templates take arbitrary arguments and forward them to other functions such that the target functions receive the same arguments as the forwarding functions

* Preserve r-valueness/l-valueness or non-constness/constness
---

### @color[orange](std::move) and @color[orange](std::forward)

do not...

... move anything

... forward anything

---

### @color[orange](std::move) and @color[orange](std::forward)

they cast...

* std::move unconditionally casts its argument to an rvalue that might be eligible for moving

* std::forward performs a cast to an rvalue only if a particular condition is fullfilled
---

### std::move
@color[grey](Simplified implementation)
```cpp
template <typename T>
typename remove_reference<T>::type&&
move(T&& param) {
	using ReturnType = typename remove_reference<T>::type&&;
	return static_cast<ReturnType>(param);
}

```
---

### std::move
@color[grey](Good example)

```cpp
class Annotation {
public:
	explicit Annotation(std::string text) // passed by value
	: value(std::move(text)); 			// moves text into value
...
private:
	std::string value;
};
```
---

### std::move
@color[grey](Bad example)
```cpp
class Annotation {
public:
	explicit Annotation(const std::string text) // passed by value
	: value(std::move(text)); 				  // this code does not do what it seems to do!
...
private:
	std::string value;
};
```
---

### std::move
@color[grey](Lessons)
* Don't declare objects const if you want to move them	

* std::move casts to rvalue reference but it doesn't:
  * Move @color[orange](text) (this is done by the std::string constructor)
  * Guarantee that @color[orange](text) will be moved
---

### std::forward
@color[grey](Example)
```cpp
void process(const Widget& lvalArg); // process lvalues
void process(Widget&& rvalArg); // process rvalues

template<typename T>
void logAndProcess(T&& param)
{
	makeLogEntry("Calling process");
	process(std::forward<T>(param));
}

// calls to logAndProcess

Widget w;
logAndProcess(w);	// call with lvalue
logAndProcess(std::move(w));    // call with rvalue

```
---

### @color[orange](std::move) and @color[orange](std::forward)
@color[grey](Summary)
* std::move performs unconditional cast to an rvalue

* std::forward casts its argument to an rvalue if that argument is bound to an rvalue
---

@size[1.5em](Distinguish universal references from rvalue references)
---
Which types are rvalue references?
```cpp
void f1(Widget&& param);

Widget&& var1 = Widget();

auto&& var2 = var1;

template<typename T>
void f2(std::vector<T>&& param);

template<typename T>
void f3(T&& param);

template<typename T>
void f4(const T&& param);
```
@[1-1]
@[3-3]
@[5-5]
@[7-8]
@[10-11]
@[13-14]
---
Solutions...
```cpp
void f1(Widget&& param);			// rvalue reference

Widget&& var1 = Widget();		  // rvalue reference

auto&& var2 = var1;				// universal reference

template<typename T>
void f2(std::vector<T>&& param);	// rvalue reference

template<typename T>
void f3(T&& param);				 // universal reference

template<typename T>
void f4(const T&& param);		   // rvalue reference
```
---

### Distinguish @color[orange](universal) references from  @color[orange](rvalue) references
* rvalue references: bind to rvalues, and identify eligible objects for moving

* universal references: bind to everything!
---

### Universal references
* Arise in type deduction contexts:
  * Template parameters
  * Auto declarations

* The form of the type declaration must be precisely T&&
---

### Universal references
@color[grey](Initializers)
```cpp
template<typename T>
void f(T&& param);		// param is a universal reference

Widget w;
f(w);					 // lvalue passed to f; param's type is Widget&

f(std::move(w));		  // rvalue passed to fl param's type is Widget&&

```
---

### Universal references
@color[grey](Summary)
* @size[0.8em](If a function template parameter has type T&& for a deduced type, or if object is declared using auto&&, the parameter or object is a universal reference)

* @size[0.8em](If the form is not precisely type&& or if type deduction does not occur, type&& denotes an rvalue reference)

* @size[0.8em](Universal references are rvalue references if initialized with rvalues, or lvalue references if initialized with lvalues)
---

@size[1.5em](Use std::move on rvalue references, std::forward on universal references)
---

### Use std::move on rvalue references
@color[grey](Example)

```cpp
class Widget {
public:
	Widget(Widget&& rhs)	// rhs definitely refers to an object eligible for moving!	
	: name(std::move(rhs.name)), p(std::move(rhs.p)) {}
...
private:
	std::string name;
	std::shared_ptr<SomeDataStructure> p;
};
```
---

### Use std::forward on universal references
@color[grey](Example)

```cpp
class Widget {
public:
	template<typename T>
	void newName(T&& rhs)	// newName is a universal reference
	{
		name = std::forward<T>(rhs);
	}
...
};
```
---

What's wrong?
```cpp
class Widget {
public:
	template<typename T>
	void newName(T&& rhs)	// newName is a universal reference
	{
		name = std::move(rhs);
	}
...

private:
	std::string name;
	std::shared_ptr<SomeDataStructure> p;
};


std::string getWidgetName();		// factory function

Widget w;

auto n = getWidgetName();
w.setName(n);

// what happens to n?

```
---

### std::move
@color[grey](Good example)
* Function returns by value

* We return an object bound to an rvalue reference

* Use std::move!

```cpp
Matrix											// by value return
operator+(Matrix&& lhs, const Matrix& rhs) {
	lhs += rhs;
	return std::move(lhs);			// move lhs into return value
}

```
---
### std::move
Why is this not good?

```cpp
Widget makeWidget() {
	Widget w;
	...
	return std::move(w); // Noooooo!
}
```
---

### @color[orange](Return value optimization)

@size[0.8em](Compilers may elide the copying or moving of a local object in a function that returns by value if:)

- @size[0.8em](the type of the local object is the same as the type returned by the function and)

- @size[0.8em](the local object is what is being returned)
---

### @color[orange](Return value optimization)

If conditions for RVO are met, but compilers choose not to perform copy elision, then they must treat the return object as an rvalue
---

@size[1.0em](Use std::move on rvalue references, std::forward on universal references)

@size[0.9em](@color[grey](Summary))

* @size[0.8em](Apply std::move to rvalue references and std::forward to universal references)

* @size[0.8em](Do the same thing for rvalue references and universal references that are returned from functions by value)

* @size[0.8em](Don't apply std::move or std::forward to local objects that are eligible for RVO)
---

## @size[1.5em](@color[grey](The End))


