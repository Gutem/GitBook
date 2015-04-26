Questa lista ha la funzione di spiegare e dimostrare alcune delle nuove caratteristiche di Solidity, a mano a mano che vengono completate

## Trattamento di Tipo Speciale per gli Interi letterali

[PT](https://www.pivotaltracker.com/story/show/83393282) Le espressioni che coinvolgono solamente interi letterali sono trattate ora come "grandi interi" (cioe', non fanno overflow) fino a che non vengono usati con un non letterale. Il tipo dell'espressione e' a quel punto determinato come il piu' piccolo tipo di intero che puo' contenere il valore risultante. Ad esempio:

```
contract IntegerLiterals {
  function f() {
    // Questa espressione avrebbe causato un errore in versioni precedenti dato che non era possible sommare int8 a uint8
    // adesso il valore della x e' impostato a 9 ed il tipo a uint8.
    var x = -1 + 10;
    // E' anche possibile utilizzare interi che non combaciano con nessuno dei tipi di Solidity purche'
    // il valore finale e' abbastanza piccolo. Il valore della y sara' 1, e il tipo uint8.
    var y = (0x100000000000000000001 * 0x100000000000000000001 * 0x100000000000000000001) & 0xff;
  }
}
```

## Contratti che ineritano tutti i membri dall'indirizzo

[PT](https://www.pivotaltracker.com/story/show/85006746) I tipi Contract sono implicitamente convertibili in `address` ed esplicitamente convertibili da e a tutti i tipi di intero. Inoltre, un tipo contract contiene tutti i membri dei tipi address con la semantica applicata all'indirizzo del contratto, a meno che siano sovrascritti dal contratto. 

```
contract Helper {
  function getBalance() returns (uint bal) {
    return this.balance; // il balance e' ereditato dal tipo indirizzo
  }
}
contract IsAnAddress {
  Helper helper;
  function setHelper(address a) {
    helper = Helper(a); // Conversione esplicita, simile a un downcast
  }
  function sendAll() {
    // invia tutti i fondi all'aiutante (sia il balancia che send sono membri dell'indirizzo
    helper.send(this.balance);
  }
}
```

## ABI requires arguments to be padded to 32 bytes

[PT](https://www.pivotaltracker.com/story/show/85006670) The latest version of the ABI specification
requires arguments to be padded to multiples of 32 bytes. This is not a language feature that can be demonstrated as code examples. Please see the automated tests `SolidityEndToEndTests::packing_unpacking_types` and `SolidityEndToEndTests::packing_signed_types`.

## Specify value and gas for function calls

[PT](https://www.pivotaltracker.com/story/show/84983014) External functions have member functions "gas" and "value" that allow to change the default amount of gas (all) and wei (0) sent to the called contract. "new expressions" also have the value member.

```
contract Helper {
  function getBalance() returns (uint bal) { return this.balance; }
}
contract Main {
  Helper helper;
  function Main() { helper = new Helper.value(20)(); }
  /// @notice Send `val` Wei to `helper` and return its new balance.
  function sendValue(uint val) returns (uint balanceOfHelper) {
    return helper.getBalance.value(val)();
  }
}
```

## delete for structs

[PT](https://www.pivotaltracker.com/story/show/82574620) `delete` clears all members of a struct.

```
contract Contract {
  struct Data {
    uint deadline;
    uint amount;
  }
  Data data;
  function set(uint id, uint deadline, uint amount) {
    data.deadline = deadline;
    data.amount = amount;
  }
  function clear(uint id) { delete data; }
}
```

Note that, unfortunately, this only works directly on structs for now, so I would propose to not announce "delete" as a feature yet.

## Contract Inheritance

[PT1](https://www.pivotaltracker.com/story/show/84976094) 
[PT2](https://www.pivotaltracker.com/story/show/86666936) Contracts can inherit from each other.

```
contract owned {
    function owned() { owner = msg.sender; }
    address owner;
}

// Use "is" to derive from another contract. Derived contracts can access all members
// including private functions and storage variables.
contract mortal is owned {
    function kill() { if (msg.sender == owner) suicide(owner); }
}

// These are only provided to make the interface known to the compiler.
contract Config { function lookup(uint id) returns (address adr) {} }
contract NameReg { function register(string32 name) {} function unregister() {} }

// Multiple inheritance is possible. Note that "owned" is also a base class of
// "mortal", yet there is only a single instance of "owned" (as for virtual
// inheritance in C++).
contract named is owned, mortal {
    function named(string32 name) {
        address ConfigAddress = 0xd5f9d8d94886e70b06e474c3fb14fd43e2f23970;
        NameReg(Config(ConfigAddress).lookup(1)).register(name);
    }

// Functions can be overridden, both local and message-based function calls take
// these overrides into account.
    function kill() {
        if (msg.sender == owner) {
            address ConfigAddress = 0xd5f9d8d94886e70b06e474c3fb14fd43e2f23970;
            NameReg(Config(ConfigAddress).lookup(1)).unregister();
// It is still possible to call a specific overridden function. 
            mortal.kill();
        }
    }
}

// If a constructor takes an argument, it needs to be provided in the header.
contract PriceFeed is owned, mortal, named("GoldFeed") {
   function updateInfo(uint newInfo) {
      if (msg.sender == owner) info = newInfo;
   }

   function get() constant returns(uint r) { return info; }

   uint info;
}
```

## Function Modifiers

[PT](https://www.pivotaltracker.com/story/show/85007072) Modifiers can be used to easily change the behaviour of functions, for example to automatically check a condition prior to executing the function. They are inheritable properties of contracts and may be overridden by derived contracts.
```
contract owned {
  function owned() { owner = msg.sender; }
  address owner;

  // This contract only defines a modifier but does not use it - it will
  // be used in derived contracts.
  // The function body is inserted where the special symbol "_" in the
  // definition of a modifier appears.
  modifier onlyowner { if (msg.sender == owner) _ }
}
contract mortal is owned {
  // This contract inherits the "onlyowner"-modifier from "owned" and
  // applies it to the "kill"-function, which causes that calls to "kill"
  // only have an effect if they are made by the stored owner.
  function kill() onlyowner {
    suicide(owner);
  }
}
contract priced {
  // Modifiers can receive arguments:
  modifier costs(uint price) { if (msg.value >= price) _ }
}
contract Register is priced, owned {
  mapping (address => bool) registeredAddresses;
  uint price;
  function Register(uint initialPrice) { price = initialPrice; }
  function register() costs(price) {
    registeredAddresses[msg.sender] = true;
  }
  function changePrice(uint _price) onlyowner {
    price = _price;
  }
}
```

Multiple modifiers can be applied to a function by specifying them
in a whitespace-separated list and will be evaluated in order.
Explicit returns from a modifier or function
body immediately leave the whole function, while control flow reaching the
end of a function or modifier body continues after the "_" in the previous
modifier.
Arbitrary expressions are allowed for modifier arguments and in this context,
all symbols visible from the function are visible in the modifier. Symbols
introduced in the modifier are not visible in the function (as they might
change by overriding).


## Conversion between String and Hash types

[PT](https://www.pivotaltracker.com/story/show/85907772) The explicit conversion between `string` and `hash` types of equal size is now allowed. Example:

```
contract Test {
  function convert(hash160 h, string20 s) returns (string20 res_s, hash160 res_h) {
    res_s = string20(h);
    res_h = hash160(s);
  }
}
```

## Access to super

[PT](https://www.pivotaltracker.com/story/show/86688340) In the following contract, the function `kill` is overridden by sibling classes. Due to the fact that the sibling classes do not know of each other, they can only call `mortal.kill()` with the effect that one of the overrides is completely bypassed. A reasonable implementation would call the kill functions in all classes in the inheritance hierarchy. 
```
contract mortal { function kill() { suicide(msg.sender); } }
contract named is mortal { function kill() { /*namereg.unregister();*/ mortal.kill(); } }
contract tokenStorage is mortal { function kill() { /*returnAllTokens();*/ mortal.kill(); } }
contract MyContract is named, tokenStorage {}
```

The `super` keyword solves this. Its type is the type of the current contract if it were empty, i.e. it contains all members of the current class' bases. Access to a member of super invokes the usual virtual member lookup, but it ends just above the current class. Using this keyword, the following works as expected:
```
contract mortal { function kill() { suicide(msg.sender); } }
contract named is mortal { function kill() { /*namereg.unregister();*/ super.kill(); } }
contract tokenStorage is mortal { function kill() { /*returnAllTokens();*/ super.kill(); } }
contract MyContract is named, tokenStorage {}
```
## State Variable Accessors
[PT](https://www.pivotaltracker.com/story/show/86308642) Public state variables now have accessors created for them. Basically any `public` state variable can be accessed by calling a function with the same name as the variable.

```
contract test {
    function test() {
        data = 42;
    }
    uint256 data;
}
```

For example in the above contract if you tried to call test's `data()` method then you would obtain the result 42.

```
contract test {
    function test() {
        data = 42;
    }
private:
    uint256 data;
}
```
On the other hand on the above contract there is no accessor generated since the state variable is private.

## Events

[PT](https://www.pivotaltracker.com/story/show/86896642) Events allow the convenient usage of the EVM logging facilities. Events are inheritable members of contracts. When they are called, they cause
the arguments to be stored in the transaction's log. Up to three parameters can receive the
attribute `indexed` which will cause the respective arguments to be treated as log topics instead
of data. The hash of the signature of the event is always one of the topics. All non-indexed
arguments will be stored in the data part of the log. Example:
```
contract ClientReceipt {
  event Deposit(address indexed _from, hash indexed _id, uint _value);
  function deposit(hash _id) {
    Deposit(msg.sender, _id, msg.value);
  }
}
```
Here, the call to `Deposit` will behave identical to
`log3(msg.value, 0x50cb9fe53daa9737b786ab3646f04d0150dc50ef4e75f59509d83667ad5adb20, sha3(msg.sender), _id);`. Note that the large hex number is equal to the sha3-hash of "Deposit(address,hash256,uint256)", the event's
signature.

## Fallback Functions

[PT](https://www.pivotaltracker.com/story/show/87035858) A contract can have exactly one unnamed
function. This function cannot have arguments and is executed on a call to the contract if
none of the other functions matches the given function identifier (or if no data was supplied at all).

```
contract Test {
  function() { x = 1; }
  uint x;
}

contract Caller {
  function callTest(address testAddress) {
    Test(testAddress).send(0);
    // results in Test(testAddress).x becoming == 1.
  }
}
```

## Events in Exported Interfaces

[PT](https://www.pivotaltracker.com/story/show/87036508) Events are exported to the JSON and Solidity interfaces generated by the compiler. The contract
```
contract c {
    event ev(uint indexed a);
}
```
generates the JSON interface
```
[
   {
      "inputs" : [
         {
            "indexed" : true,
            "name" : "a",
            "type" : "uint256"
         },
         {
            "indexed" : false,
            "name" : "b",
            "type" : "uint256"
         }
      ],
      "name" : "ev",
      "type" : "event"
   }
]
```
and the Solidity interface
`contract c{event ev(uint256 indexed a,uint256 b);}`.

## Visibility Specifiers

[PT](https://www.pivotaltracker.com/story/show/86635568) Functions and storage variables can be specified as being `public`, `protected` or `private`, where the default for functions is `public` `protected` for storage variables. Public functions are part of the external interface and can be called externally, while for storage variables, an automatic accessor function is generated. Non-public functions are only visible inside a contract and its derived contracts (there is no distinction between `protected` and `private` for now).

```
contract c {
  function f(uint a) private returns (uint b) { return a + 1; }
  uint public data;
}
```
External functions can call `c.data()` to retrieve the value of `data` in storage, but are not able to call `f`.

## Numeric Literals with Ether Subdenominations
[PT](https://www.pivotaltracker.com/story/show/84986568) Numeric literals can also be followed by the common ether subdenominations and the value of the assigned to variable will be multiplied by the proper amount.
```
contract c {
  function c()
  {
      val1 = 1 wei;    // 1
      val2 = 1 szabo;  // 1 * 10 ** 12
      val3 = 1 finney; // 1 * 10 ** 15
      val4 = 1 ether;  // 1 * 10 ** 18
 }
  uint256 val1; 
  uint256 val2;
  uint256 val3;
  uint256 val4; 
}
```

## SHA3 with arbitrary arguments
[PT](https://www.pivotaltracker.com/story/show/86896766). `sha3()` can now take an arbitrary number and type of arguments.
```
contract c {
  function c()
  {
      val2 = 123;
      val1 = sha3("foo"); // sha3(0x666f6f)
      val3 = sha3(val2, "bar", 1031); //sha3(0x7b626172407)
  }
  uint256 val1; 
  uint16 val2;
  uint256 val3;
}
```

## Optional Parameter Names
[PT](https://www.pivotaltracker.com/story/show/85594334). The names for function parameters and return parameters are now optional.
```
contract test {
  function func(uint k, uint) returns(uint){
    return k;
  }
}
```

## Generic call Method
[PT](https://www.pivotaltracker.com/story/show/86084248) Address types (and contracts by inheritance) have a method `call` that can receive an arbitrary number of arguments of arbitrary types (which can be serialized in memory) and will invoke a message call on that address while the arguments are ABI-serialized. If the first type has a memory-length of exactly four bytes, it is not padded to 32 bytes, so it is possible to specify a function signature.
```
contract test {
  function f(address addr, uint a) {
    addr.call(string4(string32(sha3("fun(uint256)"))), a);
  }
}
```

## Byte arrays
[PT](https://www.pivotaltracker.com/story/show/87037182) Basic support for variable-length byte arrays. This includes
 - `bytes` type for storage variables
 - `msg.data` is of `bytes` type and contains the calldata
 - functions taking arbitrary parameters (`call`, `sha3`, ...) can be called with `bytes` arguments.
 - copying between `msg.data` and `bytes` storage variables

What is not possible yet:
 - function parameters of `bytes` type
 - local variables of `bytes` type
 - index or slice access

```
contract c {
  bytes data;
  function() { data = msg.data; }
  function forward(address addr) { addr.call(data); }
  function getLength() returns (uint) { return addr.length; }
  function clear() { delete data; }
}
```

## Enums
[PT](https://www.pivotaltracker.com/story/show/86670106) Solidity now supports enums. Enums are explicitly convertible to all integer types but implicit conversion is not allowed.

```
contract test {
	enum ActionChoices { GoLeft, GoRight, GoStraight, SitStill };
	function test()
	{
	    choices = ActionChoices.GoStraight;
	}
	function getChoice() returns (uint d)
	{
	    d = uint256(choices);
	}
	ActionChoices choices;
	}
```

## Visibility Specifiers

[PT](https://www.pivotaltracker.com/story/show/86487946) The visibility of a function can be specified by giving at most one of the specifiers `external`, `public`, `inheritable` or `private`, where `public` is the default. "External" functions can only be called via message-calls, i.e. from other contracts or from the same contract using `this.function()` (note that this also prevents calls to overwritten functions in base classes). Furthermore, parameters of "external" functions are immutable. "Public" functions can be called from other contracts and from the same contract using stack-based calls. "Inheritable" and "private" functions can only be called via stack-based calls, while "inheritable" functions are only visible in the contract itself and its derived contracts and "private" functions are not even visible in derived contracts.

```
contract Base {
  function exte() external { }
  function publ() public /* can be omitted */ { }
  function inhe() inheritable { priv(); }
  function priv() private { }
}
contract Derived is Base {
  function g() {
    this.exte();
    // impossible: exte();
    this.publ();
    publ();
    // impossible: this.inhe();
    inhe();
    // impossible: this.priv();
    // impossible: priv();
  }
}
```

## Import Statement
[PT](https://www.pivotaltracker.com/story/show/87165660) We can now import other contracts and/or standard library contracts using the `import` keyword.
```
import "mortal";

contract Test is mortal {
    // since we import the standard library "mortal" contract and we inherit from it
    // we can call the kill() function that it provides
    function killMe() { kill();}
}
```

## Inline members initialization
[PT](https://www.pivotaltracker.com/story/show/84982976) Inline members can be initialized at declaration time.
```
contract test {
  function test(){
    m_b = 6;
  }
  uint m_a = 5;
  uint m_b;
}
```

## The arguments of the constructor of base contract
[PT](https://www.pivotaltracker.com/story/show/88454388) It is possible to pass arguments to the base contracts constructor. The arguments for the base constructor in the header will be optional later.
```
contract Base {
	function Base(uint i)
	{
		m_i = i;
	}
	uint public m_i;
}
contract Derived is Base(0) {
	function Derived(uint i) Base(i) {}
}
```

## Detect failed CALLs
[PT](https://github.com/ethereum/cpp-ethereum/pull/1212) If a CALL fails, do not just silently continue. Currently, this issues a STOP but it will throw an exception once we have exceptions.
```
contract C {
  function willFail() returns (uint) {
    address(709).call();
    return 1;
  }
}
```
`willFail` will always return an empty byte array (unless someone finds the correct private key...).

## Basic features for arrays
[PT](https://www.pivotaltracker.com/story/show/84119688) Byte arrays and generic arrays of fixed and dynamic size are supported in calldata and storage with the following features: Index access, copying (from calldata to storage, inside storage, both including implicit type conversion), enlarging and shrinking and deleting. Not supported are memory-based arrays (i.e. usage in non-external functions or local variables), array accessors and features like slicing.
Access to an array beyond its length will cause the execution to STOP (exceptions are planned for the future).
```
contract ArrayExample {
  uint[7][] data;
  bytes byteData;
  function assign(uint[4][] input, bytes byteInput) external {
    data = input; // will assign uint[4] to uint[7] correctly, would produce type error if reversed
    byteData = byteInput; // bytes are stored in a compact way
  }
  function indexAccess() {
    data.length += 20;
    data[3][5] = data[3][2];
    byteData[2] = byteData[7]; // this will access sigle bytes
  }
  function clear() {
    delete data[2]; // will clear all seven elements
    data.length = 2; // clears everything after the second element
    delete data; // clears the whole array
  }
}
```

## Now Variable
[PT](https://www.pivotaltracker.com/story/show/89728640) The global scope contains an immutable variable called `now` which is an alias to `block.timestamp`, i.e. it contains the timestamp of the current block.
```
contract TimedContract {
  uint timeout = now + 4 weeks;
}
```

## HashXX and StringXX to bytesXX
[Link to PT] (https://www.pivotaltracker.com/story/show/88146508) 
+ We replace `hash(XX*8)` and `stringXX` by `bytesXX`.
+ `bytesXX` behaves as `hash(XX*8)` in terms of convertability and operators and as `stringXX` in terms of layout in memory (alignment, etc).

+ `byte` is an alias for `bytes1`.

+ `string` is reserved for future use.


## `msg.sig` returns the function's signature hash
[Link to PT] (https://www.pivotaltracker.com/story/show/86896308) 
New magic type `msg.sig` that will provide the hash of the current function signature as `bytes4` type.

```
contract test {
	function foo(uint256 a) returns (bytes4 value) {
		return msg.sig;
	}
}
```

Calling that function will return `2FBEBD38` which is the hash of the signature of `foo(uint256)`.

## Constant variables
[PT](https://www.pivotaltracker.com/story/show/86670364) 
Added `constant` specifier for uint, mapping and bytesXX types. Variables declared with `constant` specifier should be initialized at declaration time and can not be changed later. For now local variables can not be constant. Constant variables are not stored in Storage.

```
contract Foo {
	function getX() returns (uint r) { return x; }
	uint constant x = 56;
}
```

## Anonymous Events
[PT](https://www.pivotaltracker.com/story/show/89518344) 
Added `anonymous` specifier for Event. For the event declared as anonymous the hash of the signature of the event will not be added as a first topic. The format is

```
event <name>([index list]) anonymous;
```
Anonymous property is also visible for ABI document.

## Tightly packed storage

[PT](https://www.pivotaltracker.com/story/show/90011896)
Items in storage are packed tighly as far as possible according to the following rules:

 - The first item in a storage slot is stored lower-order aligned.
 - Elementary types use only that many bytes that are necessary to store them.
 - If an elementary type does not fit the remaining part of a storage slot, it is moved to the next storage slot.
 - Structs and array data always start a new slot and occupy whole slots (but items inside a struct or array are packed tightly according to these rules).

Examples:
```
contract C {
  uint248 x; // 31 bytes: slot 0, offset 0
  uint16 y; // 2 bytes: slot 1, offset 0 (does not fit in slot 0)
  uint240 z; // 30 bytes: slot 1, offset 2 bytes
  uint8 a; // 1 byte: slot 2, offset 0 bytes
  struct S {
    uint8 a; // 1 byte, slot +0, offset 0 bytes
    uint256 b; // 32 bytes, slot +1, offset 0 bytes (does not fit)
  }
  S structData; // 2 slots, slot 3, offset 0 bytes (does not really apply)
  uint8 alpha; // 1 byte, slot 4 (start new slot after struct)
  uint16[3] beta; // 3*16 bytes, slots 5+6 (start new slot for array)
  uint8 gamma; // 1 byte, slot 7 (start new slot after array)
}
```

## Common Subexpression Elimination Excluding Memory and Storage

[PT](https://www.pivotaltracker.com/story/show/89148380)
The optimizer splits code into blocks (at all operations that have non-local side effects like JUMP, CALL, CREATE and for also all instructions that access or modify memory or storage), analyses these blocks by creating an expression graph and establishes equivalences in a bottom-up way, simplifying expressions that e.g. involve constants. In the following code-generation phase, it re-creates the set of instructions that transform a given initial stack configuration into a given target stack configuration utilizing the simplest representatives of these equivalence classes.
In conjunction with the already present jump-optimization, the two code snippets given below should be compiled into the same sequence of instructions:
```
contract test {
  function f(uint x, uint y) returns (uint z) {
    var c = x + 3;
    var b = 7 + (c * (8 - 7)) - x;
    return -(-b | 0);
  }
}
```
```
contract test {
  function f(uint x, uint y) returns (uint z) {
    return 10;
  }
}
```

## Common Subexpression Elimination for Memory and Storage

[PT](https://www.pivotaltracker.com/story/show/90785926)
This adds support for memory and storage operations to the common subexpression eliminator. This makes it possible to e.g. stretch the equality inference engine across SSTORE, MSTORE and even SHA3 computations (which go via memory). Without optimizer (because of packed storage), there are 4 SLOAD, 3 SSTORE and 4 SHA3 operations. The optimizer reduces those to a single SLOAD, SHA3 and SSTORE each.
```
contract test {
  struct s { uint8 a; uint8 b; uint8 c; }
  mapping(uint => s) data;
  function f(uint x, uint8 _a, uint8 _b, uint8 _c) {
    data[x].a = _a;
    data[x].b = _b;
    data[x].c = data[x].a;
  }
}
```
## External Types
[PT](https://www.pivotaltracker.com/story/show/88772706) 
All functions with visibility more than internal should have external types (ABI types) otherwise raise an error.
For Contract type external type is address type.
```
contract Foo {}
contract Test {
    function func() {
        Foo arg;
        this.Poo(arg);
        Poo(arg);
    }
    function Poo(Foo c) external {}
}
```
the ABI interface for Poo is Poo(address) when the Solidity interface is still Poo(Foo).

## Accessor for Arrays
[PT](https://www.pivotaltracker.com/story/show/88500646) 
For Arrays the accessor is generated which accepts the index as parameter and returns an array element
```
contract test {
    uint[3] public data;
    function test() {
        data[0] = 0;
        data[1] = 1;
        data[2] = 2;
    }
}
```
In the above contract if you tried to call the data(1) method of the test you would obtain the result 1.

## Overloading Functions
[PT](https://www.pivotaltracker.com/story/show/85511572) Contracts can have multiple functions of the same name as long as the parameters differ in number or type. If such an overloaded function is referenced, it has to be called immediately to resolve the ambiguity using the types of the arguments. It is an error if not exactly one of the possible functions can be called with the given arguments.

```
contract Base {
  function f(uint a) {}
}
contract Derived is Base {
  function f(uint8 b) {}
  function g() {
    // f(250); would create a type error since 250 can be implicitly
    // converted both to a uint8 and to a uint type
    f(2000); // calls f from Base
  }
}
```
Of course overloading a function does not need inheritance, i.e. `f(uint a)` could as well have been defined directly in `Derived`.

Overloaded functions are also present in the external interface. It is an error if two externally visible functions differ by their Solidity types but not by their external types, e.g. `f(Derived _d)` and `f(address _a)` both end up accepting an `address` type for the ABI although they are considered different inside Solidity.

## Merging of Basic Blocks
[PT](https://www.pivotaltracker.com/story/show/89148124) Blocks of assembly instructions that do not contain jumps, stops or returns are moved and modified according to the following rules:

* if the control never simply flows into a block, but it is jumped to unconditionally, the block is moved, eliminating the jump
* blocks that are never jumped to are removed

These optimizations might sound not very powerful, but together with "Common Subexpression Elimination" (which is does much more than its name might suggest), the following contract is optimized to store `8` in the mapping and return the value without any jump.

```
contract c {
  function () returns (uint) { return g(8); }
  function g(uint pos) internal returns (uint) { setData(pos, 8); return getData(pos); }
  function setData(uint pos, uint value) internal { data[pos] = value; }
  function getData(uint pos) internal { return data[pos]; }
  mapping(uint => uint) data;
}
```

## Contratti di interfaccia

[PT](https://www.pivotaltracker.com/story/show/88344782) Contracts can be marked as "not fully implemented" by containing at least one abstract function. A function is abstract if it does not have a body defined.

```
contract base { function foo(); }
contract derived is base { function foo() {} }
```

For example in the above, foo is an abstract function and as such the base contract is an interface contract. All non-interface contracts that derive from it must implement its abstract functions.
