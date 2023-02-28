# Puzzle 1

```
00      34      CALLVALUE
01      56      JUMP
02      FD      REVERT
03      FD      REVERT
04      FD      REVERT
05      FD      REVERT
06      FD      REVERT
07      FD      REVERT
08      5B      JUMPDEST
09      00      STOP
```



`CALLVALUE` returns `msg.value`.

`JUMP` takes the value on the top of the stack and sets it as **PC**(Program Counter). \
(In this case, PC will be the return value of `CALLVALUE`)

Note that program execution can jump only to `JUMPDEST`.

### Solution

`value: 8`

```
00      34      CALLVALUE
01      56      JUMP         [8]   
02      FD      REVERT
03      FD      REVERT
04      FD      REVERT
05      FD      REVERT
06      FD      REVERT
07      FD      REVERT
08      5B      JUMPDEST     [ ]
09      00      STOP         [ ]
```
