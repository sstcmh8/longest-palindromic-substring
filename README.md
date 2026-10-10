# longest palindromic substring: the three approaches that pass interviews, the one that quietly fails, and how to drill it

Ask for the longest palindromic substring in a string and most candidates reach for a neat trick: reverse the string, find the longest common substring, done. It's wrong, and the fact that it's *plausibly* wrong is a big part of why this problem still shows up in screening rounds.

What follows is the full picture: what the problem actually asks, why the popular shortcut breaks, how each real solution works, where they fall apart under pressure, and how to practice it so the center-expansion template is something you write from memory in four minutes instead of reconstructing from scratch.

## What the problem is actually asking

Given a string `s`, return the longest substring of `s` that reads the same forwards and backwards.

On LeetCode it's problem 5, and the constraints are tight enough to matter: `1 <= s.length <= 1000`, and `s` consists of digits and English letters only. Two examples do most of the teaching:

- `"babad"` → `"bab"` (or `"aba"` — both are accepted)
- `"cbbd"` → `"bb"`

The second one is the important one. `"cbbd"` has no palindromic substring longer than two characters, which means any solution that only checks odd-length palindromes around single characters returns `"b"` and fails. There's also `"ac"` → `"a"`, which catches the opposite mistake of assuming every input has a palindrome of length ≥ 2.

One distinction worth stating in an interview before you write anything: a **substring is contiguous**, a **subsequence is not**. Longest Palindromic Substring (this problem) and Longest Palindromic Subsequence (problem 516) have similar names, genuinely different answers, and different algorithms.

## Why "reverse it and find the longest common substring" is wrong

The reasoning sounds right. A palindrome reads the same in both directions, so it should appear identically in `s` and in `reverse(s)`. Find the longest common substring of those two strings and you're done.

The flaw: a shared substring between `s` and its reversal doesn't have to be a palindrome. The match can come from characters that overlap in a way the symmetry check never verifies.

Concrete counterexample. Take `s = "abacdfgdcaba"`. Its reverse is `"abacdgfdcaba"`. The longest common substring is `"abacd"`, five characters. But `"abacd"` isn't a palindrome. The real answer is `"aba"`, three characters.

A second classic case: `s = "aacxycaa"`. Reverse is `"aacyxcaa"`. Longest common substring is `"aac"`, again not a palindrome. Correct answer: `"aa"`.

The lesson transfers even though the approach doesn't. The instinct to *reduce* an unfamiliar problem to a known one is exactly what interviewers want to see — you just have to notice when the reduction loses information. Mention the trap, explain why it fails, and you've turned a wrong answer into evidence that you reason about correctness rather than matching shapes.

## Brute force: O(n³), say it in one sentence and move on

There are O(n²) substrings. Checking whether one is a palindrome costs O(n). Multiply and you get O(n³) time and O(1) extra space.

It's correct, and at `n = 1000` the worst case is on the order of a billion character comparisons. That's too slow to submit and too slow to be an answer. But naming the baseline out loud is useful: you've established a correctness anchor, and everything after it is a visible improvement rather than a guess.

## Dynamic programming: O(n²) time, O(n²) space

Define `dp[i][j]` as *true* when the substring from index `i` to `j` is a palindrome. The recurrence falls out of a palindrome's structure — strip the matching ends and what remains is still a palindrome:


dp[i][j] = (s[i] == s[j]) and (j - i < 3 or dp[i + 1][j - 1])


The `j - i < 3` guard handles every base case at once. Length 1: single characters are always palindromes. Length 2: two equal characters, no interior to check. Length 3: outer characters equal, the middle character is irrelevant. Beyond that you need the inner substring to already be known.

python
def longestPalindrome(s):
    n = len(s)
    if n < 2:
        return s
    dp = [[False] * n for _ in range(n)]
    start, best = 0, 1
    for i in range(n):
        dp[i][i] = True
    for i in range(n - 1, -1, -1):
        for j in range(i + 1, n):
            if s[i] == s[j] and (j - i < 3 or dp[i + 1][j - 1]):
                dp[i][j] = True
                if j - i + 1 > best:
                    start, best = i, j - i + 1
    return s[start:start + best]


The loop order matters and it's a common point of failure. `dp[i][j]` depends on `dp[i + 1][j - 1]`, which sits one row down and one column left, so `i` has to run **downward** from `n - 1` to `0`. Iterating `i` upward reads unfilled cells and quietly returns wrong answers on longer inputs while still passing the two examples.

Cost at `n = 1000`: a million booleans, roughly a megabyte in Python after object overhead. LeetCode accepts it. An interviewer will too, and then will usually ask whether you need the table at all — which is your cue for the next approach.

## Expand around center: O(n²) time, O(1) space

Every palindrome has a center. For a string of length `n` there are `2n - 1` possible centers: `n` single characters (odd-length palindromes) and `n - 1` gaps between adjacent characters (even-length palindromes). Walk outward from each one while the characters match, and keep the longest result.

python
def longestPalindrome(s):
    def expand(l, r):
        while l >= 0 and r < len(s) and s[l] == s[r]:
            l -= 1
            r += 1
        return s[l + 1:r]

    best = ""
    for i in range(len(s)):
        odd = expand(i, i)
        even = expand(i, i + 1)
        for candidate in (odd, even):
            if len(candidate) > len(best):
                best = candidate
    return best


Two details carry the whole solution. First, calling `expand` twice per index — once with `(i, i)`, once with `(i, i + 1)` — is what makes even-length palindromes work. Skipping the second call is the single most common bug in this problem, and `"cbbd"` is the test case that catches it. Second, the slice is `s[l + 1:r]`, not `s[l:r]`, because the loop always overshoots by one step before it stops.

Worst case is still O(n²) — a string of a thousand identical characters makes every expansion run to the boundary — but extra space is O(1), the code fits on a screen, and it's trivial to explain. This is the answer to give by default.

There's a micro-optimization worth knowing if the interviewer pushes on constant factors: when the character at the current center is equal to its neighbors, you can skip ahead and group the run of identical characters into a single central block before expanding. It doesn't change the complexity, but it removes the redundancy that makes all-same-character inputs slow.

## Manacher's algorithm: O(n) time, O(n) space

Glenn Manacher published this in 1975, and it remains the only linear-time solution for the general problem. Two ideas make it work.

**Separator insertion.** Insert a character between every pair of characters and at both ends: `"babad"` becomes `"#b#a#b#a#d#"`. Every palindrome in the original string — odd or even — now corresponds to an odd-length palindrome in the transformed string, centered on either an original character or a separator. One case instead of two.

**Mirror reuse.** Track the palindrome you've already found that reaches furthest right, described by its center `c` and right boundary `r`. For a new index `i` inside that boundary, the palindrome centered at `i` mirrors the one at `2c - i`, so you can start with an inherited radius instead of zero:

python
def longestPalindrome(s):
    t = "#" + "#".join(s) + "#"
    n = len(t)
    p = [0] * n          # p[i] = radius of the palindrome centered at i
    center = right = 0

    for i in range(n):
        if i < right:
            p[i] = min(right - i, p[2 * center - i])
        while (i - p[i] - 1 >= 0 and i + p[i] + 1 < n
               and t[i - p[i] - 1] == t[i + p[i] + 1]):
            p[i] += 1
        if i + p[i] > right:
            center, right = i, i + p[i]

    length, center = max((p[i], i) for i in range(n))
    start = (center - length) // 2
    return s[start:start + length]


The `min(right - i, p[2 * center - i])` line is the trick. You take the mirrored radius, but you can only trust it as far as the current boundary extends — beyond `right`, the mirror gives no information and the naive expansion has to run. That cap is what keeps it correct.

The linear bound comes from the expansion loop: `right` only ever moves forward, never backward, so the total number of successful expansions across the whole algorithm is at most `n`. Everything else is O(1) per index.

Should you learn it? It comes up in competitive programming and in string-heavy rounds, and it's a good answer to "can you do better than O(n²)?" — but it is rarely *required* at a typical software engineering interview. The realistic failure mode isn't missing Manacher. It's getting flustered by the question and failing to produce the clean expand-around-center solution you actually know. Have the O(n²) answer cold first.

## The four approaches side by side

| Approach | Time | Extra space | Where it belongs |
| --- | --- | --- | --- |
| Brute force | O(n³) | O(1) | Mention it, don't write it |
| DP table | O(n²) | O(n²) | Reasonable baseline you can code safely |
| Expand around center | O(n²) | O(1) | The default interview answer |
| Manacher's | O(n) | O(n) | When asked for the asymptotic best |

## The bugs and follow-ups that actually decide the outcome

- **Only expanding from single characters.** Fails `"cbbd"`. Always call expand twice per index.
- **Returning a length instead of a string.** Read the problem statement; LeetCode 5 wants the substring.
- **Off-by-one in the slice.** The expansion loop exits with `l` and `r` one step past the boundary, so the answer is `s[l + 1:r]`. A lot of candidate solutions are correct except for this.
- **Storing substrings inside the inner loop.** Track `start` and `maxLength` as integers and slice once at the end. It's not just style — repeated slicing can turn an O(n²) algorithm into something noticeably slower.
- **Not handling the degenerate inputs.** `""`, `"a"`, and `"ac"` should all be answered without special-case crashes.
- **Reciting Manacher without being able to justify the mirror line.** If you write it, be ready to explain why the inherited radius is capped at `right - i`.

Expect at least one follow-up. Common ones: *count* all palindromic substrings (LeetCode 647, same center expansion, sum the radii), longest palindromic *subsequence* (516, genuinely different), palindrome partitioning (131), shortest palindrome by prepending characters (214), and "what if the input arrives as a stream and you can't reverse it." The last one is a design conversation, not a coding exercise, and the honest answer is that the O(n²) space table is off the table.

## Problems to drill alongside this one

Solving LeetCode 5 once teaches you the recurrence. Solving it next to these teaches you the pattern:

- **647. Palindromic Substrings** — the same expansion, counting instead of maximizing
- **516. Longest Palindromic Subsequence** — the contrast case that keeps substring and subsequence straight in your head
- **5 with follow-ups: 131, 132, 214** — partitioning and construction, where you build on palindrome detection
- **125 and 680** — validating and repairing single palindromes, useful warm-ups

Four or five well-chosen problems worth of pattern recognition beats fifty problems of random grinding, which is the whole premise behind how AlgoMonster organizes its curriculum: explicit reusable patterns with templates and decision flowcharts rather than a bank of problems to work through by feel. It includes a written editorial for LeetCode 5 with Python, Java, and C++ solutions, plus the surrounding string and dynamic programming material, so the palindrome problems sit inside a framework instead of floating on their own. 👉 [See how AlgoMonster walks through Longest Palindromic Substring step by step](https://algo.monster/subscribe?fpr=li69)

Worth being straight about the tradeoffs. AlgoMonster is text and diagrams, with no video walkthroughs, so if you learn by watching someone solve a problem live, this format will feel like work. Its problem count is far smaller than LeetCode's, on purpose. System design content is introductory — not enough for a senior-level design round. It assumes you can already write working code in one language. What it's built for is a specific gap: you can implement things fine, but you spend the first fifteen minutes of a forty-five minute round trying approaches that don't fit.

## AlgoMonster plans and current pricing

Access is all-or-nothing across patterns, lessons, and problems; the tiers differ by billing period, not by content. Unlimited AI assistant usage and Speedrun are included at every paid level. Prices below are what the subscribe page showed when this was written — the platform runs frequent promotions, so treat the number you see at checkout as the real one.

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to a sample of patterns and problems | $0 | — | [Start with the free tier](https://algo.monster/subscribe?fpr=li69) |
| Pro Monthly | All patterns, lessons and problems, unlimited AI assistant, unlimited Speedrun | $18.99/mo | Billed monthly | [Check the monthly plan](https://algo.monster/subscribe?fpr=li69) |
| Pro Yearly | Everything in Monthly, plus 12 months of full access | $8.25/mo, $99/year | Billed annually | [Get the yearly plan](https://algo.monster/subscribe?fpr=li69) |
| Pro Lifetime | Everything in Yearly, plus lifetime updates and an internal referral to a top tech company for qualified candidates | $189 | One-time | [See the lifetime plan](https://algo.monster/subscribe?fpr=li69) |

The yearly option works out to roughly $8.25 a month — about the price of a coffee — which puts it under LeetCode Premium's annual price if you're comparing prep budgets. Lifetime breaks even against two years of annual billing, so it's the better pick only if you expect to interview again. Someone with a loop scheduled six weeks out gets nothing extra from it.

If you read reviews claiming AlgoMonster has no monthly plan, note that the plan lineup has changed over time. The subscribe page currently lists monthly, yearly, and lifetime. Check what's live rather than trusting a cached comparison table — including the one above. 👉 [See current AlgoMonster promo pricing](https://algo.monster?fpr=li69)

## Which approach to give, in one line each

Brute force proves you understand the problem. Dynamic programming proves you can turn an observation into a recurrence. Expand around center is the answer you actually write — O(n²) time, O(1) space, no table, no bugs waiting in the loop order. Manacher's is the one you mention when someone asks whether the quadratic bound is the end of the story.

Solve `"babad"`, `"cbbd"`, `"ac"`, and a string of five hundred identical characters. If all four pass and you can explain, unprompted, why reversing the string and taking the longest common substring doesn't work, you know this problem.
