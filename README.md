# evaluate reverse polish notation: the stack walkthrough, the operand‑order trap, and what interviewers ask next

If you searched this phrase, you probably have one of two things in front of you: LeetCode 150 open in a tab, or a take‑home assessment with a token list like `["2","1","+","3","*"]` and a submit button that does not care how you feel. Either way, the good news is that this is one of the few problems where the algorithm is genuinely short. The bad news is that the short algorithm hides two or three details that quietly break a surprising number of submissions — and those same details are what a live interviewer pokes at when you say you're done.

## What "reverse Polish notation" actually is

Reverse Polish Notation — RPN, also called postfix notation — puts the operator *after* its operands. Normal arithmetic (`2 + 1`) is infix: the operator sits between them. In postfix the same expression is `2 1 +`.

It's named after Jan Łukasiewicz, the Polish logician who introduced the parenthesis‑free notation in the 1920s (the "reverse" version, operands first, came from later computer science work). The reason it survived into engineering is practical: with operator‑after‑operand order, you never need parentheses and you never need precedence rules. `(1 + 2) * (3 + 4)` becomes `1 2 + 3 4 + *`, which reads unambiguously left to right.

You have probably already used it without knowing:

- **HP calculators.** Since the 1960s, HP's RPN calculators make you enter `2`, `Enter`, `1`, `+` instead of `2 + 1`.
- **Stack‑based languages.** Forth, PostScript, and Racket's stack machine all evaluate postfix.
- **The JVM.** Java bytecode is postfix — `iload_1, iload_2, iadd` is exactly "push, push, add".

So this isn't a LeetCode invention. It's the notation stack machines have used for decades, which is precisely why the stack solution feels inevitable once you see it.

## The algorithm, in five lines

The whole thing:

1. Create an empty stack.
2. Walk the tokens left to right.
3. If the token is a number, push it.
4. If it's an operator, pop two values, apply the operator, push the result.
5. After the loop, the stack holds exactly one value. Return it.

Why a stack and not something else? Because postfix is written so that operands are always resolved before the operator that consumes them. The most recently pushed value is always the one the next operator needs. A stack — last in, first out — matches that structure natively. If you tried this with a queue, you'd be fighting the notation.

Time and space are both **O(n)**: one pass, and in the worst case (all numbers, no operators) the stack holds every token.

## Tracing the three LeetCode examples

`tokens = ["2","1","+","3","*"]` → **9**

Push `2`. Push `1`. Hit `+`: pop `1` and `2`, compute `2 + 1 = 3`, push `3`. Push the literal `3` — the stack is now `[3, 3]`. Hit `*`: pop `3` and `3`, compute `9`. Answer: `9`.

`tokens = ["4","13","5","/","+"]` → **6**

Push `4`, `13`, `5`. Hit `/`: pop `5`, then pop `13`. Compute `13 / 5`. Because LeetCode specifies truncation toward zero for integer division, this is `2`, not `2.6`. Push `2`. Hit `+`: pop `2`, pop `4`, compute `6`. Answer: `6`. This is the example that breaks naive solutions, and we'll come back to it.

`tokens = ["10","6","9","3","+","-11","*","/","*","17","+","5","+"]` → **22**

This one looks intimidating and isn't. Working it through: `9 + 3 = 12`; `12 * -11 = -132`; `6 / -132 = 0` (truncated toward zero — this is the step where floor division gives you `-1` and the whole answer goes wrong); `10 * 0 = 0`; `0 + 17 = 17`; `17 + 5 = 22`.

Notice how much of the difficulty is integer semantics rather than data structures.

## The mistakes that actually fail submissions

### Operand order for `-` and `/`

This is the classic. A lot of people write:

python
if t == "-":
    stack.append(stack.pop() - stack.pop())


Then they move on. That line is wrong, because `stack.pop()` on the left is evaluated first, so you're actually computing `second - first`. Addition and multiplication are commutative and hide the bug; subtraction and division expose it immediately.

Pop into named variables, once, in order:

python
b = stack.pop()   # right operand — popped first
a = stack.pop()   # left operand
# now: a - b, a / b


A useful mental image: the stack top is the *right* operand, and the value below it is the *left* operand. LeetCode's own editorial puts it the same way — the first pop is the right operand.

### Truncation toward zero, not toward negative infinity

`-7 / 2` should be `-3` in this problem. If you use Python's `//` on a negative numerator you get `-4`. If you use JavaScript's `Math.floor()` you also get `-4`. Both are wrong here.

- Python: `int(a / b)` or `math.trunc(a / b)`
- JavaScript: `Math.trunc(a / b)`
- Java and C++: integer `/` already truncates toward zero, so `a / b` is correct as written

That single character difference between `Math.floor` and `Math.trunc` is worth memorizing, because it turns a "correct algorithm" into a failed test case on example 3.

### Treating negative numbers as operators

`"-11"` is a number, not a minus sign followed by `11`. If you test `if token[0] == "-"` you'll push an operator token into `parseInt` and blow up.

The fix is to compare the whole token: `token == "-"` or `ops.has(token)`, never `token[0]`. Some solutions add a `len(token) == 1` check as a belt‑and‑braces guard, which is fine, but the set comparison is cleaner and harder to get wrong.

### Overflow assumptions

LeetCode guarantees the answer and all intermediate results fit in a 32‑bit integer, with operands in `[-200, 200]` and up to 10⁴ tokens. That's why `int` in Java is safe. If you're solving a variant without that guarantee — say, a streaming calculator from real interview material — you'd want `long` or `bigint`. Knowing *why* `int` is safe here is a small thing that reads as competence.

## A solution you can actually submit

**Python**

python
class Solution:
    def evalRPN(self, tokens: list[str]) -> int:
        stack = []
        for t in tokens:
            if t in ("+", "-", "*", "/"):
                b = stack.pop()
                a = stack.pop()
                if t == "+":
                    stack.append(a + b)
                elif t == "-":
                    stack.append(a - b)
                elif t == "*":
                    stack.append(a * b)
                else:
                    stack.append(int(a / b))   # truncate toward zero
            else:
                stack.append(int(t))
        return stack[-1]


**Java**

java
class Solution {
    public int evalRPN(String[] tokens) {
        Deque<Integer> stack = new ArrayDeque<>();
        for (String t : tokens) {
            switch (t) {
                case "+" -> stack.push(stack.pop() + stack.pop());
                case "*" -> stack.push(stack.pop() * stack.pop());
                case "-" -> { int b = stack.pop(), a = stack.pop(); stack.push(a - b); }
                case "/" -> { int b = stack.pop(), a = stack.pop(); stack.push(a / b); }
                default  -> stack.push(Integer.parseInt(t));
            }
        }
        return stack.pop();
    }
}


**JavaScript**

js
var evalRPN = function (tokens) {
  const stack = [];
  const ops = new Set(["+", "-", "*", "/"]);
  for (const t of tokens) {
    if (!ops.has(t)) { stack.push(Number(t)); continue; }
    const b = stack.pop(), a = stack.pop();
    if (t === "+") stack.push(a + b);
    else if (t === "-") stack.push(a - b);
    else if (t === "*") stack.push(a * b);
    else stack.push(Math.trunc(a / b));
  }
  return stack.pop();
};


Note the asymmetry in how each language handles division. That's not incidental trivia — an interviewer asking "would this behave differently in another language?" is essentially inviting you to say `-7 / 2` is `-3` in Java and `-4` if you write `//` in Python.

## The follow‑ups that decide the round

Getting the RPN solution right takes about ten minutes. What fills the remaining half hour is usually this family of questions:

- **Convert infix to postfix.** The shunting‑yard algorithm — output queue, operator stack, precedence comparisons. Same stack reflex, different bookkeeping.
- **Evaluate an infix expression with `+ - * /` and parentheses.** That's Basic Calculator I, II, and III on LeetCode. Notably harder, because now precedence and parentheses are your problem instead of the notation's.
- **Build an expression tree from RPN, then evaluate it.** Postfix maps onto post‑order traversal almost one to one; this is a common variation at companies that care about compilers or query engines.
- **Handle a stream instead of an array.** Now the stack is state you keep across calls, and the follow‑up turns into "what's the space complexity if the input never ends?"
- **Batch evaluation with caching.** A short step from here to "how would you memoize repeated subexpressions?"

The connective tissue is pattern recognition: if the problem involves postfix, prefix, or expression evaluation, the answer is almost always a stack. Recognizing "this is the stack pattern" in under two minutes is the skill these rounds actually test — the code is downstream of that.

## Where a pattern‑based curriculum fits — and its real limits

That last point is exactly what platforms like AlgoMonster sell. AlgoMonster is a coding‑interview prep site built by ex‑Google engineers and competitive programmers, and instead of a problem bank it's organized around explicit coding patterns with reusable templates, decision flowcharts ("is the input sorted? is it a contiguous subarray?"), interactive code editors with an embedded AI assistant that debugs your submission, and a timed Speedrun mode for review. The RPN problem lives under its stack material, right next to the infix/postfix conversion and monotonic stack variants, which is a reasonable way to learn it if you've been solving problems one at a time and forgetting the approach a week later.

The honest limitations, based on what reviews consistently flag and what the platform itself doesn't claim to cover:

- **No video content at all.** Everything is text and interactive diagrams. Roughly 1,000 illustrations across the current course list, but zero walkthroughs.
- **System design coverage is introductory.** Fine for a phone screen, not enough for a senior loop with a dedicated design round.
- **No live interview simulation.** Speedrun adds a clock; it doesn't add a senior engineer asking why you chose that approach.
- **The problem bank is small** compared to LeetCode's thousands. Pattern frameworks plus volume practice is the combination most candidates end up using, not one or the other.

## AlgoMonster plans and current pricing

AlgoMonster has no free trial beyond its free tier, and — as of the official subscribe page when we checked — four options. Prices on that page move with promotions fairly often, so treat these as the figures the page displayed rather than a permanent rate card:

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns and problems | $0 | — | [Start on the free tier](https://algo.monster?fpr=li69) |
| Pro Monthly | All patterns, lessons and problems, unlimited AI assistant, unlimited Speedrun, company question sets updated monthly | $18.99 | Billed monthly, cancel anytime | [Check the monthly plan](https://algo.monster?fpr=li69) |
| Pro Yearly | Everything in Pro, 12 months of access, free content updates, member discount on live bootcamps | $69.99–$99 per year depending on the promotion in effect (about $5.84–$8.25/month) | Billed annually | [See the annual price](https://algo.monster?fpr=li69) |
| Pro Lifetime | Everything in the annual plan plus lifetime updates, one payment | $89–$189 one‑off depending on the current promotion | Billed once | [Check the lifetime price](https://algo.monster?fpr=li69) |

A few things worth knowing before you pick:

- The Pro tiers unlock the full curriculum, the AI assistant without usage limits, and Speedrun. The current course list runs to 500+ lessons across six courses with 3,000+ editorials. Some partner‑developed courses, such as Quant Interview Prep, are sold separately.
- Third‑party reviews quote older, higher lifetime prices than the subscribe page currently shows. This is a platform that runs frequent sales rather than a fixed list price, so the number you see at checkout is the number that matters.
- If your interview is six weeks out, the annual plan covers it and the lifetime plan is harder to justify unless you expect to job‑hop again. If you're the type who re‑preps every couple of years, lifetime breaks even against roughly two annual renewals.

If your actual problem is "I can solve RPN but I freeze on unfamiliar problems," the structured pattern track is a better fit than another 200 random problems. If your problem is "I need to write code under a clock with someone watching," AlgoMonster won't fix that part — pair it with mock interviews.

## A practice sequence that works

Don't stop at one accepted submission. The problem is easy; the family isn't.

1. Solve LeetCode 150 with a stack, in your main interview language, without looking anything up.
2. Re‑solve it and deliberately write the `-` and `/` cases with named operands, so the ordering becomes muscle memory.
3. Write the infix‑to‑postfix converter (shunting yard). It's about 30 lines and it will make your stack intuition noticeably sharper.
4. Do Basic Calculator II, then I. III if you have time.
5. Then decode a string with brackets — a stack problem in disguise, and a common follow‑up for RPN questions.
6. Before your interview, redo the truncation‑toward‑zero cases from memory: `-7 / 2 = -3`, `6 / -132 = 0`.

Ninety percent of candidates who fail this problem don't fail the algorithm. They fail operand order, or they use floor division, or they write `token[0] == "-"` and get a runtime error on `"-11"`. Fix those three and the RPN question becomes what it should be: ten calm minutes at the start of a round, instead of the thing you're still thinking about while the interviewer moves on to the next question.
