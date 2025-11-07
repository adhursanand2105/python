##### 🧠 OOPS (Object-Oriented Programming)

OOP — Object-Oriented Programming — is one of the most widely used and powerful programming paradigms.

---

##### ⚙️ POP (Procedure-Oriented Programming)

In **POP (Procedure-Oriented Programming)**,
the total activity of a program is divided into multiple **functions**,
and we call these functions as per our requirements.

It is **old-style** and suitable only for **small-scale projects**.


---

##### 🧩 OOP (Object-Oriented Programming)

In **OOP**, programming revolves around **real-world entities called Objects**.
Objects represent things that have **properties (data)** and **behavior/method (actions)**.

Python is an **all-rounder language** —
it supports **modular**, **procedure-oriented**, and **object-oriented** programming.

___
***class**, **object** and **reference variable***
___
1. **class**→ acts as a **template**/**blueprint**/**plan**/**model** or **design** for creating objects.
---
#### 🏗️ Class Creation in Python

A **class** is a **blueprint** or **template** used to create objects.
It defines both the **properties (data)** and **behaviors (methods)** of the objects created from it.

**Syntax**:
```python
# --------- Creating a class ---------
class ClassName:
    # class body
    variables
    methods
```
• `class`→ keyword used to define a class.
• `ClassName`→ should follow PascalCase naming (e.g., `Student`, `Employee`).
• **Variables**→ represent data or attributes of the class.
• **Methods**→ represent actions or behavior.

**Example**:
```python
# --------- Creating a class ---------
class Student:
    # class body
    # ----------- method -----------
     def study(self):
         print("The student is studying.")
```
Here:
• `class Student:`→ defines a class named `Student`.
• `def study(self)`→ defines a method (behavior) inside the class.
• `self`→ refers to the object that will call this method.

___
2. **object**→ The **physical existence** of a class.
- Without a class, an object **cannot exist**.
- A class can have multiple objects.
- Properties(data) of an object may be same with another object, but their **addresses** or **id** are always different.
```python

# Different objects with same properties(data)
s=Student("Vivek", 101, 90)         # Object 1
s1=Student("Vivek", 101, 90)       # Object 2

# -------- Displaying object IDs --------
print(id(s1))
print(id(s2))
```

> Output
```
140714033498544
140714033499072
```

___
#### 📦 Object Creation in Python

Once a class is defined, we create an **object** (instance) of it:
```python
# -------- Creating an object --------
s1 = Student()
```

Now `s1` is a **reference variable** that refers to the **object** of the `Student` class.

3. **reference variable**→ The *identifier* (name) refers to the object.
- By using it, we can invoke/call functionality of object.
- An object can have multiple reference variables.

To call the method:
```python

s1.study()
```

> Output
```
The student is studying.
```

**Summary**:

| Term                 | Definition                               |
| -------------------- | ---------------------------------------- |
| `class`              | blueprint that defines data and behavior |
| `object`             | Instance (real form) of a class          |
| `reference variable` | Name that refers to the object           |
| `self` variable      | Refers to the current object             |
