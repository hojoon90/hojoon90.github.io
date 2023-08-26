---
layout: post
title: "Python 조건부 표현식"
tags: [Python]
comments: true
---

JAVA에서는 if else문을 간단하게 표현하기 위해 **'3항연산자'** 라는 것을 사용한다.
Python에서도 이와 유사한 **'조건부 표현식'** 이라는게 있는데, 이는 JAVA와는 조금 다르게 동작한다.
일반적인 JAVA 3항 연산자의 경우,

```python
int a = 800;
int b = 700;

int c = a > b ? a : b;

System.out.println(c);

//출력값 -> 800
```
위와 같이
> {변수} = {조건} ? {참일 경우} : {거짓일 경우}

의 형태로 사용이 된다.
하지만 파이썬의 경우 문법과 조건의 위치가 조금 다른데,

```python
a = 800
b = 700

c = a if a > b else b

print(c)

#출력값 -> 800
```

> {변수} = {참일 경우} if {조건} else {거짓일 경우}

JAVA와 같이 '?' ':' 가 사용되지 않고, if else 문을 이용하여 표현하고 있다.
위의 조건부 표현식을 풀어서 표현하면 아래와 같다.

```python
if {조건}:
    {변수} = {참일 경우}
else:
    {변수} = {거짓일 경우}
```
이렇게 표현하는 이유는 파이썬에서는 대부분 조건값이 참이기 때문에 가독성을 위해 참인 경우를 앞쪽에 놓고 예외적일 경우를 뒤로 두기 때문이라고 한다.