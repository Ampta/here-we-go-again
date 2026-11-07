# Company-Wise DSA & Stream API Master Solutions Guide
> **Comprehensive Solutions, Scratch Explanations, Dry Runs, Complexity Analysis, and Under-the-Hood JVM Built-In Method Mechanics from the Company-Wise Technical Interview & Architecture Master Guide**

---

## 📑 Master Table of Contents
1. [Part 1: Array & Two-Pointer DSA Problems](#part-1-array--two-pointer-dsa-problems)
   - [Problem 1: Second Largest Number in an Array (C-EDGE)](#problem-1-second-largest-number-in-an-array-c-edge)
   - [Problem 2: Remove Duplicates from an Array/List (Applied Cloud Computing)](#problem-2-remove-duplicates-from-an-arraylist-applied-cloud-computing)
   - [Problem 3: Shift All Zeros to the Right in $O(N)$ (NuSummit & Servosys)](#problem-3-shift-all-zeros-to-the-right-in-on-nusummit--servosys)
   - [Problem 4: Reverse Array In-Place Without Built-ins (Inferyx Hinjewadi)](#problem-4-reverse-array-in-place-without-built-ins-inferyx-hinjewadi)
   - [Problem 5: Even at Even Index and Odd at Odd Index in Sorted Order (Inferyx Hinjewadi)](#problem-5-even-at-even-index-and-odd-at-odd-index-in-sorted-order-inferyx-hinjewadi)
   - [Problem 6: Maximum Sum Subarray of Length > 3 (FinQuest & Team Computer)](#problem-6-maximum-sum-subarray-of-length--3-finquest--team-computer)
   - [Problem 7: Longest Consecutive Increasing Subarray with Indices (Servosys)](#problem-7-longest-consecutive-increasing-subarray-with-indices-servosys)
2. [Part 2: String Algorithms & Text Processing Problems](#part-2-string-algorithms--text-processing-problems)
   - [Problem 8: Palindrome Verification using Stream API (Team Computers)](#problem-8-palindrome-verification-using-stream-api-team-computers)
   - [Problem 9: Anagram Verification using Stream API (Team Computers)](#problem-9-anagram-verification-using-stream-api-team-computers)
   - [Problem 10: String Run-Length Compression (Qtech Software Solution)](#problem-10-string-run-length-compression-qtech-software-solution)
   - [Problem 11: Reverse Words in a String In-Place (In Solutions Global & Servosys)](#problem-11-reverse-words-in-a-string-in-place-in-solutions-global--servosys)
   - [Problem 12: Find the First Duplicate Word in a Sentence (NeoSoft & Cbt Infotech)](#problem-12-find-the-first-duplicate-word-in-a-sentence-neosoft--cbt-infotech)
   - [Problem 13: Convert Integer to Binary String without Predefined Methods (NeoSoft)](#problem-13-convert-integer-to-binary-string-without-predefined-methods-neosoft)
3. [Part 3: Stream API Transformations & Collector Puzzles](#part-3-stream-api-transformations--collector-puzzles)
   - [Problem 14: Count File Extensions into a Frequency Map (Applied Cloud Computing)](#problem-14-count-file-extensions-into-a-frequency-map-applied-cloud-computing)
   - [Problem 15: Nth Highest Price Product from Map (Applied Cloud Computing)](#problem-15-nth-highest-price-product-from-map-applied-cloud-computing)
   - [Problem 16: Sort Map by Values & Find Key from Value (Team Computer)](#problem-16-sort-map-by-values--find-key-from-value-team-computer)
   - [Problem 17: Multi-Level Grouping: Gender & Designation (NeoSoft & Cbt Infotech)](#problem-17-multi-level-grouping-gender--designation-neosoft--cbt-infotech)
   - [Problem 18: Word Frequency Preserving Insertion Order (Bajaj Finserv)](#problem-18-word-frequency-preserving-insertion-order-bajaj-finserv)
   - [Problem 19: Multiply All Elements using Stream `reduce()` (Saint Gobain)](#problem-19-multiply-all-elements-using-stream-reduce-saint-gobain)
   - [Problem 20: Stream Laziness & Intermediate vs Terminal Operations Puzzle](#problem-20-stream-laziness--intermediate-vs-terminal-operations-puzzle)
4. [Part 4: JVM Under-the-Hood Reference for Stream & Array Methods](#part-4-jvm-under-the-hood-reference-for-stream--array-methods)

---

# Part 1: Array & Two-Pointer DSA Problems

---

### Problem 1: Second Largest Number in an Array (C-EDGE)

#### 1. Concept from Scratch
Given an integer array `arr = [12, 35, 1, 10, 34, 1]`, find the second distinct maximum value.
- **The Pitfall**: Simply sorting in descending order and accessing index `1` fails if the array has duplicates at the maximum (e.g., `[35, 35, 10]`, where index `1` is still `35`).
- **Optimal Goal**: Find the second largest in a single linear pass ($O(N)$ time, $O(1)$ auxiliary space) without modifying the array.

#### 2. Code Implementation

```java
import java.util.*;

public class SecondLargestNumber {

    // Approach 1: Imperative Single-Pass DSA - O(N) Time, O(1) Space
    public static int findSecondLargestImperative(int[] nums) {
        if (nums == null || nums.length < 2) {
            throw new IllegalArgumentException("Array must contain at least 2 elements");
        }

        int largest = Integer.MIN_VALUE;
        int secondLargest = Integer.MIN_VALUE;

        for (int num : nums) {
            if (num > largest) {
                // Current largest is displaced to second largest
                secondLargest = largest;
                largest = num;
            } else if (num > secondLargest && num != largest) {
                // Number is between largest and secondLargest
                secondLargest = num;
            }
        }

        if (secondLargest == Integer.MIN_VALUE) {
            throw new NoSuchElementException("No distinct second largest element found (all values equal)");
        }
        return secondLargest;
    }

    // Approach 2: Java Stream API - Functional
    public static OptionalInt findSecondLargestStream(int[] arr) {
        return Arrays.stream(arr)
            .distinct()                           // Step 1: Strip duplicate values
            .boxed()                              // Step 2: IntStream -> Stream<Integer>
            .sorted(Comparator.reverseOrder())    // Step 3: Sort descending
            .skip(1)                              // Step 4: Skip the largest element
            .mapToInt(Integer::intValue)          // Step 5: Convert back to primitive
            .findFirst();                         // Step 6: Terminal operation
    }

    public static void main(String[] args) {
        int[] data = {12, 35, 1, 10, 34, 35, 1};
        System.out.println("Imperative Second Largest: " + findSecondLargestImperative(data)); // 34
        findSecondLargestStream(data).ifPresent(val -> System.out.println("Stream Second Largest: " + val)); // 34
    }
}
```

#### 3. How the Built-in Methods Work Under the Hood
1. **`Arrays.stream(int[] array)`**:
   - Returns a primitive **`IntStream`** backed by an `ArraySpliterator.OfInt`. It operates directly on primitive `int` elements in memory, avoiding boxing overhead (`int` $\to$ `Integer`) during pipeline traversal.
2. **`distinct()`**:
   - For an ordered stream, `distinct()` uses a stateful intermediate stage backed internally by a `LinkedHashSet` (or primitive set). As each element passes through, it checks `.add(element)`. If `.add()` returns `false`, the element was already seen and is discarded.
3. **`boxed()`**:
   - Bridges primitive `IntStream` to an object stream `Stream<Integer>` by invoking `Integer.valueOf(i)` on each element. This is required because `Comparator.reverseOrder()` expects reference objects implementing `Comparable<T>`, which primitive `int` cannot satisfy.
4. **`sorted(Comparator.reverseOrder())`**:
   - Buffers all elements into an internal array and runs **TimSort** (an adaptive, stable sorting algorithm derived from MergeSort and InsertionSort) with time complexity $O(N \log N)$.
5. **`skip(1)`**:
   - An intermediate operation that maintains an internal counter. It discards the first $n$ elements and allows subsequent elements through.
6. **`findFirst()`**:
   - A short-circuiting terminal operation that returns an `Optional<T>` containing the first element of the remaining stream, immediately terminating pipeline evaluation.

---

### Problem 2: Remove Duplicates from an Array/List (Applied Cloud Computing)

#### 1. Concept from Scratch
- **Scenario A (Unsorted List)**: Remove all duplicate occurrences while preserving original order.
- **Scenario B (Sorted Array - LeetCode 26)**: In-place removal using two pointers without allocating a new array ($O(1)$ extra space), returning the count of unique elements.

#### 2. Code Implementation

```java
import java.util.*;

public class RemoveDuplicates {

    // Scenario A: Stream distinct (Unsorted List)
    public static List<Integer> removeDuplicatesStream(List<Integer> list) {
        return list.stream()
            .distinct()
            .toList(); // Java 16+ unmodifiable list
    }

    // Scenario B: In-place Two-Pointer DSA on Sorted Array - O(N) Time, O(1) Space
    public static int removeDuplicatesFromSortedArray(int[] nums) {
        if (nums == null || nums.length == 0) return 0;

        int writeIndex = 1; // Slow pointer: points to next unique write position

        for (int readIndex = 1; readIndex < nums.length; readIndex++) { // Fast pointer
            if (nums[readIndex] != nums[readIndex - 1]) {
                nums[writeIndex] = nums[readIndex];
                writeIndex++;
            }
        }
        return writeIndex; // Number of unique elements
    }

    public static void main(String[] args) {
        List<Integer> list = List.of(1, 2, 3, 2, 4, 1, 5);
        System.out.println("Unique List: " + removeDuplicatesStream(list)); // [1, 2, 3, 4, 5]

        int[] sortedArr = {1, 1, 2, 2, 3, 4, 4, 5};
        int uniqueCount = removeDuplicatesFromSortedArray(sortedArr);
        System.out.println("Unique Count: " + uniqueCount); // 5
        System.out.println("In-place Array: " + Arrays.toString(Arrays.copyOf(sortedArr, uniqueCount))); // [1, 2, 3, 4, 5]
    }
}
```

#### 3. How the Built-in Methods Work Under the Hood
1. **`list.stream()`**:
   - Invokes `Collection.stream()`, returning a `ReferencePipeline.Head` initialized with `list.spliterator()`.
2. **`toList()` (Java 16+)**:
   - Unlike `collect(Collectors.toList())` which returns a mutable `ArrayList`, `Stream.toList()` directly allocates a compact internal unmodifiable array list (`java.util.ImmutableCollections.ListN`), minimizing heap allocation and avoiding extra builder object wrappers.

---

### Problem 3: Shift All Zeros to the Right in $O(N)$ (NuSummit & Servosys)

#### 1. Concept from Scratch
- **Input**: `[1, 0, 5, 7, 0, 6, 4, 0, 2, 0, 8, 0]`
- **Output**: `[1, 5, 7, 6, 4, 2, 8, 0, 0, 0, 0, 0]`
- **Logic**: Use a write pointer (`nonZeroIndex`). Traverse the array: whenever a non-zero number is encountered, write it to `nums[nonZeroIndex]` and increment `nonZeroIndex`. After the loop completes, fill all remaining indices up to `nums.length - 1` with `0`.

#### 2. Code Implementation

```java
import java.util.Arrays;

public class ShiftZerosRight {

    // Imperative Two-Pointer: O(N) Time, O(1) Extra Space
    public static void shiftZerosInPlace(int[] nums) {
        int nonZeroIndex = 0;

        // Step 1: Shift all non-zero elements to the front
        for (int i = 0; i < nums.length; i++) {
            if (nums[i] != 0) {
                nums[nonZeroIndex] = nums[i];
                nonZeroIndex++;
            }
        }

        // Step 2: Fill the rest of the array with zeros
        while (nonZeroIndex < nums.length) {
            nums[nonZeroIndex] = 0;
            nonZeroIndex++;
        }
    }

    // Java Stream Approach: Functional Filter & Concat
    public static int[] shiftZerosStream(int[] nums) {
        int[] nonZeros = Arrays.stream(nums).filter(n -> n != 0).toArray();
        int[] result = new int[nums.length];

        // Copy non-zeros into front; remainder is default 0 in Java
        System.arraycopy(nonZeros, 0, result, 0, nonZeros.length);
        return result;
    }

    public static void main(String[] args) {
        int[] arr = {1, 0, 5, 7, 0, 6, 4, 0, 2, 0, 8, 0};
        shiftZerosInPlace(arr);
        System.out.println("Shifted Zeros: " + Arrays.toString(arr));
        // Output: [1, 5, 7, 6, 4, 2, 8, 0, 0, 0, 0, 0]
    }
}
```

#### 3. How the Built-in Methods Work Under the Hood
1. **`System.arraycopy(src, srcPos, dest, destPos, length)`**:
   - A native JVM C++ method (`JVM_ArrayCopy`) that performs raw memory block copies (`memmove`). It is significantly faster than any manual Java `for` loop because it leverages CPU vector instructions (AVX/SSE) to copy multiple bytes per cycle.
2. **`filter(IntPredicate predicate)`**:
   - An intermediate operation. Under the hood, it constructs a `StatelessOp` pipeline stage. When elements are pushed downstream via `accept(int value)`, it evaluates `predicate.test(value)`. If `false`, downstream invocation is skipped.

---

### Problem 4: Reverse Array In-Place Without Built-ins (Inferyx Hinjewadi)

#### 1. Concept from Scratch
- **Constraint**: Cannot use `Collections.reverse()`, extra arrays, or helper data structures.
- **Algorithm**: Classic two-pointer opposite-end swap. Initialize `left = 0` and `right = length - 1`. While `left < right`, swap `arr[left]` with `arr[right]`, then increment `left` and decrement `right`.

#### 2. Code Implementation

```java
import java.util.Arrays;

public class InPlaceArrayReverse {

    public static void reverse(int[] arr) {
        if (arr == null || arr.length <= 1) return;

        int left = 0;
        int right = arr.length - 1;

        while (left < right) {
            // Swap using temp variable
            int temp = arr[left];
            arr[left] = arr[right];
            arr[right] = temp;

            left++;
            right--;
        }
    }

    public static void main(String[] args) {
        int[] arr = {10, 20, 30, 40, 50};
        reverse(arr);
        System.out.println("Reversed: " + Arrays.toString(arr)); // [50, 40, 30, 20, 10]
    }
}
```

#### 3. Complexity & Mechanics
- **Time Complexity**: $O(N/2) = O(N)$ — Exactly $N/2$ swaps.
- **Space Complexity**: $O(1)$ — Only a single primitive integer `temp` allocated on the thread's stack.

---

### Problem 5: Even at Even Index and Odd at Odd Index in Sorted Order (Inferyx Hinjewadi)

#### 1. Concept from Scratch
- **Problem**: Input `[5, 10, 9, 14, 7, 12]`. Output must place sorted even numbers at even indices (0, 2, 4...) and sorted odd numbers at odd indices (1, 3, 5...).
- **Result**: `[10, 5, 12, 7, 14, 9]`
- **Constraint**: Implement without using `Arrays.sort()` or `Collections.sort()`.

#### 2. Code Implementation

```java
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;

public class EvenOddArrangement {

    public static void arrange(int[] arr) {
        List<Integer> evens = new ArrayList<>();
        List<Integer> odds = new ArrayList<>();

        // Step 1: Partition into even and odd buckets
        for (int num : arr) {
            if (num % 2 == 0) evens.add(num);
            else odds.add(num);
        }

        // Step 2: Custom In-Place Sort without built-in sort (Insertion Sort)
        customSort(evens);
        customSort(odds);

        // Step 3: Reconstruct into target array
        int evenIdx = 0, oddIdx = 0;
        for (int i = 0; i < arr.length; i++) {
            if (i % 2 == 0) {
                arr[i] = evens.get(evenIdx++);
            } else {
                arr[i] = odds.get(oddIdx++);
            }
        }
    }

    private static void customSort(List<Integer> list) {
        for (int i = 0; i < list.size() - 1; i++) {
            for (int j = 0; j < list.size() - i - 1; j++) {
                if (list.get(j) > list.get(j + 1)) {
                    int temp = list.get(j);
                    list.set(j, list.get(j + 1));
                    list.set(j + 1, temp);
                }
            }
        }
    }

    public static void main(String[] args) {
        int[] arr = {5, 10, 9, 14, 7, 12};
        arrange(arr);
        System.out.println("Arranged: " + Arrays.toString(arr)); // [10, 5, 12, 7, 14, 9]
    }
}
```

---

### Problem 6: Maximum Sum Subarray of Length > 3 (FinQuest & Team Computer)

#### 1. Concept from Scratch
- **Problem**: Find the maximum contiguous subarray sum with the restriction that the subarray length must be **strictly greater than 3** (i.e., length $\ge 4$).
- **Algorithm (Optimized Sliding Window + Kadane's Extension)**:
  1. Calculate the initial sum of the first 4 elements ($K = 4$).
  2. Slide the window of size $K$ across the array.
  3. Maintain a running tracker of maximum sums for window sizes expanding beyond 4.

#### 2. Code Implementation

```java
public class MaxSubarrayLengthGreaterThanThree {

    public static int maxSubarraySumMinLen4(int[] nums) {
        int k = 4;
        if (nums == null || nums.length < k) {
            throw new IllegalArgumentException("Array length must be at least 4");
        }

        // Calculate sum of initial window of size 4
        int windowSum = 0;
        for (int i = 0; i < k; i++) {
            windowSum += nums[i];
        }

        int maxSum = windowSum;
        int currentSum = windowSum;

        // Slide window from index k to end
        for (int i = k; i < nums.length; i++) {
            // Option A: Extend previous larger subarray (+ nums[i])
            currentSum += nums[i];
            
            // Option B: Start fresh minimum window of size 4
            windowSum += nums[i] - nums[i - k];
            currentSum = Math.max(currentSum, windowSum);
            maxSum = Math.max(maxSum, currentSum);
        }

        return maxSum;
    }

    public static void main(String[] args) {
        int[] arr = {1, -2, 3, 4, 5, -1, 2, -3};
        System.out.println("Max Sum (Len >= 4): " + maxSubarraySumMinLen4(arr)); // 13 (Subarray: [3, 4, 5, -1, 2])
    }
}
```

---

### Problem 7: Longest Consecutive Increasing Subarray with Indices (Servosys)

#### 1. Concept from Scratch
- **Input**: `arr = [1, 5, 6, 7, 9, 10, 11, 12, 13]`
- **Output**: Subarray `[9, 10, 11, 12, 13]` (Length 5, starting index 4).
- **Rule**: Consecutive integers strictly increasing by 1 ($arr[i] == arr[i-1] + 1$).

#### 2. Code Implementation

```java
import java.util.Arrays;

public class LongestConsecutiveSubarray {

    public static int[] findLongestConsecutiveIncreasing(int[] arr) {
        if (arr == null || arr.length == 0) return new int[0];

        int maxLen = 1;
        int currentLen = 1;
        int maxStartIndex = 0;
        int currentStartIndex = 0;

        for (int i = 1; i < arr.length; i++) {
            if (arr[i] == arr[i - 1] + 1) {
                currentLen++;
            } else {
                if (currentLen > maxLen) {
                    maxLen = currentLen;
                    maxStartIndex = currentStartIndex;
                }
                currentLen = 1;
                currentStartIndex = i;
            }
        }

        // Final check for sequence ending at last element
        if (currentLen > maxLen) {
            maxLen = currentLen;
            maxStartIndex = currentStartIndex;
        }

        System.out.println("Length: " + maxLen + ", Start Index: " + maxStartIndex);
        return Arrays.copyOfRange(arr, maxStartIndex, maxStartIndex + maxLen);
    }

    public static void main(String[] args) {
        int[] arr = {1, 5, 6, 7, 9, 10, 11, 12, 13};
        int[] result = findLongestConsecutiveIncreasing(arr);
        System.out.println("Longest Subarray: " + Arrays.toString(result)); // [9, 10, 11, 12, 13]
    }
}
```

#### 3. How `Arrays.copyOfRange()` Works Under the Hood
- `Arrays.copyOfRange(arr, from, to)` validates array boundaries and allocates a new array of type `arr.getClass().getComponentType()` with length `to - from`.
- It then executes native `System.arraycopy()` to copy memory bytes in a single call.

---

# Part 2: String Algorithms & Text Processing Problems

---

### Problem 8: Palindrome Verification using Stream API (Team Computers)

#### 1. Concept from Scratch
A string is a palindrome if it reads the same forward and backward after removing all non-alphanumeric characters and ignoring casing (e.g., `"A man, a plan, a canal: Panama"`).
- We can compare the character at index `i` with the character at index `length - 1 - i` for the first half of the string ($0 \le i < \text{length}/2$).

#### 2. Code Implementation

```java
import java.util.stream.IntStream;

public class StreamPalindrome {

    public static boolean isPalindrome(String str) {
        if (str == null) return false;

        // Clean string: alphanumeric only, lowercase
        String clean = str.replaceAll("[^a-zA-Z0-9]", "").toLowerCase();

        // Stream over index range 0 to clean.length()/2
        return IntStream.range(0, clean.length() / 2)
            .allMatch(i -> clean.charAt(i) == clean.charAt(clean.length() - i - 1));
    }

    public static void main(String[] args) {
        System.out.println("Is 'racecar' palindrome? " + isPalindrome("racecar")); // true
        System.out.println("Is 'A man, a plan...' palindrome? " + isPalindrome("A man, a plan, a canal: Panama")); // true
        System.out.println("Is 'hello' palindrome? " + isPalindrome("hello")); // false
    }
}
```

#### 3. How the Built-in Methods Work Under the Hood
1. **`str.replaceAll(regex, replacement)`**:
   - Compiles the regex into a `java.util.regex.Pattern` and runs a `Matcher` across the string's backing `byte[]` / `char[]` buffer.
2. **`IntStream.range(0, clean.length() / 2)`**:
   - Generates a sequential primitive `IntStream` from `startInclusive` (0) to `endExclusive` (`length/2`), incrementing by 1 on each step without allocating integer objects.
3. **`allMatch(IntPredicate predicate)`**:
   - A **short-circuiting terminal operation**. It tests each integer with the predicate. The moment any comparison returns `false`, it aborts evaluation immediately and returns `false`, avoiding redundant checks.

---

### Problem 9: Anagram Verification using Stream API (Team Computers)

#### 1. Concept from Scratch
Two strings are **Anagrams** if they contain the exact same characters with identical frequencies, regardless of order (e.g., `"listen"` and `"silent"`).
- Using streams, we build a character frequency map (`Map<Character, Long>`) for both strings and verify that the maps are equal.

#### 2. Code Implementation

```java
import java.util.Map;
import java.util.function.Function;
import java.util.stream.Collectors;

public class StreamAnagram {

    public static boolean isAnagram(String s1, String s2) {
        if (s1 == null || s2 == null || s1.length() != s2.length()) {
            return false;
        }

        Map<Character, Long> freq1 = getCharFrequencyMap(s1);
        Map<Character, Long> freq2 = getCharFrequencyMap(s2);

        // Map.equals() verifies all keys exist with identical values
        return freq1.equals(freq2);
    }

    private static Map<Character, Long> getCharFrequencyMap(String s) {
        return s.toLowerCase().chars()
            .mapToObj(c -> (char) c)
            .collect(Collectors.groupingBy(
                Function.identity(),
                Collectors.counting()
            ));
    }

    public static void main(String[] args) {
        System.out.println("Is 'listen' & 'silent' anagram? " + isAnagram("listen", "silent")); // true
        System.out.println("Is 'apple' & 'pale' anagram? " + isAnagram("apple", "pale")); // false
    }
}
```

#### 3. How the Built-in Methods Work Under the Hood
1. **`s.chars()`**:
   - Returns an `IntStream` containing the UTF-16 code points of the characters.
2. **`mapToObj(c -> (char) c)`**:
   - Casts each primitive `int` code point to a `Character` object, producing a `Stream<Character>`.
3. **`Function.identity()`**:
   - A static utility method returning `t -> t`, returning the character itself as the grouping key.
4. **`Collectors.groupingBy(classifier, downstream)`**:
   - Uses an internal `HashMap<K, A>`. For each character, it invokes `map.computeIfAbsent(key, k -> downstream.supplier().get())` and feeds the element into `downstream.accumulator()`.
5. **`Collectors.counting()`**:
   - Internally defined as `reducing(0L, e -> 1L, Long::sum)`, accumulating counts as a `Long`.
6. **`freq1.equals(freq2)`**:
   - `AbstractMap.equals()` verifies that both maps have the same size and that for every key in `freq1`, `freq2.get(key).equals(freq1.get(key))` evaluates to `true`.

---

### Problem 10: String Run-Length Compression (Qtech Software Solution)

#### 1. Concept from Scratch
- **Problem**: Compress repeated characters in a string: `"aabcccccaaa"` $\to$ `"a2b1c5a3"`.
- **Logic**: Traverse the string while maintaining a run-length counter. When the character at `i` differs from `i + 1`, append the character and its count to a `StringBuilder` and reset the counter to 1.

#### 2. Code Implementation

```java
public class StringRunLengthEncoder {

    public static String compress(String s) {
        if (s == null || s.isEmpty()) return s;

        StringBuilder sb = new StringBuilder();
        int count = 1;

        for (int i = 0; i < s.length(); i++) {
            // Check if next character is the same
            if (i + 1 < s.length() && s.charAt(i) == s.charAt(i + 1)) {
                count++;
            } else {
                // Character run ended
                sb.append(s.charAt(i)).append(count);
                count = 1; // Reset for next character
            }
        }
        return sb.toString();
    }

    public static void main(String[] args) {
        System.out.println("Compressed: " + compress("aabcccccaaa")); // a2b1c5a3
        System.out.println("Compressed: " + compress("aabbccdd"));   // a2b2c2d2
    }
}
```

#### 3. How `StringBuilder.append()` Works Under the Hood
- `StringBuilder` wraps a mutable `byte[] value` (in Java 9+ Compact Strings).
- `append()` checks if the new length exceeds `value.length`. If so, it calls `newCapacity()` to double the capacity ($2 \times \text{old} + 2$) and uses `System.arraycopy()` to expand the buffer, achieving amortized $O(1)$ append time.

---

### Problem 11: Reverse Words in a String In-Place (In Solutions Global & Servosys)

#### 1. Concept from Scratch
- **Input**: `"hello world"`
- **Expected Output**: `"olleh dlrow"` (Order of words is preserved; characters inside each word are reversed).

#### 2. Code Implementation

```java
import java.util.Arrays;
import java.util.stream.Collectors;

public class WordReversal {

    // Approach 1: Java Streams + StringBuilder
    public static String reverseWordsStream(String str) {
        if (str == null) return null;
        return Arrays.stream(str.split(" "))
            .map(word -> new StringBuilder(word).reverse().toString())
            .collect(Collectors.joining(" "));
    }

    // Approach 2: Pure In-Place Two-Pointer on char[]
    public static String reverseWordsInPlace(String str) {
        if (str == null) return null;
        char[] chars = str.toCharArray();
        int start = 0;

        for (int end = 0; end <= chars.length; end++) {
            // Word boundary encountered (space or end of string)
            if (end == chars.length || chars[end] == ' ') {
                reverseRange(chars, start, end - 1);
                start = end + 1;
            }
        }
        return new String(chars);
    }

    private static void reverseRange(char[] arr, int left, int right) {
        while (left < right) {
            char temp = arr[left];
            arr[left] = arr[right];
            arr[right] = temp;
            left++;
            right--;
        }
    }

    public static void main(String[] args) {
        System.out.println("Stream Result: " + reverseWordsStream("hello world")); // olleh dlrow
        System.out.println("In-Place Result: " + reverseWordsInPlace("Java Spring Boot")); // avaJ gnirpS tooB
    }
}
```

#### 3. How `Collectors.joining(" ")` Works Under the Hood
- Uses `java.util.StringJoiner`.
- As stream items arrive, it appends the delimiter (`" "`) and the element into an internal `char[]`/`byte[]` buffer, returning the final concatenated string via `toString()` without intermediate allocations.

---

### Problem 12: Find the First Duplicate Word in a Sentence (NeoSoft & Cbt Infotech)

#### 1. Concept from Scratch
- **Problem**: Given a sentence `"Spring Boot microservices architecture uses Spring Cloud"`, find the first word that appears more than once (`"Spring"`).
- **Trick**: Use a `HashSet`. The method `Set.add(element)` returns **`false`** if the element is already present in the set!

#### 2. Code Implementation

```java
import java.util.HashSet;
import java.util.Optional;
import java.util.Set;
import java.util.Arrays;

public class FirstDuplicateWord {

    // Approach 1: Imperative Loop with Set.add()
    public static String findFirstDuplicate(String sentence) {
        if (sentence == null || sentence.isBlank()) return null;

        String[] words = sentence.toLowerCase().split("\\s+");
        Set<String> seen = new HashSet<>();

        for (String word : words) {
            if (!seen.add(word)) { // Returns false if word already exists!
                return word;
            }
        }
        return null; // No duplicate found
    }

    // Approach 2: Stream API Filter
    public static Optional<String> findFirstDuplicateStream(String sentence) {
        Set<String> seen = new HashSet<>();
        return Arrays.stream(sentence.toLowerCase().split("\\s+"))
            .filter(word -> !seen.add(word))
            .findFirst();
    }

    public static void main(String[] args) {
        String input = "Spring Boot microservices architecture uses Spring Cloud";
        System.out.println("First Duplicate (Loop): " + findFirstDuplicate(input)); // spring
        findFirstDuplicateStream(input).ifPresent(w -> System.out.println("First Duplicate (Stream): " + w)); // spring
    }
}
```

#### 3. How `HashSet.add()` Works Under the Hood
- `HashSet` is backed internally by a `HashMap<E, Object>`.
- When `seen.add(word)` is called, it executes `map.put(word, PRESENT) == null`.
- If the key already existed, `map.put()` returns the previous value (`PRESENT`), making `== null` evaluate to **`false`**.

---

### Problem 13: Convert Integer to Binary String without Predefined Methods (NeoSoft)

#### 1. Concept from Scratch
- **Constraint**: Cannot use `Integer.toBinaryString(n)`.
- **Algorithm**:
  - Repeatedly extract the least significant bit using bitwise AND (`n & 1`) or modulo (`n % 2`).
  - Shift `n` right by 1 (`n >>> 1`).
  - Prepend bits to a `StringBuilder`.

#### 2. Code Implementation

```java
public class IntToBinaryConverter {

    public static String toBinary(int number) {
        if (number == 0) return "0";

        StringBuilder binary = new StringBuilder();
        int n = number;

        // Support for positive integers
        while (n > 0) {
            int remainder = n % 2;
            binary.append(remainder);
            n = n / 2;
        }

        // Reverse because remainders were collected in reverse order
        return binary.reverse().toString();
    }

    // Bitwise version supporting full 32-bit representations
    public static String toBinaryBitwise(int n) {
        if (n == 0) return "0";
        StringBuilder sb = new StringBuilder();

        for (int i = 31; i >= 0; i--) {
            int bit = (n >> i) & 1;
            sb.append(bit);
        }
        // Strip leading zeros
        String result = sb.toString().replaceFirst("^0+(?!$)", "");
        return result;
    }

    public static void main(String[] args) {
        System.out.println("10 to Binary: " + toBinary(10)); // 1010
        System.out.println("255 to Binary (Bitwise): " + toBinaryBitwise(255)); // 11111111
    }
}
```

---

# Part 3: Stream API Transformations & Collector Puzzles

---

### Problem 14: Count File Extensions into a Frequency Map (Applied Cloud Computing)

#### 1. Concept from Scratch
- **Problem**: Given a list of filenames: `["a.txt", "b.pdf", "c.txt", "d.pdf", "e.pdf", "f.docx"]`, produce a `Map<String, Long>` containing:
  `{ "txt": 2, "pdf": 3, "docx": 1 }`.
- **Logic**: Extract the substring after the last dot `.` and group by that extension with `Collectors.counting()`.

#### 2. Code Implementation

```java
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

public class FileExtensionCounter {

    public static Map<String, Long> countExtensions(List<String> fileNames) {
        return fileNames.stream()
            .filter(name -> name != null && name.contains("."))
            .map(name -> name.substring(name.lastIndexOf(".") + 1))
            .collect(Collectors.groupingBy(
                extension -> extension.toLowerCase(),
                Collectors.counting()
            ));
    }

    public static void main(String[] args) {
        List<String> files = List.of("a.txt", "b.pdf", "c.txt", "d.pdf", "e.pdf", "f.docx");
        System.out.println("File Extension Map: " + countExtensions(files));
        // Output: {docx=1, pdf=3, txt=2}
    }
}
```

#### 3. How the Built-in Methods Work Under the Hood
1. **`lastIndexOf(".")`**:
   - Scans the string's backing byte array backwards starting from index `length - 1` until it matches `.` (ASCII 46).
2. **`substring(index)`**:
   - Creates a new `String` object wrapping the sub-range of the byte array.
3. **`Collectors.groupingBy(classifier, counting())`**:
   - Collects extensions into an internal map, executing `count++` for each key.

---

### Problem 15: Nth Highest Price Product from Map (Applied Cloud Computing)

#### 1. Concept from Scratch
- **Input**: A `Map<String, String>` where keys are product names and values are price strings (e.g., `{"Laptop": "1200", "Phone": "800", "TV": "1500", "Mouse": "50"}`).
- **Goal**: Find the $N$-th highest priced product (e.g., if $N=2$, return `Map.Entry("Laptop", 1200.0)`).

#### 2. Code Implementation

```java
import java.util.*;

public class NthHighestProduct {

    public static Optional<Map.Entry<String, Double>> findNthHighest(Map<String, String> map, int n) {
        if (map == null || n <= 0 || n > map.size()) {
            return Optional.empty();
        }

        return map.entrySet().stream()
            // Map String values to Double
            .map(e -> Map.entry(e.getKey(), Double.parseDouble(e.getValue())))
            // Sort by price descending
            .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
            // Skip top (n - 1) highest items
            .skip(n - 1)
            // Pick the next element
            .findFirst();
    }

    public static void main(String[] args) {
        Map<String, String> catalog = Map.of(
            "Laptop", "1200",
            "Phone", "800",
            "TV", "1500",
            "Mouse", "50"
        );

        findNthHighest(catalog, 2).ifPresent(entry -> 
            System.out.println("2nd Highest: " + entry.getKey() + " -> $" + entry.getValue()) // Laptop -> $1200.0
        );
    }
}
```

#### 3. How `Map.Entry.comparingByValue().reversed()` Works Under the Hood
- `Map.Entry.comparingByValue()` returns a serializable comparator: `(c1, c2) -> c1.getValue().compareTo(c2.getValue())`.
- Calling `.reversed()` wraps that comparator in `Collections.reverseOrder(comparator)`, which inverts the comparison: `(e1, e2) -> comparator.compare(e2, e1)`.

---

### Problem 16: Sort Map by Values & Find Key from Value (Team Computer)

#### 1. Concept from Scratch
- **Task A**: Sort a `Map<String, Integer>` in ascending order by value and store it in a map that maintains that sorted order (`LinkedHashMap`).
- **Task B**: Look up a key corresponding to a specific value.

#### 2. Code Implementation

```java
import java.util.*;
import java.util.stream.Collectors;

public class MapOperations {

    // Task A: Sort Map by Values into a LinkedHashMap
    public static Map<String, Integer> sortMapByValue(Map<String, Integer> map) {
        return map.entrySet().stream()
            .sorted(Map.Entry.comparingByValue())
            .collect(Collectors.toMap(
                Map.Entry::getKey,
                Map.Entry::getValue,
                (oldVal, newVal) -> oldVal,
                LinkedHashMap::new // Preserves sorted insertion order!
            ));
    }

    // Task B: Find Key by Value
    public static Optional<String> findKeyByValue(Map<String, Integer> map, Integer targetValue) {
        return map.entrySet().stream()
            .filter(entry -> Objects.equals(entry.getValue(), targetValue))
            .map(Map.Entry::getKey)
            .findFirst();
    }

    public static void main(String[] args) {
        Map<String, Integer> scores = Map.of("Alice", 85, "Bob", 92, "Charlie", 78);
        System.out.println("Sorted by Score: " + sortMapByValue(scores)); 
        // Output: {Charlie=78, Alice=85, Bob=92}

        findKeyByValue(scores, 92).ifPresent(name -> System.out.println("Scored 92: " + name)); // Bob
    }
}
```

#### 3. How `Collectors.toMap(keyMapper, valueMapper, mergeFunction, mapFactory)` Works Under the Hood
- Standard `Collectors.toMap()` returns a `HashMap`, which does **not** guarantee iteration order.
- By providing the 4th argument `LinkedHashMap::new`, the collector instantiates a `LinkedHashMap`, which maintains a doubly-linked list across all its entries, preserving the sorted stream order.

---

### Problem 17: Multi-Level Grouping: Gender & Designation (NeoSoft & Cbt Infotech)

#### 1. Concept from Scratch
Given a list of employee records, generate a nested report of counts grouped first by **Gender** and then by **Designation**:
`Map<String, Map<String, Long>>`
```json
{
  "Female": { "Architect": 3, "Developer": 8 },
  "Male":   { "Architect": 5, "Developer": 12 }
}
```

#### 2. Code Implementation

```java
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

public record EmployeeRecord(String id, String name, String gender, String designation) {}

public class EmployeeAggregation {

    public static Map<String, Map<String, Long>> getGenderDesignationCount(List<EmployeeRecord> employees) {
        return employees.stream()
            .collect(Collectors.groupingBy(
                EmployeeRecord::gender, // Outer grouping: Gender
                Collectors.groupingBy(
                    EmployeeRecord::designation, // Inner grouping: Designation
                    Collectors.counting()       // Downstream calculation: Count
                )
            ));
    }

    public static void main(String[] args) {
        List<EmployeeRecord> team = List.of(
            new EmployeeRecord("1", "Alice", "Female", "Developer"),
            new EmployeeRecord("2", "Bob", "Male", "Developer"),
            new EmployeeRecord("3", "Carol", "Female", "Architect"),
            new EmployeeRecord("4", "David", "Male", "Developer"),
            new EmployeeRecord("5", "Eve", "Female", "Developer")
        );

        System.out.println(getGenderDesignationCount(team));
        // Output: {Female={Architect=1, Developer=2}, Male={Developer=2}}
    }
}
```

#### 3. How Multi-Level Grouping Works Under the Hood
- The outer `groupingBy` processes an `EmployeeRecord` and looks up the outer map by `gender`.
- Instead of inserting the record directly, it delegates downstream to the inner collector `Collectors.groupingBy(designation, counting())`.
- The inner collector creates an inner map, finds the key for `designation`, and delegates to `counting()` to increment the counter.

---

### Problem 18: Word Frequency Preserving Insertion Order (Bajaj Finserv)

#### 1. Concept from Scratch
- **Input**: `"India is my country, I love my country"`
- **Expected Output**: Order of first appearance preserved:  
  `India=1, is=1, my=2, country=2, I=1, love=1`

#### 2. Code Implementation

```java
import java.util.Arrays;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.function.Function;
import java.util.stream.Collectors;

public class WordOrderFrequency {

    public static Map<String, Long> countWordsPreservingOrder(String statement) {
        // Strip punctuation and split by whitespace
        String[] words = statement.replaceAll("[^a-zA-Z\\s]", "").split("\\s+");

        return Arrays.stream(words)
            .collect(Collectors.groupingBy(
                Function.identity(),
                LinkedHashMap::new,  // Preserves insertion order of first occurrence!
                Collectors.counting()
            ));
    }

    public static void main(String[] args) {
        String sentence = "India is my country, I love my country";
        System.out.println(countWordsPreservingOrder(sentence));
        // Output: {India=1, is=1, my=2, country=2, I=1, love=1}
    }
}
```

---

### Problem 19: Multiply All Elements using Stream `reduce()` (Saint Gobain)

#### 1. Concept from Scratch
- **Input**: `List.of(1, 2, 3, 4, 5)`
- **Goal**: Calculate product: $1 \times 2 \times 3 \times 4 \times 5 = 120$.
- **Reduction**: A reduction operation (fold) takes a sequence of input elements and combines them into a single summary result by repeated application of a binary operator.

#### 2. Code Implementation

```java
import java.util.List;

public class StreamMultiply {

    public static int multiplyAll(List<Integer> list) {
        if (list == null || list.isEmpty()) return 0;

        return list.stream()
            // 1 is the identity value for multiplication: 1 * x = x
            .reduce(1, (a, b) -> a * b);
    }

    public static void main(String[] args) {
        List<Integer> numbers = List.of(1, 2, 3, 4, 5);
        System.out.println("Product: " + multiplyAll(numbers)); // 120
    }
}
```

#### 3. How `reduce(identity, accumulator)` Works Under the Hood
- The identity element `1` initializes the accumulator: `result = 1`.
- For each element `e` in the stream: `result = accumulator.apply(result, e)`.
- If the stream is evaluated in parallel (`parallelStream()`), the stream partitions data into chunks, evaluates each chunk independently with identity `1`, and then combines the partial products using the combiner.

---

### Problem 20: Stream Laziness & Intermediate vs Terminal Operations Puzzle

#### 1. The Interview Puzzle (Applied Cloud Computing)
```java
List<Integer> list = new ArrayList<>();
list.add(4);
list.add(4);
list.add(4);
list.add(4);

list.stream().map(i -> i * i); // Line X

for (Integer num : list) {
    System.out.println(num);
}
```
- **Interview Question**: What is the output of the code above? Does it print 16 or 4?
- **Answer**: It prints:
  ```
  4
  4
  4
  4
  ```

#### 2. Why it happens (The Core Concept)
1. **Streams are Lazy**: Intermediate operations like `.map()`, `.filter()`, and `.sorted()` are purely declarative configurations. They do **not** execute until an eager **terminal operation** (e.g., `.collect()`, `.toList()`, `.forEach()`) is invoked on that stream. Because there is no terminal operation attached on Line X, the `.map()` lambda `i -> i * i` is **never executed**.
2. **Streams are Non-Mutating**: Even if a terminal operation had been invoked (e.g., `list.stream().map(i -> i * i).toList()`), the original `list` remains completely unmodified because Streams never mutate their backing data source.

---

# Part 4: JVM Under-the-Hood Reference for Stream & Array Methods

| Built-In Method | Classification | Time Complexity | Statefulness | Internal JVM Mechanism |
| :--- | :--- | :--- | :--- | :--- |
| **`Arrays.stream(arr)`** | Source | $O(1)$ | Stateless | Creates an `ArraySpliterator.OfInt` directly indexing the primitive array. |
| **`filter(Predicate)`** | Intermediate | $O(N)$ | Stateless | Inserts a `StatelessOp` pipeline stage evaluating `predicate.test()`. Drops unmatching elements. |
| **`map(Function)`** | Intermediate | $O(N)$ | Stateless | 1-to-1 transformation applying `function.apply()` per downstream element. |
| **`flatMap(Function)`** | Intermediate | $O(N)$ | Stateless | Maps each element to a sub-stream and flattens them into a unified pipeline. |
| **`distinct()`** | Intermediate | $O(N)$ | Stateful | Uses a `LinkedHashSet` internally; drops elements where `set.add(e) == false`. |
| **`sorted(Comparator)`** | Intermediate | $O(N \log N)$ | Stateful | Buffers all pipeline elements into an internal array and executes **TimSort**. |
| **`skip(n)`** | Intermediate | $O(N)$ | Stateful | Maintains an atomic long counter; discards the first $n$ elements. |
| **`limit(n)`** | Intermediate | $O(n)$ | Stateful / Short-circuiting | Terminates ingestion once $n$ elements pass through downstream. |
| **`allMatch(Predicate)`** | Terminal | $O(N)$ | Short-circuiting | Cancels stream pipeline on the very first element returning `false`. |
| **`findFirst()`** | Terminal | $O(1)$ | Short-circuiting | Returns the first available element wrapped in `Optional` and halts traversal. |
| **`collect(Collector)`** | Terminal | $O(N)$ | Eager | Manages supplier, accumulator, combiner, and finisher to fold stream elements into a target container. |
| **`reduce(identity, op)`**| Terminal | $O(N)$ | Eager | Folds stream elements into an accumulator using associative binary operations. |
| **`System.arraycopy()`** | Native System | $O(N)$ | Direct Memory Copy | Native C++ `memmove` utilizing CPU vector/AVX registers for high-speed block copies. |
| **`HashSet.add()`** | Collection API | $O(1)$ avg | Hash Lookup | Delegates to `HashMap.put(e, PRESENT) == null`, returning `false` on collision. |

