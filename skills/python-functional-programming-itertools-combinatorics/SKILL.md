---
name: python-itertools-combinatorics
description: Instructs the agent on sophisticated iteration using the itertools standard library for grouping, combining, filtering, and safely generating combinatorial products, permutations, and combinations.
---

# Python Itertools and Combinatorics Guidelines

You are an expert Python developer specializing in functional programming and efficient data iteration. When asked to process, combine, filter, or permute sequences of data, you must utilize the `itertools` standard library and adhere to the following rules:

## 1. Grouping and Combining Iterables

Do not build custom stateful loops to combine or group sequences. Use the built-in `itertools` functions:

- **`groupby()`:** Use this to decompose a single iterable into a sequence of iterables over subsets of the input data [3]. **Crucial Rule:** The input to `groupby()` _must_ be sorted by the matching key values prior to grouping, otherwise, it will not group identically keyed items correctly [4].
- **`chain()`:** Use this to combine multiple iterables serially into a single continuous sequence [3, 5].
- **`zip_longest()`:** Use this when merging elements from multiple iterables of different lengths. Unlike the standard `zip()` function which truncates at the shortest list, `zip_longest()` pads the shorter iterables with a specified `fillvalue` (defaulting to `None`) [6, 7].

## 2. Filtering Sequences

Avoid materializing full lists just to extract subsets.

- **`compress()`:** Use this to filter one iterable based on a second, parallel iterable of Boolean values [6]. This is highly efficient for applying a sequence of true/false rules to a data sequence [8].
- **`islice()`:** Use this to apply slice notation (start, stop, step) to an iterable or generator without the overhead of materializing the entire collection in memory [6, 9].

## 3. Generating Combinatorics

When generating complex data arrangements, use the combinatorial functions but always consider computational complexity [10].

- **`product()`:** Use this to compute the Cartesian product of multiple collections [11]. It acts as a highly optimized equivalent to nested `for` statements [12].
- **`combinations()`:** Use this to yield all possible subsets of an original set of length `r` in sorted order, without repeated elements [11, 12]. This is ideal when order does not matter [13].
- **`permutations()`:** Use this to yield tuples of length `r` in all possible orderings [12].
- **Combinatoric Warning:** You must proactively consider the size of the result set. Combinatorial explosion scales extremely poorly (e.g., $20!$ permutations would take roughly 12,000 years to compute) [14]. Never apply `permutations()` or `product()` to large collections without implementing a divide-and-conquer or filtering strategy [15, 16].

---

## Code Examples

Below are best-practice examples demonstrating these `itertools` functions.

**1. Grouping and Combining (`groupby`, `chain`, `zip_longest`)**

```python
from itertools import groupby, chain, zip_longest

# Combining multiple readers/iterators serially
def process_all_files(file_iterators):
    return chain(*file_iterators)

# Grouping data (Ensuring the data is sorted by the key first)
def group_data_by_key(data):
    # groupby REQUIRES sorted input to work properly
    sorted_data = sorted(data, key=lambda x: x['category'])

    for key, group_iter in groupby(sorted_data, key=lambda x: x['category']):
        print(f"Category: {key}")
        print(f"Items: {list(group_iter)}")

# Merging uneven lists
def merge_uneven(list1, list2):
    # Shorter list padded with 'UNKNOWN' instead of truncating
    return list(zip_longest(list1, list2, fillvalue='UNKNOWN'))
2. Filtering Iterables (compress, islice)
from itertools import compress, islice

def get_boolean_subset(data, boolean_selectors):
    # Evaluates lazily, returning items where the selector is True
    return compress(data, boolean_selectors)

def get_paged_results(data_iterator, page_size):
    # Slice a generator without converting the whole generator to a list
    return list(islice(data_iterator, 0, page_size))
3. Combinatorics (product, combinations)
from itertools import product, combinations

def get_cartesian_product(suits, ranks):
    # Replaces a nested loop with a single memory-efficient generator
    return list(product(ranks, suits))

def get_variable_pairs(variables):
    # Generates all unique pairs of variables for correlation testing
    # e.g., combinations([2, 17, 18], 2) -> (1,2), (1,3), (2,3)
    return list(combinations(variables, 2))
```
