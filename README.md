# Secure bash

Tips for avoiding unexpected surprises in production bash scripts beyond those contained in [1]. 

## Strings in integer-valued variables

As detailed in [2] (section 3.4), when a variable has its integer attribute, any value assigned to it undergoes arithmetic expansion. For example, the following snippet
```
unset A B
declare -i A="B"
``` 
will result in `A` being assigned `0` (zero). However, the integer attribute can be set on an existing variable without any change in the value assigned to it. 

Specifically, consider the following snippet
```
unset A B
declare A="B"
declare -i A
declare -p A
```
will produce 
```
declare -i A="B"
```
Note that the integer attribute is set on but the value of `A` is not an integer but a string and no warning has been generated or conversions carried out. 

Consequently, to be on the safe side, remember that

+ `declare` does not redeclare.
+ Activated attributes are no guarantee of enforced attributes.


## Overwritten indexed arrays

As mentioned in [2], when indexed arrays are assigned, the index of the current element, when not supplied, is the one in the previous assignment plus unity, with indexes starting from zero. For example,
```
declare -a A=(A [1]=B C [3]=D)
```
is equivalent to 
```
declare -a A='([0]="A" [1]="B" [2]="C" [3]="D")'
```
(to test the equivalence, use `declare -p A`).

However, this same feature means that both
```
declare -a A=(A [2]=B [1]=C D)
```
and
```
declare -a A=(A [2]=B [1]=C [2]=D)
```
are equivalent to 
```
declare -a A='([0]="A" [1]="C" [2]="D")
```
That is, in both cases, the value of `A[2]` was first assigned `B` and then silently overwritten with `D`.

Associative arrays require the index to be provided, thus avoiding potential issues due to the index-computation. However, the same index can still be assigned multiple times in the same statement without notice.


## Incomplete traversal of indexed arrays

The length of an indexed array need not be equal to the index of the last element plus unity. For example, 
```declare -a A=([1]=1) && echo ${#A[@]}```
will produce `1` instead of `2`. The same will occur after unsetting some of the array values, e.g.
```declare -a A=(0 1) && unset "A[0]" && echo ${#A[@]}```
Hence, the following construction will not traverse over all set elements of `A`
```
for i in $(seq 1 ${#A[@]}); do 
    # Do something with A[i-1] ...
done
```
but, as detailed in [2], the following construction will certainly do
```
for i in ${!A[@]}; do 
    # Do something with A[i-1] ...
done
```

## References

[1]: https://mywiki.wooledge.org/BashPitfalls
[2]: https://www.gnu.org/software/bash/manual/html_node/Arrays.html
