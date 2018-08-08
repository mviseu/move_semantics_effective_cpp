# Move Semantics

@color[gray](Effective Modern C++: Rvalue references, move semantics and perfect forwarding)

@color[gray](Scott Meyers)

---

## Content
* lvalues and rvalues
* Understand std::move and std::forward
* Distinguish universal references from rvalue references
* Use std::move on rvalue references, std::forward on universal references
* Assume that move operations are not present, not cheap and not used
---

## lvalues and rvalues
---

### lvalues and rvalues
* distinguishing expressions that are rvalues from those that are rvalues is the foundation of move semantics
* rvalues indicate objects that are eligible for moving
---

### lvalues and rvalues
Examples
```cpp

void someFunc(Widget w); // someFunc's parameter passed by value

Widget wid; 			 // wid is some Widget

someFun(wid); 			 // wid is an lvalue
			  			 // w is a copy of widget that's created via copy constructor

someFun(Widget()); 		 // the temporary Widget is an rvalue
						 // w is a copy of widget that's created via move constructor

```
---

### lvalues and rvalues
Expressions vs. Types
* The type of an expression is independent of whether it is an rvalue or lvalue

```cpp
class Widget {
public:
	Widget(Widget&& rhs); // rhs is an lvalue, though it has an rvalue reference type
	...
};
```
---

## Understand std::move and std::forward
---

### Understand std::move and std::forward
General Idea
Move
* Replace expensive copy operations with less expensive moves using move constructors and assignment operators
* Enable the creation of move-only types (std::unique_ptr)
Forward
* Used with function templates take arbitrary arguments and forward them to other functions such that the target functions receive the same arguments as the forwarding functions. 
* Preserve r-valueness/l-valueness or non-constness/constness. 
---

### std::move and std::forward

Do not...

... move anything

... forward anything

---

### std::move and std::forward
Do...

... cast

* std::move unconditionally casts its argument to an rvalue that might be eligible for moving
* std::forward performs a cast to an rvalue only if a particular condition is fullfilled
---

### std::move
Simplified implementation
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
Good Example

```cpp
class Annotation {
public:
	explicit Annotation(std::string text) // passed by value
	: value(std::move(text)); 			  // moves text into value
...
private:
	std::string value;
};
```
---

### std::move
Bad Example
```cpp
class Annotation {
public:
	explicit Annotation(const std::string text) // passed by value
	: value(std::move(text)); 					// this code does not do what it seems to do!
...
private:
	std::string value;
};
```
---

### std::move
Lessons
* Don't declare objects const if you want to move them	
* std::move casts to rvalue reference but it doesn't:
  * Move text (this is done by the std::string constructor)
  * Guarantee that text will be moved
---

### std::forward
Example
```cpp
void process(const Widget& lvalArg);		// process lvalues
void process(Widget&& rvalArg); 			// process rvalues

template<typename T>
void logAndProcess(T&& param)
{
	makeLogEntry("Calling process");
	process(std::forward<T>(param));
}

// calls to logAndProcess

Widget w;
logAndProcess(w);				// call with lvalue
logAndProcess(std::move(w));	// call with rvalue

```
@[140-150]
@[152-154]
---

### std::move and std::forward
Summary
* std::move performs unconditional cast to an rvalue
* std::forward casts its argument to an rvalue if that argument is bound to an rvalue
---

## Distinguish universal references from rvalue references
---

### Distinguish universal references from rvalue references
Examples

Which types are value references?
```cpp
void f(Widget&& param);

Widget&& var1 = Widget();

auto&& var2 = var1;

template<typename T>
void f(std::vector<T>&& param);

template<typename T>
void f(T&& param);

template<typename T>
void f(const T&& param);
```
@[175-175]
@[177-177]
@[179-179]
@[181-182]
@[184-185]
@[187-188]
---

### Distinguish universal references from rvalue references

Examples
```cpp
void f(Widget&& param);			// rvalue reference

Widget&& var1 = Widget();		// rvalue reference

auto&& var2 = var1;				// universal reference

template<typename T>
void f(std::vector<T>&& param);	// rvalue reference

template<typename T>
void f(T&& param);				// universal reference

template<typename T>
void f(const T&& param);		// rvalue reference
```
---

### Distinguish universal references from rvalue references
* rvalue references: bind to rvalues, and identify eligible objects for moving
* universal references: bind to anything
---

### Universal references
* Arise in type deduction contexts:
  * Template parameters
  * Auto declarations
* The form of the type declaration must be precisely T&&
---

### Universal references
Initializers
```cpp
template<typename T>
void f(T&& param);		// param is a universal reference

Widget w;
f(w);					// lvalue passed to f; param's type is Widget&

f(std::move(w));		// rvalue passed to fl param's type is Widget&&

```
---

### Universal references
Summary
* If function template parameter has type T&& for a deduced type, or if object is declared using auto&&, the parameter or object is a universal reference
* If the form is not precisely type&& or if type deduction does not occur, type&& denotes an rvalue reference.
* Universal references are rvalue references if initialized with rvalues, or lvalue references if initialized with lvalues
---

## Use std::move on rvalue references, std::forward on universal references
---

### Use std::move on rvalue references
Example

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
Example

```cpp
class Widget {
public:
	template<typename T>
	void newName(T&& rhs)	// newName is a universal reference
	{
		name = std::forward<T>(newName);
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
		name = std::move(newName);
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
@[289-301]
@[304-311]
---

### std::move
Good example
* Function returns by value
* We return an object bound to an rvalue reference
* Use std::move!

```cpp
Matrix											// by value return
operator+(Matrix&& lhs, const Matrix& rhs) {
	lhs += rhs;
	return std::move(lhs);						// move lhs into return value
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

### Return Value Optimization
"Compilers may elide the copying or moving of a local object in a function that returns by value if:

	1) the type of the local object is the same as the type returned by the function and 

	2) the local object is what is being returned"
---

### Return Value Optimization
"If conditions for RVO are met, but compilers choose not to perform copy elision, then they must treat the return object as an rvalue"

---
### Use std::move on rvalue references, std::forward on universal references
Summary
	- Apply std::move to rvalue references and std::forward to universal references
	- Do the same thing for rvalue references and universal references that are returned from functions by value
	- Don't apply std::move or std::forward to local objects that are eligible for RVO
---

## Assume move operations are not present, not cheap, and not used
---

### Assume move operations are not present, not cheap, and not used
* Move semantics is arguably the premier feature of C++
* But let's keep expectation grounded
---

### Move operations aren't always present
* Standard types were all updated in C++11 to take advantage of move constructors/assignment operators
* But we might be using old libraries...
* Or our own types might not respect the rule of 5 and so move operations might be disabled
---

### Move operations aren't always that cheap
* Most containers store memory on the heap, and hold a pointer to this memory that stores all the elements of the container (e.g. std::vector)
* There are containers that behave differently (e.g. std::array)
---

### Move operations aren't always used
* Strong exception safety guarantee
* Make sure move operations don't throw and mark them noexcept!
---

## Conclusion
* Assume that move operations are not present, not cheap, not used.
* In code with known types & support for move semantics don't assume: use move semantics!
---

## @size[1.5em](@color[grey](The End))


