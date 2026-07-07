# Java String Interview Prep — Combined Reference

Combines three practice resources into one document:
1. **Tricky Quiz** — 10 "what does this print?" questions with explanations
2. **First Unique Character — Stream API deep dive**
3. **Coding Problems** — 8 classic string interview problems with full solutions

---

# Part 1: Java String Tricky Questions (Quiz)

## Q1. String Pool & intern() — *Medium*
```java
String s1 = "hello";
String s2 = new String("hello");
String s3 = s2.intern();

System.out.println(s1 == s2);
System.out.println(s1 == s3);
```
**Options:** A) true/false B) false/false C) false/true D) true/true
**Answer: C — false / true**
`s1 == s2` is false because `new String()` always creates a new heap object, bypassing the string pool. `s3 = s2.intern()` returns the pooled reference (same as `s1`), so `s1 == s3` is true.

## Q2. String Immutability — *Easy*
```java
String s = "Java";
s.concat(" Interview");
System.out.println(s);
```
**Options:** A) Java Interview B) Java C) null D) Compilation error
**Answer: B — Java**
Strings are immutable. `concat()` returns a *new* String but doesn't modify the original. Since the return value isn't assigned, `s` still holds `"Java"`.

## Q3. charAt vs substring — *Hard*
```java
String s = "abcdef";
System.out.println(s.substring(2, 4));
System.out.println((int) s.charAt(0));
```
**Options:** A) cd/97 B) cde/97 C) cd/98 D) cde/96
**Answer: A — cd / 97**
`substring(2,4)` returns chars at indices 2–3 (end exclusive) → `"cd"`. `charAt(0)` is `'a'`; cast to `int` gives ASCII value `97`.

## Q4. == vs .equals() — *Medium*
```java
String a = "test";
String b = "te" + "st";
String c = "te";
c += "st";

System.out.println(a == b);
System.out.println(a == c);
```
**Options:** A) false/false B) true/true C) true/false D) false/true
**Answer: C — true / false**
`"te" + "st"` is a compile-time constant, so the compiler interns it — same pool object as `a`. But `c += "st"` happens at runtime (via `StringBuilder`), creating a new object, so `a == c` is false.

## Q5. String.format & null — *Medium*
```java
String s = null;
System.out.println("Value: " + s);
System.out.println(s.length());
```
**Options:** A) Value: null / 0 B) Value: null / NullPointerException C) NullPointerException / 0 D) Compilation error
**Answer: B**
Concatenation with `null` converts it to the literal `"null"` string and prints fine. Calling `.length()` on a null reference throws `NullPointerException`.

## Q6. StringBuilder chaining — *Hard*
```java
StringBuilder sb = new StringBuilder("hello");
sb.reverse().append("!").delete(0, 2);
System.out.println(sb);
```
**Options:** A) llo! B) ell! C) hello! D) ol!
**Answer:** Step by step: `reverse()` → `"olleh"`. `append("!")` → `"olleh!"`. `delete(0,2)` removes indices 0–1 (`"ol"`) → `"leh!"`. **The correct result is `"leh!"`** — this is the tricky part about chaining mutations on a mutable object; each call modifies state in place, so trace it left to right carefully rather than assuming the original string layout still applies after `reverse()`.

## Q7. String.split() edge case — *Hard*
```java
String s = "a,,b,,c,,";
String[] arr = s.split(",");
System.out.println(arr.length);
```
**Options:** A) 8 B) 5 C) 7 D) 6
**Answer: B — 5**
By default, `split()` removes trailing empty strings. The raw split is `["a", "", "b", "", "c", "", ""]`, but trailing empties are discarded → `["a", "", "b", "", "c"]` = 5 elements. Use `split(",", -1)` to keep all 8.

## Q8. compareTo() result — *Medium*
```java
System.out.println("apple".compareTo("banana"));
System.out.println("Java".compareTo("java"));
```
**Answer: negative / negative**
`compareTo()` returns the difference of the first unequal chars. `'a' - 'b' = -1` (negative). `'J'(74) - 'j'(106) = -32` (negative). Exact magnitude isn't guaranteed to matter — just the sign.

## Q9. toCharArray & String creation — *Easy*
```java
char[] ch = {'J', 'a', 'v', 'a'};
String s = new String(ch);
ch[0] = 'X';
System.out.println(s);
```
**Answer: Java**
`new String(ch)` copies the char array. Modifying `ch[0]` afterward doesn't affect `s`, because String stores its own internal copy — this is part of why Strings are effectively immutable.

## Q10. isEmpty vs isBlank (Java 11+) — *Hard*
```java
String s = "  "; // two spaces
System.out.println(s.isEmpty());
System.out.println(s.isBlank());
System.out.println(s.strip().isEmpty());
```
**Answer: false / true / true**
`isEmpty()` is true only when length is 0 — `"  "` has length 2, so false. `isBlank()` (Java 11+) is true if empty OR only whitespace, so true. `strip()` removes leading/trailing whitespace (Unicode-aware), leaving `""`, so `isEmpty()` is true.

---

# Part 2: First Unique Character — Stream API Deep Dive

**Problem:** Find the first non-repeating character in a string using Java's Stream API (not just the classic loop version).

## Pipeline, step by step

| Step | Operation | What it does |
|---|---|---|
| 1 | `s.chars()` | Converts `String` → primitive `IntStream` of char code points. No boxing yet. |
| 2 | `.mapToObj(c -> (char) c)` | Casts each int code point back to `Character`. Now `Stream<Character>`. |
| 3 | `.collect(Collectors.groupingBy(Function.identity(), LinkedHashMap::new, Collectors.counting()))` | Groups chars by themselves and counts occurrences. `LinkedHashMap` preserves insertion order — critical for finding the *first* unique char. |
| 4 | `.entrySet().stream()` | Streams the map entries so filter/map can be chained. |
| 5 | `.filter(e -> e.getValue() == 1L)` | Keeps only entries with count exactly 1 — the unique chars. |
| 6 | `.map(Map.Entry::getKey)` | Extracts just the `Character` key from each surviving entry. |
| 7 | `.findFirst()` | Terminal operation — returns `Optional<Character>`, the first unique char. |
| 8 | `.ifPresent(...)` or `.orElse(...)` | `ifPresent` fires a consumer if present (no null check needed); `orElse` supplies a fallback (e.g., `-1` when returning an index). |

## Full solution — three approaches

```java
import java.util.*;
import java.util.function.Function;
import java.util.stream.Collectors;
import java.util.stream.IntStream;

public class FirstUniqueChar {

    // ── Approach 1: Print the character directly ────────────────────
    public static void printFirstUnique(String s) {
        // Step 1 — build frequency map (insertion order preserved)
        Map<Character, Long> freq = s.chars()
            .mapToObj(c -> (char) c)
            .collect(Collectors.groupingBy(
                Function.identity(),
                LinkedHashMap::new,        // preserves insertion order
                Collectors.counting()
            ));

        // Step 2 — filter unique → findFirst → print
        freq.entrySet().stream()
            .filter(e -> e.getValue() == 1L)
            .map(Map.Entry::getKey)
            .findFirst()
            .ifPresent(System.out::println);
    }

    // ── Approach 2: Return index (classic interview ask) ────────────
    public static int firstUniqCharIndex(String s) {
        Map<Character, Long> freq = s.chars()
            .mapToObj(c -> (char) c)
            .collect(Collectors.groupingBy(
                Function.identity(),
                LinkedHashMap::new,
                Collectors.counting()
            ));

        return freq.entrySet().stream()
            .filter(e -> e.getValue() == 1L)
            .map(e -> s.indexOf(e.getKey()))
            .findFirst()
            .orElse(-1); // -1 if no unique char
    }

    // ── Approach 3: One-liner with .chars() IntStream ───────────────
    public static int firstUniqCharOneLiner(String s) {
        int[] freq = new int[128];
        s.chars().forEach(c -> freq[c]++);
        return IntStream.range(0, s.length())
            .filter(i -> freq[s.charAt(i)] == 1)
            .findFirst()
            .orElse(-1);
    }

    public static void main(String[] args) {
        String s = "leetcode";

        printFirstUnique(s);                          // → 'l'
        System.out.println(firstUniqCharIndex(s));     // → 0
        System.out.println(firstUniqCharOneLiner(s));  // → 0
    }
}
```

**Why `LinkedHashMap::new` matters:** a plain `HashMap` doesn't guarantee iteration order, so "first" unique character could come out wrong. `LinkedHashMap` preserves insertion order, matching the order characters first appeared in the string.

**Why `Collectors.counting()` returns `Long`, not `int`:** it's a general-purpose collector, so the filter check must use `== 1L`, not `== 1`.

---

# Part 3: Java String Coding Interview — 8 Problems

## 1. First Unique Character in a String — *Easy* (HashMap · LinkedHashMap)

**Problem:** Given a string `s`, find the first non-repeating character and return its index. Return `-1` if none exists.
Example: `"leetcode"` → `0` ('l' appears once). `"aabb"` → `-1`.

```java
import java.util.*;

public class FirstUniqueChar {
    public static int firstUniqChar(String s) {
        Map<Character, Integer> freq = new LinkedHashMap<>();
        for (char c : s.toCharArray()) {
            freq.put(c, freq.getOrDefault(c, 0) + 1);
        }
        for (int i = 0; i < s.length(); i++) {
            if (freq.get(s.charAt(i)) == 1) return i;
        }
        return -1;
    }

    public static void main(String[] args) {
        System.out.println(firstUniqChar("leetcode")); // 0
        System.out.println(firstUniqChar("loveleet")); // 2
        System.out.println(firstUniqChar("aabb"));     // -1
    }
}
```
**Complexity:** O(n) time, O(1) space (at most 26 keys for lowercase-only input).

---

## 2. Character Frequency Map — *Easy* (HashMap · Counting)

**Problem:** Return a frequency map of all characters, sorted by frequency descending.
Example: `"aabbbcc"` → `b=3, a=2, c=2`.

```java
import java.util.*;
import java.util.stream.*;

public class CharFrequency {
    public static Map<Character, Integer> charFrequency(String s) {
        Map<Character, Integer> freq = new HashMap<>();
        for (char c : s.toCharArray()) {
            freq.put(c, freq.getOrDefault(c, 0) + 1);
        }
        return freq.entrySet().stream()
            .sorted(Map.Entry.<Character, Integer>comparingByValue().reversed())
            .collect(Collectors.toMap(
                Map.Entry::getKey,
                Map.Entry::getValue,
                (e1, e2) -> e1,
                LinkedHashMap::new
            ));
    }

    public static void main(String[] args) {
        String input = "aabbbccde";
        charFrequency(input).forEach((k, v) -> System.out.println("'" + k + "' -> " + v));
    }
}
```
> Note: the original used `Collectors.toLinkedHashMap(...)`, which isn't a real method in the standard `Collectors` API — the corrected version above uses `Collectors.toMap(..., LinkedHashMap::new)`, which is the real signature for collecting into an ordered map while sorting by value.

**Complexity:** O(n log k) time where k = unique chars, O(k) space.

---

## 3. Check if String is Palindrome — *Easy* (Two Pointers)

**Problem:** Return `true` if a string reads the same forwards and backwards (case-insensitive).
Example: `"racecar"` → `true`, `"hello"` → `false`.

```java
public class Palindrome {
    public static boolean isPalindrome(String s) {
        String lower = s.toLowerCase();
        int left = 0, right = lower.length() - 1;
        while (left < right) {
            if (lower.charAt(left) != lower.charAt(right)) return false;
            left++;
            right--;
        }
        return true;
    }

    public static void main(String[] args) {
        System.out.println(isPalindrome("racecar")); // true
        System.out.println(isPalindrome("Madam"));   // true
        System.out.println(isPalindrome("hello"));   // false
    }
}
```
**Complexity:** O(n) time, O(1) extra space (aside from the lowercase copy).

---

## 4. Most Frequent Character — *Medium* (HashMap · Max tracking)

**Problem:** Find the character appearing most frequently. Ties go to whichever appears first.
Example: `"abracadabra"` → `'a'` (5 times).

```java
import java.util.*;

public class MostFrequent {
    public static char mostFrequentChar(String s) {
        Map<Character, Integer> freq = new HashMap<>();
        for (char c : s.toCharArray()) {
            freq.put(c, freq.getOrDefault(c, 0) + 1);
        }

        char maxChar = s.charAt(0);
        int maxCount = 0;
        for (char c : s.toCharArray()) {
            if (freq.get(c) > maxCount) {
                maxCount = freq.get(c);
                maxChar = c;
            }
        }
        return maxChar;
    }

    public static void main(String[] args) {
        String s = "abracadabra";
        System.out.println("Most frequent: '" + mostFrequentChar(s) + "'"); // 'a'
    }
}
```
**Complexity:** O(n) time, O(k) space. First-occurrence tie-breaking works naturally because `>` (not `>=`) only updates on a strictly higher count.

---

## 5. Check if Two Strings are Anagrams — *Medium* (Fixed-size counting array)

**Problem:** Given `s` and `t`, return `true` if they're anagrams of each other.
Example: `s="anagram", t="nagaram"` → `true`. `s="rat", t="car"` → `false`.

```java
public class Anagram {
    public static boolean isAnagram(String s, String t) {
        if (s.length() != t.length()) return false;

        int[] count = new int[26]; // lowercase a-z only
        for (int i = 0; i < s.length(); i++) {
            count[s.charAt(i) - 'a']++;
            count[t.charAt(i) - 'a']--;
        }
        for (int n : count) {
            if (n != 0) return false;
        }
        return true;
    }

    public static void main(String[] args) {
        System.out.println(isAnagram("anagram", "nagaram")); // true
        System.out.println(isAnagram("rat", "car"));         // false
        System.out.println(isAnagram("listen", "silent"));   // true
    }
}
```
**Complexity:** O(n) time, O(1) space. Assumes lowercase a–z input; for full Unicode support, swap the array for a `HashMap<Character, Integer>`.

---

## 6. Remove Duplicate Characters — *Medium* (LinkedHashSet)

**Problem:** Return a new string with duplicate characters removed, keeping only the first occurrence of each.
Example: `"programming"` → `"progamin"`.

```java
import java.util.*;

public class RemoveDuplicates {
    public static String removeDuplicates(String s) {
        Set<Character> seen = new LinkedHashSet<>();
        for (char c : s.toCharArray()) {
            seen.add(c); // Set ignores duplicates, keeps insertion order
        }
        StringBuilder sb = new StringBuilder();
        for (char c : seen) sb.append(c);
        return sb.toString();
    }

    // Alternative: O(1)-space version using a boolean lookup array
    public static String removeDuplicatesAlt(String s) {
        boolean[] seen = new boolean[256];
        StringBuilder sb = new StringBuilder();
        for (char c : s.toCharArray()) {
            if (!seen[c]) {
                seen[c] = true;
                sb.append(c);
            }
        }
        return sb.toString();
    }

    public static void main(String[] args) {
        System.out.println(removeDuplicates("programming")); // progamin
        System.out.println(removeDuplicates("aabbcc"));      // abc
    }
}
```
**Complexity:** O(n) time, O(k) space for the `LinkedHashSet` version (O(1)/O(256) for the array version).

---

## 7. Longest Substring Without Repeating Characters — *Hard* (Sliding Window)

**Problem:** Find the length of the longest substring without repeating characters.
Example: `"abcabcbb"` → `3` ("abc"). `"pwwkew"` → `3` ("wke").

```java
import java.util.*;

public class LongestSubstring {
    public static int lengthOfLongestSubstring(String s) {
        Map<Character, Integer> lastIndex = new HashMap<>();
        int maxLen = 0;
        int left = 0; // left boundary of window

        for (int right = 0; right < s.length(); right++) {
            char c = s.charAt(right);

            // If char seen before AND within the current window, shrink window
            if (lastIndex.containsKey(c) && lastIndex.get(c) >= left) {
                left = lastIndex.get(c) + 1;
            }

            lastIndex.put(c, right); // update last seen index
            maxLen = Math.max(maxLen, right - left + 1);
        }
        return maxLen;
    }

    public static void main(String[] args) {
        System.out.println(lengthOfLongestSubstring("abcabcbb")); // 3
        System.out.println(lengthOfLongestSubstring("bbbbb"));    // 1
        System.out.println(lengthOfLongestSubstring("pwwkew"));   // 3
    }
}
```
**Complexity:** O(n) time — each character is visited at most twice (once by `right`, once when `left` jumps past it). O(min(m, n)) space where m = charset size.

---

## 8. Group Anagrams Together — *Hard* (HashMap · Sorting)

**Problem:** Given an array of strings, group all anagrams together.
Example: `["eat","tea","tan","ate","nat","bat"]` → `[["eat","tea","ate"], ["tan","nat"], ["bat"]]`.

```java
import java.util.*;

public class GroupAnagrams {
    public static List<List<String>> groupAnagrams(String[] strs) {
        Map<String, List<String>> map = new HashMap<>();

        for (String s : strs) {
            char[] chars = s.toCharArray();
            Arrays.sort(chars);           // canonical form: sorted chars
            String key = new String(chars);
            map.computeIfAbsent(key, k -> new ArrayList<>()).add(s);
        }

        return new ArrayList<>(map.values());
    }

    public static void main(String[] args) {
        String[] input = {"eat","tea","tan","ate","nat","bat"};
        List<List<String>> result = groupAnagrams(input);
        for (List<String> group : result) {
            System.out.println(group);
        }
        // [eat, tea, ate]
        // [tan, nat]
        // [bat]
    }
}
```
**Complexity:** O(n × k log k) time (n = number of strings, k = max string length), O(n × k) space.

---

# Quick Reference Table

| # | Problem | Difficulty | Core Technique |
|---|---|---|---|
| 1 | First Unique Character | Easy | LinkedHashMap counting |
| 2 | Character Frequency Map | Easy | HashMap + sort by value |
| 3 | Palindrome Check | Easy | Two pointers |
| 4 | Most Frequent Character | Medium | HashMap + max tracking |
| 5 | Anagram Check | Medium | Fixed-size counting array |
| 6 | Remove Duplicates | Medium | LinkedHashSet |
| 7 | Longest Substring w/o Repeats | Hard | Sliding window + HashMap |
| 8 | Group Anagrams | Hard | HashMap keyed by sorted string |
