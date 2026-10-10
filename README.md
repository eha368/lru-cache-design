# lru cache: hash map + doubly linked list, plus the O(1) follow-ups interviewers actually ask

Two very different people search this term. One is on LeetCode 146, staring at the line "both operations must run in O(1)" and wondering how an ordering problem can possibly be constant time. The other is in a design round being asked "why a doubly linked list instead of a singly linked one?", and realizing they don't have a clean answer.

Both come down to the same design decision. Once you see it, the implementation writes itself and the follow-ups stop being scary.

## What the name actually promises

LRU stands for least recently used. It's an eviction policy, and it only matters when the cache is full: you have a bounded amount of memory, a new key needs space, and something has to go.

The policy says: drop the entry whose last access was the longest ago. Not the entry inserted longest ago (that's FIFO), and not the entry accessed fewest times (that's LFU). Recency, not frequency.

That distinction explains why the implementation is a little fussy. Every read counts as a use, and so does every update to an existing key. If `get(7)` returns a hit, key 7 just became the most recently used item in the cache, and it has to move to the front of the eviction queue. Forgetting that one line is the single most common way a correct-looking solution fails on the hidden test cases.

LFU is harder for a related reason: it needs a per-key access counter, and you have to decide how to break ties and whether counts decay over time. LRU needs none of that bookkeeping beyond ordering.

## Why one data structure is never enough

The O(1) requirement is what forces the design. Try each structure alone and it breaks in a specific way.

| Structure | Problem |
| --- | --- |
| Array or list | Reordering on access means shifting elements, O(n) per touch |
| Hash map alone | O(1) lookup, but nothing tracks which key is oldest |
| Singly linked list alone | Finding a key is O(n), and you can't unlink an interior node without a pointer to its predecessor |
| Doubly linked list alone | O(1) splice anywhere, but finding the node for a key still means walking the list |

The trick is that these two structures have complementary failures. The map handles "find by key" natively and is bad at order. The linked list handles order and splicing perfectly and is bad at lookup. Bind them together and each operation is handled by the half that's good at it.

## The design in one paragraph

Keep a hash map whose values are **pointers to nodes, not values**. Each node holds a key, a value, and `prev`/`next` pointers. The list runs from most recently used at one end to least recently used at the other. Add a sentinel head and a sentinel tail so the list is never empty and no splice operation needs a null check.

Two invariants hold the thing together:

1. The map and the list always contain exactly the same set of keys. Every insert into one is paired with an insert into the other; every delete is paired too.
2. The map's value is always a node reference. The user-facing value lives inside the node.

With sentinels, `remove(node)` is two pointer writes and `add_to_front(node)` is four, regardless of whether the cache holds zero entries or a thousand. No branching for "is this the head?" or "is this the tail?", which is where most of the edge-case bugs live.

## Walking through get and put

Take the canonical example with capacity 2.

- `put(1, 1)` → cache holds key 1. List order from MRU: `[1]`.
- `put(2, 2)` → `[2, 1]`. Key 1 is now the least recently used.
- `get(1)` → returns 1. Key 1 is unlinked from its position and spliced back in at the MRU end. Order becomes `[1, 2]`.
- `put(3, 3)` → capacity is exceeded, so the node before the tail sentinel is evicted. That's key 2. Order becomes `[3, 1]`.
- `get(2)` → not in the map, returns -1. Key 2 is gone.

That last step is the part that catches people. Without the recency update inside `get`, key 2 would still be in the cache and `get(2)` would return 2.

`put` has two branches. If the key already exists, update the value and move the node to the MRU end; that counts as a use, so no eviction happens. If the key is new, insert it at MRU, then check the size: if it now exceeds capacity, take `tail.prev`, unlink it, and delete its key from the map. The key is the reason each node stores its own key. During eviction you're holding a node, and you need its key to delete the correct map entry.

## The five bugs that show up in interviews

**Not storing the key in the node.** This is the classic. You evict the tail node, then try to delete it from the map, and you have a value but no key. Wasteful workaround: scan the map for the entry whose node matches. Correct fix: the node carries its key from the moment it's created.

**No recency update on `get`.** Covered above. Missing it means recently read keys still get evicted.

**No recency update on `put` when the key exists.** Same failure in a different branch. An update is a touch.

**Rewriting pointers before unlinking.** If you reassign `node.prev` and `node.next` first and then try to disconnect the node from its old neighbors, you've already destroyed the information you needed. Unlink first, then splice into the new position.

**Map and list drifting apart.** Every structural change has to touch both. A node in the list with no map entry is unreachable but still consumes capacity; a map entry with no node is a dangling pointer waiting to crash. Debugging this is much easier if you write a small helper that prints the list order and dumps the key set, then assert they match after every operation.

## The standard library shortcuts, and when you're allowed to use them

Most languages ship a container that is already this dual structure with the bookkeeping hidden inside.

- Python: `OrderedDict`, with `move_to_end()` for the touch and `popitem(last=False)` for the eviction.
- Java: `LinkedHashMap` constructed with `accessOrder = true`, plus an override of `removeEldestEntry`.
- C++: `std::list` with `splice()`, paired with an `unordered_map` of iterators.
- Go: `container/list` with `MoveToFront`.

These are legitimate in plenty of interview rooms, and using them shows language fluency. Some interviewers ban them explicitly, because the whole point of the question is whether you understand the invariant the container is maintaining internally. Either way, say the invariant out loud before you write code: every key in the map points to a node in the list, the list encodes recency order, and both structures hold the same key set at all times. That sentence tells the interviewer you know why the shortcut works, not just that it exists.

Real production caches like Caffeine and the LRU-eviction logic inside Redis go further, because a single lock around the whole structure is a bottleneck under concurrent load.

## The follow-up questions, in order of how often they come up

**"What if multiple threads hit the cache at once?"** A single global lock is correct and slow. Mentioning finer-grained locking, sharding the cache by key range so each shard has its own lock, or read-write locks for read-heavy workloads shows you've thought past the LeetCode version. For a distributed cache, the conversation moves to Redis, Memcached, and cache-aside patterns.

**"How would you implement LFU instead?"** You need frequency buckets: a map from key to node, a map from frequency to an ordered list of nodes with that frequency, and a pointer to the current minimum frequency. Eviction takes the least-frequently-used node from the lowest non-empty bucket. It's meaningfully more code than LRU, and interviewers usually just want to hear that you know why.

**"What about TTL?"** Add an absolute expiry timestamp to each entry and check it lazily on access. The tradeoff: lazy-only expiry keeps expired-but-unread keys in memory, so you typically add a low-frequency background sweep on top.

**"The input is streamed rather than loaded into memory. What changes?"** Recency tracking has to be approximate. Redis, for example, samples a subset of keys and evicts the oldest of those rather than maintaining a perfect global order, because exact ordering doesn't survive at that scale.

**"Why not just use a singly linked list?"** Because unlinking an arbitrary node in O(1) requires rewriting the predecessor's `next` pointer, and a singly linked list gives you no way to reach the predecessor. Some variants store a pointer-to-pointer ("link stealing") to work around this, and mentioning it as an option is a nice touch.

## Where to practice this instead of reading it

Reading a walkthrough is not the same as writing the pointer bookkeeping under time pressure, which is where the bug list above actually bites.

AlgoMonster covers LRU cache from both directions. There's a **low-level design lesson** that treats it as a Design-the-Data-Structure interview problem — decision flowcharts, class layout, and the follow-up discussion about concurrency — and a separate LeetCode 146 walkthrough with runnable code in Java, Python, JavaScript, C++, and Go, plus an inline editor so you can break your own pointer logic and watch it fail.

👉 [Work through the LRU cache low-level design lesson](https://algo.monster/courses/lld/lld_lru_cache?fpr=li69)

👉 [Solve LeetCode 146 with the AlgoMonster editorials and templates](https://algo.monster/liteproblems/146?fpr=li69)

The platform's whole pitch is a pattern library rather than a problem pile, which matters here: LRU cache is the canonical example of "hash map + ordered structure" hybrid design, and the same reasoning reappears in LFU cache, rate limiters, and the key-value store problems. AlgoMonster's own content makes that connection explicitly — the in-memory key-value store lesson reuses the same LRU list as a component.

## AlgoMonster plans, as they stand on the official page

There's a free tier with limited sample content, which is enough to judge whether the text-and-diagram teaching style suits you. Paid access comes in two forms, and there's no monthly option.

| Plan | What you get | Price | Billing | Link |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns and problems | $0 | — | [Explore the free content](https://algo.monster?fpr=li69) |
| Pro — Yearly | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; 500+ lessons across 6 courses; 1,000+ illustrations; 3,000+ editorials; company question sets updated monthly; bonus Foundation and Build-an-LLM courses | $8.25/mo, billed as **$99/year** | 12 months, cancel or upgrade anytime | [See the current plans](https://algo.monster/subscribe?fpr=li69) |
| Pro — Lifetime | Everything in Yearly, plus lifetime updates | **$189** | One-time | [Check lifetime access](https://algo.monster/subscribe?fpr=li69) |

A caution about numbers floating around elsewhere. Several blog posts quote $89/year for the annual plan or $119–$459 ranges for lifetime, and at least one claims a specific promo code stacks an extra 10% on top of a 50% sale. Those figures conflict with each other and with the official page, which shows $99/year and $189. The plan page is the only number worth trusting; promotional pricing moves.

For the LRU cache question specifically, the annual plan is the honest recommendation if you have interviews in the next two to three months. The lifetime plan only makes sense if you expect to be back in interview prep in a couple of years — which, for most engineers, is exactly what happens.

## What AlgoMonster does not do well

The independent reviews that exist — from Lodely, Final Round AI, and a few dev.to writeups — converge on the same two complaints, and both are fair.

There is no video. Every explanation is text and interactive diagrams. If you learn by watching someone code on screen, this will feel dry compared to NeetCode or AlgoExpert, and the format is a deliberate design choice rather than a gap they plan to fill.

System design coverage is introductory. The low-level design material is genuinely useful for questions like LRU cache, key-value stores, and rate limiters, but if you're interviewing at L5 or above where a full distributed systems round is standard, you'll need a separate resource for it. AlgoMonster does not replace that.

A third limitation is worth naming: the problem set is small by design. Around 230 lessons and problems will teach you the patterns, but it won't give you the volume that makes them automatic. Most people who use it pair it with LeetCode, using AlgoMonster for the framework and LeetCode for reps.

## The short version

LRU cache is a two-structure problem. A hash map gives you O(1) lookup by key, a doubly linked list gives you O(1) reordering and O(1) eviction at the tail, and sentinels keep you out of edge-case hell. Every `get` and every `put` on an existing key is a touch. New nodes carry their own key so eviction can clean up the map. Everything else is pointer bookkeeping you have to write by hand until it stops being interesting.

If you can explain that in ninety seconds, defend the choice of a doubly linked list, and sketch the class in the language of your choice, you're ready for the question. The rest is reps.

👉 [Start the LRU cache design lesson here](https://algo.monster/courses/lld/lld_lru_cache?fpr=li69)
