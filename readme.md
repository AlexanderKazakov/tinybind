# tinybind - the neat and nearly unobstructive xml binding and serialization scheme for C++

## Quick start

Assume we want the following format:
```xml
//xml1
<bakery name='Janes'>
<cake name='New York Cheese'>
	<fruit name='blue berries' quantity='25'>freshness topping</fruit>
</cake>
<cake name='white forest'>
	<fruit name='blue berries' quantity='10' illegal='attribute'></fruit>
	<fruit name='strawberries' quantity='8'></fruit>
</cake>
</bakery>
```
and we want to deserialize and bind to follow:
```C
struct fruit
{
	string name;
	string text;
	int quantity;
};
struct cake
{
	string name;
	vector<fruit> fruits;
};
struct bakery
{
	string name;
	vector<cake> cakes;
	int unbinded;
};
```

### how to do it in tinybind?
define the following schema, heavily sweetened with C macros:
```C
STRUCT(fruit)
{
	ATTR( string, name);
	TEXT( string, text);
	ATTR( int, quantity);
};
STRUCT(cake)
{
	ATTR( string, name);
	CHILD( fruit, fruits);
};
STRUCT(bakery)
{
	ATTR( string, name);
	CHILD( cake, cakes);
};
```
parse by:
```C
TiXmlDocument DOC;
DOC.Parse( xml1);
bakery bake;
TXB_fromxmldoc(&bake, &DOC);
```
then we can access the data by:
```C
cout << "The name of the bakery is " << bake.name << endl;
cout << "There are " << bake.cakes.size() << " cakes" << endl;
cout << "The first one is " << bake.cakes[0].name << ", ";
cout << "which has " << bake.cakes[0].fruits.size() << " topping " << bake.cakes[0].fruits[0].name << "." << endl;
cout << "The second one is " << bake.cakes[1].name << ", ";
cout << "which has " << bake.cakes[1].fruits.size() << " toppings, including " << bake.cakes[1].fruits[0].name << " and " << bake.cakes[1].fruits[1].name << "." << endl;
```
giving:
```
The name of the bakery is Janes
There are 2 cakes
The first one is New York Cheese, which has 1 topping blue berries.
The second one is white forest, which has 2 toppings, including blue berries and strawberries.
```
that's it. neat and simple.

## auto schema generation
if you dont mind the `STRUCT(bakery)` syntax, the struct definitions can be automatically derived from the schema, using C macro tricks:
```C
#include "../tinybind_partA.h"
#include "2_svg.h" //the first pass, defining structs
#include "../tinybind_partB.h"
#include "2_svg.h" //the second pass, defining schema
#include "../tinybind_partC.h"
```

## under the hood
what the 'schema' is doing is actually defining functions:
```C
STRUCT(bakery)                |   bool TXB_binding( bakery* str, TiXmlElement* xmle, bool m)
{                             |   {
	ATTR( string, name);      |        TXB_attr_bind( xmle, m, &str->name, "name");
	CHILD( cake, cakes);      |        TXB_ele_bind<cake>( xmle, m, &str->cakes, "cake");
};                            |   };
```
the overloaded `TXB_binding` functions handles binding of objects.

each `TXB_attr_bind` handles binding for attributes. for example, the integer binder:
```C
void TXB_attr_bind( TiXmlElement* xmle, bool from, int* attr_content, const char* attr_name)
{
	if ( from) //from xml
	{
		if ( xmle->Attribute(attr_name))
			*attr_content = (int)strtol(xmle->Attribute(attr_name),0,0);
	}
	else //to xml
	{
		char buf[100];
		snprintf(buf,100,"%d",*attr_content);
		xmle->SetAttribute( attr_name, buf);
	}
}
```
several built in types are provided, including `int`, `unsigned int`, `float`, `double`, `char`, `unsigned char`, `bool` etc.

that means, if you have pointer members in a struct, it is up to you to implement a binder, overloading `TXB_attr_bind`.

## advanced features
inheritance also works!
```C
STRUCT_INHERIT(A,B)     |     struct A : B
```

## what others do
do you seriously want this?
```C
struct svg_base
{
	const char*	id;
	const char*	style;
};
TiXmlBinding<svg_base> const *
GetTiXmlBinding( svg_base const &, Identity<svg_base> )
{
	static MemberTiXmlBinding<svg_base> binding;
	if( binding.empty() ) {
		binding.AddMember( "id",	MemberAttribute(	&svg_base::id ))	->setFlags(MemberOptional);
		binding.AddMember( "style",	MemberAttribute(	&svg_base::style) )	->setFlags(MemberOptional);
	}
	return &binding;
}
```
Protocal buffer is better, but the binding is still obstructive and you need a separate compiler. (okay you win, performance is more important!)
```C
message Person
{
  required string name = 1;
  required int32 id = 2;
  optional string email = 3;
}
Person person;
person.set_name("John Doe");
person.set_id(1234);
person.set_email("jdoe@example.com");
fstream output("myfile", ios::out | ios::binary);
person.SerializeToOstream(&output);
```

## lessons learned
- using stl `string` and `vector` can make your data structure neat, without the hassle of managing memory manually and keeping a `length` member
- C macros are dirty and powerful
- tinyxml rocks
- tinybind gives you the closest thing to 'native' data binding without a code generator

## Credit
- tinybind
	- Chris Tsang
- TinyXML
	- Lee Thomason, Yves Berquin, Andrew Ellerton 

## License

TinyXML is released under the zlib license:

This software is provided 'as-is', without any express or implied warranty. In no event will the authors be held liable for any damages arising from the use of this software.

Permission is granted to anyone to use this software for any purpose, including commercial applications, and to alter it and redistribute it freely, subject to the following restrictions:

1. The origin of this software must not be misrepresented; you must not claim that you wrote the original software. If you use this software in a product, an acknowledgment in the product documentation would be appreciated but is not required.

2. Altered source versions must be plainly marked as such, and must not be misrepresented as being the original software.

3. This notice may not be removed or altered from any source distribution.
