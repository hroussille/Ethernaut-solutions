# 00. Hello Ethernaut

This first challenge is quite simple :

You need to send the password to the authenticate function...

Start with the info method as the challenge page states :

```js
await contract.info();
"You will find what you need in info1()."
```

Calling info1 :

```js
await contract.info1();
"Try info2(), but with \"hello\" as a parameter."
```

Calling info2 with "hello":

```js
await contract.info2("hello");
"The property infoNum holds the number of the next info method to call."
```

Checking the infoNum property :

```js
await contract.infoNum();
42
```

Calling info42:

```js
await contract.info42();
"theMethodName is the name of the next method."
```

Calling theMethodName:

```js
await contract.theMethodName();
"The method name is method7123949."
```

Calling method7123949 :

```js
await contract.method7123949();
"If you know the password, submit it to authenticate()."
```

## Finding the password

Going through the contract ABI we can see a password property...

Checking the password property :

```js
await contract.password();
"ethernaut0"
```

## Solving the challenge

Simply call the authenticate method with the password that we just found :

```js
await contract.authenticate("ethernaut0");
```