# Java Interview Practice — Questions & Answers

---

### Q1. Find names that repeat more than once in an array

**Problem with original code:** used `arr.length()` (arrays use `.length`, no parentheses), compared `arr[i] == arr[i+1]` (only checks adjacent elements, and `==` compares references not content), and declared `count`/`name` inside the loop so they reset every iteration.

```java
public class Main {
    public static void main(String[] args) {
        String[] arr = {"Rahul", "Suresh", "Amit", "Rahul"};
        Map<String, Integer> countMap = new HashMap<>();

        for (String s : arr) {
            countMap.merge(s, 1, Integer::sum);
        }

        for (Map.Entry<String, Integer> entry : countMap.entrySet()) {
            if (entry.getValue() > 1) {
                System.out.println("Name: " + entry.getKey() + " Name's count = " + entry.getValue());
            }
        }
    }
}
```
**Output:** `Name: Rahul Name's count = 2`

---

### Q2. Find the second highest number in an array

```java
int[] array = {1, 3, 5, 2, 6, 7, 10};

int highest = Integer.MIN_VALUE, secondHighest = Integer.MIN_VALUE;
for (int num : array) {
    if (num > highest) {
        secondHighest = highest;
        highest = num;
    } else if (num > secondHighest && num != highest) {
        secondHighest = num;
    }
}
System.out.println("Second highest: " + secondHighest); // 7
```

**Stream one-liner:**
```java
int second = Arrays.stream(array).boxed()
        .sorted(Comparator.reverseOrder())
        .distinct()
        .skip(1)
        .findFirst()
        .orElseThrow();
```

---

### Q3. Validation for REST API request values

General approach in Spring Boot: use **Bean Validation (JSR-380)** annotations on the request DTO, and let `@Valid` trigger validation automatically.

```java
public class EmployeeRequest {
    @NotBlank(message = "Name is required")
    private String name;

    @Min(value = 1000, message = "Salary must be at least 1000")
    private double salary;

    @NotNull
    private Integer deptId;
    // getters/setters
}

@RestController
public class EmployeeController {
    @PostMapping("/employee")
    public ResponseEntity<?> save(@Valid @RequestBody EmployeeRequest req) {
        return ResponseEntity.ok(req);
    }
}

@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<?> handleValidation(MethodArgumentNotValidException ex) {
    return ResponseEntity.badRequest().body(ex.getBindingResult().getAllErrors());
}
```

---

### Q4. Group employees by department using streams

```java
class Employee {
    int id;
    String name;
    double salary;
    int deptId;
    // constructor, getters
}

Map<Integer, List<Employee>> empListByDept =
        empList.stream().collect(Collectors.groupingBy(Employee::getDeptId));
```
Result shape: `{101=[emp1, emp4, emp7], 102=[emp2, emp3, emp5]}`

---

### Q5. Find the highest-salaried employee

```java
Employee highestPaid = empList.stream()
        .max(Comparator.comparingDouble(Employee::getSalary))
        .orElseThrow();
```

---

### Q6. Find an employee named "Ram"

**Bug:** `.filter.findMatch(...)` — not valid stream syntax. `filter()` takes a `Predicate` lambda, and you use `findFirst()`/`findAny()`, not `findMatch`.

```java
Optional<Employee> ram = empList.stream()
        .filter(emp -> emp.getName().equals("Ram"))
        .findFirst();

ram.ifPresentOrElse(
        e -> System.out.println("Found: " + e.getName()),
        () -> System.out.println("Not found")
);
```

---

### Q7. Sort an array (fix the bubble sort)

**Bugs:** `arr.length()` should be `arr.length`; `int[] sortedArray = new int[]` is invalid (missing size); stray `)` after assignments; single pass isn't enough — needs a nested loop; loop condition should stop at `length-1` to avoid `ArrayIndexOutOfBoundsException`.

```java
int[] arr = {78, 34, 1, 3, 90, 34, -1, -4, 6, 55, 20, -65};

for (int i = 0; i < arr.length - 1; i++) {
    for (int j = 0; j < arr.length - 1 - i; j++) {
        if (arr[j] > arr[j + 1]) {
            int tmp = arr[j];
            arr[j] = arr[j + 1];
            arr[j + 1] = tmp;
        }
    }
}
System.out.println(Arrays.toString(arr));
// [-65, -4, -1, 1, 3, 6, 20, 34, 34, 55, 78, 90]
```

---

### Q8. SQL: Top 3 highest salaries

```sql
SELECT salary FROM employee ORDER BY salary DESC LIMIT 3;
-- (LIMIT 0,3 also works in MySQL — offset 0, count 3)
```

---

### Q9. Fix the Employee class

**Bugs:** `retun` typo, constructor assigns `this.name = name` but field is `empName`, two methods both named `getEmpId()` (duplicate method signature — won't compile), missing `getEmpName()`.

```java
class Employee {
    private Integer id;
    private String empName;

    Employee(Integer id, String empName) {
        this.id = id;
        this.empName = empName;
    }

    public Integer getEmpId() {
        return id;
    }

    public String getEmpName() {
        return empName;
    }
}
```

---

### Q10. What is a "Qualifier" (Spring)?

`@Qualifier` is used alongside `@Autowired` in Spring to resolve ambiguity when **more than one bean of the same type** exists. It tells Spring exactly which bean to inject by name.

```java
@Component("emailService")
class EmailNotifier implements Notifier {}

@Component("smsService")
class SmsNotifier implements Notifier {}

@Service
class OrderService {
    @Autowired
    @Qualifier("smsService")
    private Notifier notifier;
}
```

---

### Q11. How to use serialization / custom serialization

Standard serialization: implement `Serializable` — the JVM handles converting the object to bytes.

```java
class Employee implements Serializable {
    private static final long serialVersionUID = 1L;
    private String name;
    private transient String password; // excluded from serialization
}
```

**Custom serialization** — override `writeObject`/`readObject` to control exactly what gets written/read:

```java
class Employee implements Serializable {
    private String name;
    private transient String password;

    private void writeObject(ObjectOutputStream out) throws IOException {
        out.defaultWriteObject();
        out.writeObject(encrypt(password));
    }

    private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
        in.defaultReadObject();
        this.password = decrypt((String) in.readObject());
    }
}
```
Use `transient` to exclude sensitive/non-serializable fields, and custom methods when you need extra logic (encryption, versioning, etc.) during save/load.

---

### Q12. What does this REST endpoint return? (Parameter binding trap)

```java
@RestController
public class SampleController {
    @RequestMapping("/map")
    public String map(@RequestParam("bar") String foo, @RequestParam("foo") String bar) {
        return bar + foo;
    }
}
```
This compiles and runs fine — it's a naming trick, not a bug. The **query parameter name** is what matters (`"bar"`, `"foo"`), not the Java variable name. So calling `/map?foo=X&bar=Y` binds `foo="Y"` (because it's annotated `@RequestParam("bar")`) and `bar="X"`. The method returns `bar + foo` = `"X" + "Y"` = `"XY"`.

---

### Q13. Method overriding + checked exceptions (what's the output?)

```java
class Parent {
    public void m1() { System.out.println("Inside Parent class M1 Method"); }
    public void m2() throws NullPointerException { System.out.println("Inside Parent class M2 Method"); }
}

class Child extends Parent {
    public void m1() { System.out.println("Inside Child class M1 Method"); }
    public void m2() { System.out.println("Inside Child class M2 Method"); }
    public void m3() { System.out.println("Inside Child class M3 Method"); }
}

public class DemoClass {
    public static void main(String[] args) {
        Parent p = new Parent();
        p.m1(); p.m2();

        Parent p1 = new Child();
        p1.m1(); p1.m2();

        Child c = new Child();
        c.m1(); c.m2(); c.m3();
    }
}
```
**Output:**
```
Inside Parent class M1 Method
Inside Parent class M2 Method
Inside Child class M1 Method
Inside Child class M2 Method
Inside Child class M1 Method
Inside Child class M2 Method
Inside Child class M3 Method
```
**Key point:** `NullPointerException` is a `RuntimeException` (unchecked), so declaring `throws NullPointerException` doesn't force callers to catch it — that's why removing it in the override still compiles. Method calls resolve based on **runtime type** (dynamic dispatch) for overridden methods, but `m3()` can only be called through a `Child` reference since it isn't in `Parent`.

---

### Q14. User-defined custom exception

```java
class InsufficientBalanceException extends Exception {
    public InsufficientBalanceException(String message) {
        super(message);
    }
}

class BankAccount {
    private double balance;

    void withdraw(double amount) throws InsufficientBalanceException {
        if (amount > balance) {
            throw new InsufficientBalanceException("Insufficient balance for withdrawal of " + amount);
        }
        balance -= amount;
    }
}
```

---

### Q15. String pool behavior

```java
String str1 = "Hello";
String str2 = new String("Hello");
String str3 = "Hello";

System.out.println(str1 == str2); // false — new String() creates a new object on the heap
System.out.println(str1 == str3); // true  — both point to the same literal in the String pool
System.out.println(str1.equals(str2)); // true — content is equal
```
Note: `str1.toUpperCase()` (not `toUppercase()`) returns a **new** String — `String` is immutable, so it doesn't modify `str1` in place; you must capture the return value: `str1 = str1.toUpperCase();`

---

### Q16. Get even numbers using Streams

```java
List<Integer> nums = List.of(10, 15, 8, 49, 25, 98, 32);

List<Integer> evens = nums.stream()
        .filter(n -> n % 2 == 0)
        .collect(Collectors.toList());
// [10, 8, 98, 32]
```

---

### Q17. Find the maximum value from a list of maps

```java
List<Map<String, Integer>> list = List.of(
        Map.of("a", 1, "b", 2),
        Map.of("a", 3, "b", 4),
        Map.of("a", 11, "b", 22),
        Map.of("a", 43, "b", 44)
);

int max = list.stream()
        .flatMap(m -> m.values().stream())
        .max(Integer::compareTo)
        .orElseThrow();
// 44
```

---

### Q18. Sort a list and remove duplicates using streams

**Bugs:** `Arrays.IntStream()` isn't valid — it's `Arrays.stream(array)`; `distict`/`collet`/`Collectors.toList()` typos.

```java
int[] arr = {15, 89, 54, 12, 36, 79, 15, 98, 89};

List<Integer> result = Arrays.stream(arr)
        .boxed()
        .distinct()
        .sorted()
        .collect(Collectors.toList());
// [12, 15, 36, 54, 79, 89, 98]
```

---

### Q19. Find names containing vowels

```java
List<List<String>> names = List.of(
        List.of("John", "Bravo"),
        List.of("Mary", "Lee"),
        List.of("Bob", "Johnson")
);

Set<Character> vowels = Set.of('a', 'e', 'i', 'o', 'u');

names.stream()
     .flatMap(List::stream)
     .forEach(name -> {
         String found = name.toLowerCase().chars()
                 .filter(c -> vowels.contains((char) c))
                 .distinct()
                 .mapToObj(c -> String.valueOf((char) c))
                 .collect(Collectors.joining(","));
         System.out.println(name + " -> " + found);
     });
```

---

### Q20. Reverse a string without built-in functions

```java
public static String reverse(String str) {
    char[] chars = str.toCharArray();
    int left = 0, right = chars.length - 1;
    while (left < right) {
        char temp = chars[left];
        chars[left] = chars[right];
        chars[right] = temp;
        left++;
        right--;
    }
    return new String(chars);
}
// reverse("Hello") -> "olleH"
```

---

### Q21. Count employees with age > 30, grouped by name

```java
Map<String, Long> countByName = empList.stream()
        .filter(e -> e.getAge() > 30)
        .collect(Collectors.groupingBy(Employee::getName, Collectors.counting()));
```

---

### Q22. Reverse each word in a sentence, keep word order

Goal: `"Times of India"` → `"semiT fo aidnI"`

**Bugs:** `int arr : strArr` should be `String word : strArray`; `arr.lenght()` → `word.length()`.

```java
String str = "Times of India";
String[] words = str.split(" ");
StringBuilder result = new StringBuilder();

for (String word : words) {
    for (int i = word.length() - 1; i >= 0; i--) {
        result.append(word.charAt(i));
    }
    result.append(" ");
}
System.out.println(result.toString().trim()); // semiT fo aidnI
```

---

### Q23. SQL: Find the second-highest (Nth-highest) salary

```sql
SELECT MAX(salary) FROM employee
WHERE salary < (SELECT MAX(salary) FROM employee);
```
General Nth-highest pattern (MySQL 8+):
```sql
SELECT DISTINCT salary FROM employee
ORDER BY salary DESC
LIMIT 1 OFFSET (N-1);
```

---

### Q24. Group students by department using streams

```java
class Student {
    String id, name, department;
}

Map<String, List<Student>> byDept = studentList.stream()
        .collect(Collectors.groupingBy(Student::getDepartment));
```

---

### Q25. Comparator classes for sorting Employees

```java
class Employee {
    int id;
    String name;
    double salary;
}

class SortBySalary implements Comparator<Employee> {
    public int compare(Employee e1, Employee e2) {
        return Double.compare(e1.getSalary(), e2.getSalary());
    }
}

class SortByName implements Comparator<Employee> {
    public int compare(Employee e1, Employee e2) {
        return e1.getName().compareTo(e2.getName());
    }
}

class SortById implements Comparator<Employee> {
    public int compare(Employee e1, Employee e2) {
        return Integer.compare(e1.getId(), e2.getId());
    }
}
```

**Equivalent inline stream sorts (fixing the typos `Comparoter`, `Collection.toList()` → `Collectors.toList()`):**
```java
List<Employee> bySalary = employeeList.stream()
        .sorted(Comparator.comparingDouble(Employee::getSalary))
        .collect(Collectors.toList());

List<Employee> byName = employeeList.stream()
        .sorted(Comparator.comparing(Employee::getName))
        .collect(Collectors.toList());

List<Employee> byId = employeeList.stream()
        .sorted(Comparator.comparingInt(Employee::getId))
        .collect(Collectors.toList());

List<Employee> namedRahul = employeeList.stream()
        .filter(e -> e.getName().equals("Rahul"))
        .collect(Collectors.toList());
```
Note: `Double.compare`/subtraction directly on salary (`e1.getSalary() - e2.getSalary()`) is risky with floating point — prefer `Double.compare()` or `Comparator.comparingDouble()`.

---

### Q26. Count occurrences of employee names using HashMap

**Bugs:** `conatinsKey` typo, `map.getValue(...)` isn't a method (should be `map.get(...)`).

```java
Map<String, Integer> countMap = new HashMap<>();

for (Employee e : employeeList) {
    if (countMap.containsKey(e.getName())) {
        int count = countMap.get(e.getName());
        countMap.put(e.getName(), count + 1);
    } else {
        countMap.put(e.getName(), 1);
    }
}
// Simpler: countMap.merge(e.getName(), 1, Integer::sum);
```

---

### Q27. Overriding a method with a narrower checked exception — does this compile?

```java
class Vehicle {
    void run() throws Exception { System.out.println("Vehicle is running"); }
}

class MyException extends Exception {}

class Bike2 extends Vehicle {
    public void run() throws MyException { System.out.println("Bike is running safely"); }
}

public class Overloading {
    public static void main(String[] args) throws Exception {
        Vehicle obj = new Bike2();
        obj.run();
    }
}
```
**Yes, this compiles.** A rule of overriding: an overriding method may declare the **same, a narrower, or no checked exception** compared to the parent — but never a broader/new checked exception. Since `MyException` extends `Exception` (a narrower/sibling-but-subclass relationship isn't required here — `MyException` is a subtype of `Exception`), this is legal. Also note access modifiers: `run()` was package-private in `Vehicle` (no modifier) but `public` in `Bike2` — widening visibility is allowed.

**Output:** `Bike is running safely`

---

### Q28. Swap using a wrapper object (simulate pass-by-reference)

**Bugs:** `swapFunction(a, b)` should reference `aWrraper, bWrraper`; missing closing `)` on the last `println`.

```java
public class CallJava {
    public static void main(String[] args) {
        IntWrapper aWrapper = new IntWrapper(30);
        IntWrapper bWrapper = new IntWrapper(45);
        System.out.println("Before swapping, a = " + aWrapper.a + " and b = " + bWrapper.a);

        swapFunction(aWrapper, bWrapper);
        System.out.println("After swapping, a = " + aWrapper.a + " and b = " + bWrapper.a);
    }

    public static void swapFunction(IntWrapper a, IntWrapper b) {
        int temp = a.a;
        a.a = b.a;
        b.a = temp;
    }
}

class IntWrapper {
    public int a;
    public IntWrapper(int a) { this.a = a; }
}
```
**Output:**
```
Before swapping, a = 30 and b = 45
After swapping, a = 45 and b = 30
```
Java is always **pass-by-value**, but for objects the value passed is a reference/pointer — so mutating fields *through* that reference works, unlike primitives.

---

### Q29. Merge two sorted arrays

```java
int[] a1 = {1, 7, 9};
int[] a2 = {2, 4, 6, 8, 10};

int[] merged = new int[a1.length + a2.length];
int i = 0, j = 0, k = 0;

while (i < a1.length && j < a2.length) {
    merged[k++] = (a1[i] <= a2[j]) ? a1[i++] : a2[j++];
}
while (i < a1.length) merged[k++] = a1[i++];
while (j < a2.length) merged[k++] = a2[j++];

System.out.println(Arrays.toString(merged));
// [1, 2, 4, 6, 7, 8, 9, 10]
```

---

### Q30. Find all unique strings

**Bugs:** `array.lenght()` → `array.size()`, missing `)` in the `if`.

```java
List<String> array = new ArrayList<>();
List<String> unique = new ArrayList<>();
Set<String> seen = new HashSet<>();

for (String s : array) {
    if (seen.add(s)) {       // add() returns true only if it wasn't already present
        unique.add(s);
    }
}
```

---

### Q31. Find all numbers ending in 1 (e.g., 1, 11, 21...)

**Bug:** `i % 10 == 1` actually checks the **last digit**, not whether the number "starts with 1" — for that you'd check the string representation.

```java
// Numbers ending in 1:
List<Integer> endingIn1 = list.stream()
        .filter(i -> i % 10 == 1)
        .collect(Collectors.toList());

// Numbers starting with 1 (e.g. 1, 10, 15, 100...):
List<Integer> startingWith1 = list.stream()
        .filter(i -> String.valueOf(i).startsWith("1"))
        .collect(Collectors.toList());
```

---

### Q32. REST controller: save & delete employee

**Bug:** `@PathParam` is a JAX-RS annotation, not Spring — use `@PathVariable`.

```java
@RestController
public class EmployeeController {

    @PostMapping("/saveEmployee")
    public Employee saveEmployee(@RequestBody Employee employee) {
        // save logic
        return employee;
    }

    @DeleteMapping("/deleteEmployee/{employeeId}")
    public void deleteEmployee(@PathVariable("employeeId") Integer employeeId) {
        // delete logic
    }
}
```

---

### Q33. Check if a string of brackets is balanced

**Bugs:** `char` has no `.equals()` for string literals like `"("` (compares char vs String — won't compile); the `test.charAt(length-i)` indexing is not how bracket matching works — you need a **stack**, not index arithmetic.

```java
public static boolean isBalanced(String test) {
    Deque<Character> stack = new ArrayDeque<>();
    Map<Character, Character> pairs = Map.of(')', '(', ']', '[', '}', '{');

    for (char ch : test.toCharArray()) {
        if (ch == '(' || ch == '[' || ch == '{') {
            stack.push(ch);
        } else if (pairs.containsKey(ch)) {
            if (stack.isEmpty() || stack.pop() != pairs.get(ch)) {
                return false;
            }
        }
    }
    return stack.isEmpty();
}

// isBalanced("({[]})") -> true
```

---

### Q34. Quick-fire concepts

- **`1.0 / 0.0`** → In Java, floating-point division by zero does **not** throw an exception; it evaluates to `Infinity` (or `-Infinity`, or `NaN` for `0.0/0.0`). Only *integer* division by zero (`1/0`) throws `ArithmeticException`.
- **Overloading ambiguity — `abc(int a, long b)` vs `abc(long a, int b)`:** both are valid overloads (different parameter type order), so this compiles fine as two distinct methods. But calling `abc(5, 10)` (two `int` literals) would be **ambiguous** — the compiler can't decide which parameter to widen — and fails to compile.
- **Quartz** — a Java library for scheduling jobs (cron-like), commonly used for background/recurring tasks in Spring apps (`@Scheduled` in Spring is a lighter built-in alternative).
- **`poll()`** — used in `Queue`/`Deque` implementations to retrieve and remove the head of the queue, returning `null` if empty (unlike `remove()`, which throws an exception on an empty queue).

---
