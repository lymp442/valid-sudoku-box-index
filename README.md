# Valid Sudoku: The Three Rules, the Box-Index Trick, and a One-Pass Solution That Survives Follow-Up Questions

Most people who type "valid sudoku" into a search bar are not looking for a puzzle to play. They've just opened LeetCode 36, read the problem statement twice, and are trying to work out why the answer to the first example is `true` while the second one — which looks almost identical — is `false`.

So let's start there, because that's where the confusion actually lives.

## What "valid" means here (and why it isn't "solvable")

The task is to decide whether a partially filled 9×9 board breaks any of three rules:

1. Each row contains the digits 1–9 with no repeats.
2. Each column contains the digits 1–9 with no repeats.
3. Each of the nine 3×3 sub-boxes contains the digits 1–9 with no repeats.

Empty cells are written as `'.'` and are ignored entirely. The problem statement is explicit about this: only filled cells get validated, and a board can be valid without being solvable. That second point throws people off. A board with a couple of numbers placed is trivially "valid" even if there's no way to finish it, because nothing has been duplicated yet. You are not solving the puzzle. You are checking for contradictions.

That distinction matters more than it sounds, because if you start reaching for backtracking or constraint propagation, you've misread the problem and you'll write 80 lines where 15 would do.

## Why the second example is invalid

The two examples in the problem look nearly the same. The only change in example 2 is that the top-left cell changes from `'5'` to `'8'`, and the expected output flips from `true` to `false`.

The reason is the box rule. The top-left 3×3 box already contains an `8` at position (2,1) — row 2, column 1. Dropping another `8` into the corner cell at (0,0) puts two 8s in the same box. No row or column is duplicated by that single edit, which is exactly why the box check is the one people forget to implement.

If your solution passes example 1 and fails example 2, you almost certainly wrote the row and column checks and skipped or botched the box check. That's the single most common failure mode on this problem.

## Approach 1: Three separate passes

The most readable version loops over the board three times, once for each rule, using a fresh set each time.

python
def isValidSudoku(board):
    # rows
    for r in range(9):
        seen = set()
        for c in range(9):
            if board[r][c] != '.':
                if board[r][c] in seen:
                    return False
                seen.add(board[r][c])

    # columns
    for c in range(9):
        seen = set()
        for r in range(9):
            if board[r][c] != '.':
                if board[r][c] in seen:
                    return False
                seen.add(board[r][c])

    # boxes
    for box_row in range(0, 9, 3):
        for box_col in range(0, 9, 3):
            seen = set()
            for r in range(box_row, box_row + 3):
                for c in range(box_col, box_col + 3):
                    if board[r][c] != '.':
                        if board[r][c] in seen:
                            return False
                        seen.add(board[r][c])

    return True


It's correct, it's easy to explain out loud, and an interviewer will accept it. The downside is that you touch each cell three times and repeat the same check-and-add logic three times. It works, but it invites the follow-up question: "can you do this in one pass?"

## Approach 2: One pass, 27 sets

You can validate all three rules simultaneously if, for every filled cell, you check three separate trackers at once: the row it's in, the column it's in, and the box it's in.

The row and column trackers are trivial. The box tracker is where the interesting part lives.

### The box index formula

Nine boxes, numbered 0 through 8, laid out in reading order. You want a function that takes `(row, col)` and returns the box number.

python
box_index = (row // 3) * 3 + (col // 3)


Why it works: `row // 3` gives you the box row (0, 1, or 2 for the top, middle, and bottom bands). Multiplying by 3 turns that into the starting number of that band — 0, 3, or 6. Then `col // 3` gives the box column within the band — 0, 1, or 2 — and you add it.

Cell (8,8) is the bottom-right corner: `(8//3)*3 + (8//3)` = `2*3 + 2` = box 8. Cell (4,1) is the middle-left box: `(4//3)*3 + (1//3)` = `3 + 0` = box 3. Correct.

python
def isValidSudoku(board):
    rows = [set() for _ in range(9)]
    cols = [set() for _ in range(9)]
    boxes = [set() for _ in range(9)]

    for r in range(9):
        for c in range(9):
            val = board[r][c]
            if val == '.':
                continue

            b = (r // 3) * 3 + (c // 3)

            if val in rows[r] or val in cols[c] or val in boxes[b]:
                return False

            rows[r].add(val)
            cols[c].add(val)
            boxes[b].add(val)

    return True


There's a compact variant some people prefer, which collapses the three trackers into one set of tuples:

python
seen = set()
for r in range(9):
    for c in range(9):
        val = board[r][c]
        if val == '.':
            continue
        keys = ((r, val), (val, c), (r // 3, c // 3, val))
        if any(k in seen for k in keys):
            return False
        seen.update(keys)
return True


It's shorter but slightly harder to read under pressure. If you're writing code on a whiteboard while someone watches, the 27-set version is usually the safer pick — the structure mirrors the three rules one-to-one.

## Approach 3: Bitmasks

Since digits are 1–9, you can replace each set with a 9-bit integer. Bit `d-1` set means digit `d` has been seen in that row, column, or box.

python
def isValidSudoku(board):
    rows = [0] * 9
    cols = [0] * 9
    boxes = [0] * 9

    for r in range(9):
        for c in range(9):
            val = board[r][c]
            if val == '.':
                continue

            bit = 1 << (int(val) - 1)
            b = (r // 3) * 3 + (c // 3)

            if rows[r] & bit or cols[c] & bit or boxes[b] & bit:
                return False

            rows[r] |= bit
            cols[c] |= bit
            boxes[b] |= bit

    return True


This is a nice thing to have in your back pocket for the "can you make it more memory-efficient?" follow-up. It's not faster in any way that matters, and looking up which bit maps to which digit adds a mental step when you're debugging. Bring it out on request, not as your opening move.

## Complexity: everything is O(1), and that's the point

Time complexity is O(1). Space complexity is O(1). Both of those statements are true and both of them sound like a trick, so let's be precise about why.

The board is always 9×9 — 81 cells, fixed by the problem constraints. Whether you do one pass or three, the work is bounded by a constant number of cell visits: 81, or 243 in the three-pass version. Neither grows with input size, because there is no input size to grow. The space is similarly fixed: 27 sets holding at most 9 elements each, or 27 integers.

If you want the "honest" formulation for an interviewer, say O(n²) time with n = 9 and note that n is fixed, so it collapses to constant. Then say the space is O(n²) worst case if you allocate for every box and row, again constant.

What interviewers are actually testing with the complexity question is whether you understand the difference between "constant because the input is bounded" and "constant because I used a hash set." Those are different claims, and only one of them is correct here.

## The mistakes that cost people the interview

A few failure patterns show up again and again:

**Skipping the box check entirely.** Covered above. This is the big one.

**Wrong box index.** Writing `(r // 3) + (c // 3)` gives you a number from 0 to 4 instead of 0 to 8, which silently merges boxes and produces wrong answers that still pass some test cases. If your solution passes example 1 and fails example 2, check this line first.

**Trying to solve the puzzle.** Reaching for backtracking because the word "Sudoku" triggered it. Read the problem: only validate filled cells.

**Forgetting to skip `'.'`.** Every dot gets added to your sets and immediately collides with the next dot, so the function returns `False` on an empty board. An empty board is valid.

**Validating with `1-9` as integers while the input is characters.** The board is `List[List[str]]`, not `List[List[int]]`. `'5' != 5`. If you're using sets, this usually works out anyway because all elements are strings — but the moment you mix in an integer conversion in one branch and not the other, you get a false pass.

**Reset bugs in the three-pass version.** Creating the set outside the loop instead of inside it means digits leak between rows and every board looks invalid.

## How to talk through it out loud

The code is the easy half. The part candidates lose points on is the narration. A workable order:

State the three rules in your own words, then state the one-pass plan: "For every filled cell, I check whether the digit already appears in its row, its column, or its box. If it does, the board is invalid. Otherwise I record it in all three and move on."

Then flag the box index before you write it, because it's the only non-obvious line: "The box number is `(row // 3) * 3 + (col // 3)`, which maps the nine boxes to 0 through 8 in reading order."

Then edge cases. An empty board returns `true`. A board with a single digit returns `true`. A board that's valid but unsolvable still returns `true`. Saying that last one out loud before the interviewer asks is worth doing — it shows you read the note at the bottom of the problem statement.

The likely follow-ups are "can you do it in fewer passes?", "can you reduce the memory?", and "what if the board were n×n instead of 9×9?" That last one is the interesting one, because it's where the complexity answer changes. For a general n×n board with √n × √n boxes, the box index becomes `(r // s) * s + (c // s)` where `s = int(n ** 0.5)`, and you're back to genuine O(n²) time and O(n²) space.

## Where this fits in a bigger prep plan

Valid Sudoku isn't a hard problem. It's an easy one that tests two things: whether you can juggle three conditions in a single pass, and whether you notice the box-index arithmetic. That combination — matrix traversal plus hash-based deduplication — shows up constantly in harder problems, which is why it's worth being able to write from memory rather than re-deriving each time.

This is the same argument behind pattern-based prep platforms. AlgoMonster, built by engineers who worked at Google, organizes its curriculum around roughly 48 reusable coding patterns rather than a flat list of problems, with decision flowcharts for picking the right pattern, text and diagram explanations instead of video, a Speedrun mode for timed review, and an AI assistant you can select text to have explained. If you learn best by reading and coding rather than watching someone else code, the format is built for that. It's worth knowing the tradeoffs upfront: there are no video walkthroughs at all, and reviewers consistently note that the system design and behavioral coverage is thinner than the algorithmic content, so senior candidates typically supplement it.

The numbers on the official subscribe page differ from what plenty of older reviews still quote — the page currently advertises 500+ lessons across 6 courses, 1,000+ illustrations, and 3,000+ editorials, alongside bonus courses on CS foundations and building an LLM. Several blog posts written earlier still cite "48 patterns, 231 lessons," which suggests the catalog has grown since those were published.

## AlgoMonster plans and pricing

AlgoMonster's subscribe page currently lists two paid plans, both shown at promotional pricing, plus free access to a limited sample of the content.

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns and problems — enough to judge whether the text-and-diagram format works for you | $0 | — | [Start with the free content](https://algo.monster?fpr=li69) |
| Pro – Yearly | All patterns, lessons and problems unlocked, unlimited AI assistant usage, unlimited Speedrun, real questions from top companies updated monthly, 500+ lessons across 6 courses, 1,000+ illustrations, 3,000+ editorials, member discount on live bootcamps, 12 months of access | $99/year (shown as $8.25/mo billed annually, $0.28/day) | Billed annually, cancel or upgrade anytime | [Check the yearly plan](https://algo.monster/subscribe?fpr=li69) |
| Pro – Lifetime | Everything in the annual plan, plus lifetime updates — one payment, no renewal | $189 | One-time | [See the lifetime option](https://algo.monster/subscribe?fpr=li69) |

A note on the numbers: the page shows the discounted rate, and at least one partner promotional page tracks the list prices as $198/year and $378 for lifetime, describing the current display as a 50% sale. A third-party partner page also advertises a stacking code (`SCHELLEY`) for an extra ~10% off at checkout. Codes like this change without notice, so check what actually applies at checkout rather than assuming.

There's no monthly billing tier listed on the current page, and no team or enterprise plan — this is built for individual candidates doing self-directed prep.

If you're deciding between the two: the yearly plan makes sense if you have an interview loop in the next few weeks and want the whole catalog open while you work through it. The lifetime plan only really pays off if you expect to come back to this material in two or three years, which most engineers do every time they change jobs. If you're going to finish the curriculum and never open it again, lifetime is the more expensive way to buy the same six weeks of study.

## FAQ

**Can I finish AlgoMonster in a month?** The curriculum is designed around a 4–8 week window at roughly 1–2 hours a day. Whether a month is realistic depends on how much of the earlier material you already know.

**Does AlgoMonster have video explanations?** No. The entire platform is text, code, and interactive diagrams. That's a deliberate choice, and it's the single biggest reason people either love it or bounce off it.

**Is Valid Sudoku actually asked in interviews?** It's a common warm-up or screen question rather than a final-round problem. The value is that it compresses three checks into one pass, which is a skill that carries into matrix and deduplication problems that are asked.

**Do I need a paid plan to solve Valid Sudoku?** No. The solution above is complete, and the problem is free on LeetCode.

## Bottom line

The board is fixed at 9×9, so the whole problem collapses to a single pass with 27 trackers and one piece of arithmetic worth memorizing: `(row // 3) * 3 + (col // 3)`. Get that line right, skip the dots, and remember that validity has nothing to do with solvability, and example 2 stops being a mystery.
