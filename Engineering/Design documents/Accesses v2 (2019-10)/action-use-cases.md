

# Actions on streams



## use cases



#### Stream structure

```
    A
   / \
  B	  C
 /  
D	 
```

### Only positive

1. A => all
2. B => B, D
3. X (out of range - as deleted) => none

### With negatives

1. A, !B => A,C
2. A, !D => A, B, C

### Intertwined

1. A, !B, D => A, C, D: 

```
[
	A[C],
	D
]
```

   

## Add tags to this



