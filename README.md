# find median from data stream: the two-heap balance rule explained, with working Python, Java, and C++ code, plus the follow-ups interviewers actually ask

The problem shows up as LeetCode 295, "Find Median from Data Stream," and it's a Hard for a specific reason: the interface is two lines long, but almost everyone gets the invariant wrong on the first attempt.

You're asked to design a class with two methods:


addNum(int num)   // add a number from the stream
findMedian()      // return the median of everything added so far


Constraints, straight from the problem statement: values sit in the range −10⁵ to 10⁵, there are at most 5 × 10⁴ calls to `addNum` and `findMedian` combined, and `findMedian` is never called before at least one number has been added.

The example everyone knows:


addNum(1)          → [1]           median = 1
addNum(2)          → [1, 2]        median = 1.5
findMedian()       → 1.5
addNum(3)          → [1, 2, 3]     median = 2
findMedian()       → 2


Two things to notice. The median is a *running* value, not a final one — every insertion can shift it. And when the count is even, the answer is the average of the two middle values, which is why it's a `double` and not an `int`.

## The obvious solutions, and the exact line where they break

Keep everything in an array and sort inside `findMedian`. Correct, and it dies on the call budget: sorting a growing list on each of 50,000 queries is O(n log n) work repeated n times.

Insert into a sorted list instead — `bisect.insort` in Python, `List.add(index, value)` in Java. Now `findMedian` is O(1) index arithmetic, but each insertion shifts elements, so `addNum` is O(n). With 50,000 calls, an O(n) insert is roughly a billion element moves in the worst case. Too slow.

The insight that unlocks the problem is that **you don't need full sorted order.** You need exactly two values:

- the largest number in the lower half
- the smallest number in the upper half

Everything else in either half is irrelevant. A **max-heap** hands you the largest element of a group in O(1). A **min-heap** hands you the smallest. So split the stream into a lower half and an upper half, and keep the median sitting on the boundary between them.

## Two heaps, one invariant

Store the smaller half in a max-heap (`lower`) and the larger half in a min-heap (`upper`). Two rules keep it honest:

1. Every value in `lower` is ≤ every value in `upper`.
2. `len(lower)` is either equal to `len(upper)` or exactly one bigger.

If both rules hold, the median is one of two things. Odd count — `lower` holds the extra element and its top is the median. Even count — the median is the average of the two tops.


lower (max-heap)        upper (min-heap)
[3, 2, 1]               [5, 7, 8]
    ^ largest of small      ^ smallest of large
    median boundary sits here


The elegant version of the insertion rewrite is three heap operations and no branching on values:

1. Push the new number onto `lower`.
2. Pop `lower`'s maximum and push it onto `upper`.
3. If `upper` is now bigger than `lower`, pop `upper`'s minimum back onto `lower`.

The middle step is what guarantees rule 1. The number you just pushed either belongs in the lower half or not, but either way the biggest element of `lower` gets moved across, and a swap corrects the cases where it shouldn't have gone. That's why the code is short enough to write from memory — and why the version with `if (num <= maxHeap.top())` branches and manual rebalancing inside each branch is the one that tends to break under interview pressure.

## Working code in three languages

Python:

python
import heapq

class MedianFinder:
    def __init__(self):
        self.lower = []   # max-heap, stored negated
        self.upper = []   # min-heap

    def addNum(self, num: int) -> float:
        heapq.heappush(self.lower, -num)
        # move the largest of the small half across the boundary
        heapq.heappush(self.upper, -heapq.heappop(self.lower))
        # keep lower >= upper so the extra element lives in lower
        if len(self.upper) > len(self.lower):
            heapq.heappush(self.lower, -heapq.heappop(self.upper))

    def findMedian(self) -> float:
        if len(self.lower) > len(self.upper):
            return float(-self.lower[0])
        return (-self.lower[0] + self.upper[0]) / 2.0


Java:

java
class MedianFinder {
    private final PriorityQueue<Integer> lower =
        new PriorityQueue<>(Comparator.reverseOrder());
    private final PriorityQueue<Integer> upper = new PriorityQueue<>();

    public void addNum(int num) {
        lower.offer(num);
        upper.offer(lower.poll());
        if (upper.size() > lower.size()) lower.offer(upper.poll());
    }

    public double findMedian() {
        if (lower.size() > upper.size()) return lower.peek();
        return (lower.peek() + upper.peek()) / 2.0;
    }
}


C++:

cpp
class MedianFinder {
    priority_queue<int> lower;                              // max-heap
    priority_queue<int, vector<int>, greater<int>> upper;   // min-heap
public:
    void addNum(int num) {
        lower.push(num);
        upper.push(lower.top()); lower.pop();
        if (upper.size() > lower.size()) { lower.push(upper.top()); upper.pop(); }
    }
    double findMedian() {
        if (lower.size() > upper.size()) return lower.top();
        return (lower.top() + upper.top()) / 2.0;
    }
};


Python has no max-heap in the standard library, so the lower half stores negated values. That's the only language quirk worth memorizing. Note the `float()` cast and the `/ 2.0` — in Python 2-era habits and in C++, integer division silently truncates the median, and that's a real test-case failure, not a style issue.

## A trace, because the balancing rule is easier to trust than to believe

Stream: 5, 3, 8, 1.


add 5: lower=[5]                  upper=[]
       move max across → lower=[] upper=[5] → upper bigger → move back
       lower=[5]                  upper=[]        median = 5

add 3: lower=[5,3]                upper=[]
       move max across → lower=[3] upper=[5]
       sizes equal                               median = (3+5)/2 = 4.0

add 8: lower=[3,8] → max moves → lower=[3] upper=[5,8] → upper bigger → move back
       lower=[5,3]                upper=[8]        median = 5

add 1: lower=[5,3,1] → max moves → lower=[3,1] upper=[5,8]
       sizes equal                               median = (3+5)/2 = 4.0


Sorted so far: [1, 3, 5, 8]. Median is (3+5)/2 = 4.0. Matches.

## Complexity, next to the approaches you'd be compared against

| Approach | addNum | findMedian | Space | Notes |
| --- | --- | --- | --- | --- |
| Sort on every query | O(1) push | O(n log n) | O(n) | Simplest to write, fails the call budget |
| Sorted array + binary search insert | O(n) | O(1) | O(n) | Shifting dominates |
| Ordered multiset / `SortedList` | O(log n) | O(1) | O(n) | Needs `sortedcontainers` in Python or `TreeMap`-style bookkeeping in Java |
| **Two heaps** | **O(log n)** | **O(1)** | **O(n)** | Standard-library only, in every language |

Each `addNum` performs a constant number of heap operations on heaps of size at most n, so the log factor is small. `findMedian` touches at most two array slots.

## The mistakes that actually cost people this problem

Getting the problem wrong rarely looks like "didn't know about heaps." It looks like one of these:

- **Balancing backwards.** Letting `upper` be the heap that's allowed to grow by one. It works until the odd-count case, where `findMedian` returns the wrong top. Pick one side (lower) and enforce it every time.
- **Forgetting the value-ordering rule.** Sizes balanced but elements interleaved between the halves gives a *believable* wrong median — often off by one position, which is the worst kind of bug to debug.
- **Integer division.** `(maxHeap.top() + minHeap.top()) / 2` in C++ or Java with `int` operands truncates. `[-1, -2]` gives you an expected `-1.5` and an actual `-1`.
- **Peeking an empty heap.** If you allow either heap to be empty when the count is even, both must be non-empty at that moment — which the invariant guarantees. Branching on emptiness instead of on sizes is where null-dereference crashes come from.
- **Negative values.** They're inside the stated range, and a solution that "works" on positives can still pass sample tests while failing on negatives if the ordering rule has a sign error.
- **Negative numbers plus Python negation.** `-lower[0]` on a negated heap is easy to get backwards in the even case. Write the trace above by hand once and it stops being a guessing game.

## Follow-ups interviewers add after your code passes

The base problem is a warm-up for variants. Knowing the shape of these is worth more than another 10 minutes of code-polishing:

**"What if 99% of the values are between 0 and 100?"** If the value range is small and known, you don't need heaps at all. Keep a count array indexed by value, plus the running total. `addNum` is O(1); `findMedian` walks bucket indices, which is O(range) — trivial when the range is 101. This is the classic "use the constraint, not the general data structure" follow-up.

**"What if the stream doesn't fit in memory?"** Heaps want O(n) storage. Options depend on what the interviewer actually wants: bounded-range counting still works if the values are bounded, or you accept approximate answers from a quantile sketch.

**"What if the window moves?"** Sliding window median (LeetCode 480) keeps the same two-heap core and adds lazy deletion — mark removed values, then pop them from the heap tops when they surface. Same invariant, harder bookkeeping.

**"Where does the median come from in the k-th largest variant?"** Different problem, same idea: heaps on a boundary rather than a fully ordered structure.

## Learning this as a pattern instead of a one-off

Here's the thing about "find median from data stream": you can memorize that code in twenty minutes and still fail the variant, because the transferable skill isn't the code — it's recognizing that *a running statistic over a stream with a boundary condition* means two heaps, and knowing which side gets the extra element.

That's the gap AlgoMonster is built around. It's a structured interview-prep platform from ex-Google engineers that teaches 48 named coding patterns with templates and decision flowcharts rather than dumping a problem bank on you, and the heap work sits inside that pattern framework. The site publishes a free editorial for this exact problem at its LeetCode 295 page, using the same two-heap structure with `heappushpop` — worth reading as a second implementation of the same invariant.

👉 [Read the two-heap walkthrough for LeetCode 295](https://algo.monster/liteproblems/295?fpr=li69)

## What you actually pay if you want the full pattern library

The free tier gives you limited access to a subset of patterns and problems — enough to see whether the text-and-diagram teaching style clicks for you. Full access is one of two paid plans, and there's no monthly option.

| Plan | What's included | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns and problems | $0 | — | [Start free](https://algo.monster?fpr=li69) |
| Pro Yearly | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; real interview questions from top tech companies, updated monthly; 500+ lessons across 6 courses, 1,000+ illustrations, 3,000+ editorials; member discount on live bootcamps | $99/year ($8.25/mo billed annually), discounted from $198 | 12 months of full access | [Get Pro Yearly](https://algo.monster/subscribe?fpr=li69) |
| Pro Lifetime | Everything in the yearly plan plus lifetime updates — all current and future content at no extra cost | $189 one-time, discounted from $378 | Billed once | [Get Pro Lifetime](https://algo.monster/subscribe?fpr=li69) |

Two things worth knowing before you buy. The subscribe page rotates its promotional pricing — during deeper sales it has displayed the yearly plan at $69.99 and lifetime at $89, so the number you see depends on the promotion running when you visit. And subscriptions include the Foundation Course and the Build an LLM Course as bonus content, while some specialized courses developed with external partners (the Quant Interview Prep course, for example) are sold separately.

If your interview is six weeks out and your weak spot is pattern selection — recognizing within two minutes that a problem is a two-heap problem — the yearly plan covers that window. If you expect to interview again in two or three years, the lifetime plan is a one-time cost instead of a renewal.

👉 [Check the current AlgoMonster pricing and promotion](https://algo.monster/subscribe?fpr=li69)

## Quick answers to the questions people actually search

**Is there a way to do this without heaps?** Yes, if the value range is bounded: a counting array or a Fenwick tree both get you O(log n) or O(1) updates with exact medians. For unbounded values, heaps are the standard answer.

**What's the time complexity again?** `addNum` is O(log n) — a constant number of heap operations. `findMedian` is O(1) — at most two top-of-heap reads. Space is O(n) because every element is stored exactly once, split across the two heaps.

**Why does the lower half get the extra element on odd counts?** Because then `lower[0]` — the largest of the smaller half — *is* the median when the count is odd, with no averaging and no extra branch. You could equally keep the extra on the upper side; you just have to be consistent.

**Do I need `findMedian` to handle an empty structure?** The constraints say no — it's only called after at least one `addNum`. Defensive code for the empty case is fine, but don't let it hide a broken invariant.

**Does the same code work for floats?** Not for the LeetCode version, which takes `int`s. The pattern generalizes, but a max-heap on floats plus averaging arithmetic needs care with precision.

The reason this problem keeps appearing in interviews is that it tests four things at once: whether you reach for the right structure, whether you can state an invariant out loud, whether you can keep that invariant under insertions, and whether your complexity analysis matches the code you wrote. Get the balance rule into your hands — push, move across, correct — and it stops being a Hard you hope doesn't come up.
