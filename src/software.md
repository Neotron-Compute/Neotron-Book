# Introduction

The software in a Neotron system is divided into layers. This approach allows each layer to concentrate on its own areas of responsibility (known as [_separation of concerns_](https://en.wikipedia.org/wiki/Separation_of_concerns)). These layers are arranged into a stack, with the interface to the physical hardware at the bottom, and the user application at the very top.

```
+---------+-------------+
|         |             |
|  Shell  | Application |
|         |             |
+---------+-------------+
|                       |
|    Operating System   |
|                       |
+-----------------------+
|                       |
|         BIOS          |
|                       |
+-----------------------+
|                       |
|     Embedded HAL      |
|                       |
+-----------------------+
|                       |
|Peripheral Access Crate|
|                       |
+-----------------------+
|                       |
|      Raw Hardware     |
|                       |
+-----------------------+
```
