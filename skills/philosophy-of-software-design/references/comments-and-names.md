# Comments and names

Comments and names carry what the signature cannot. They are part of the design, not a cleanup pass.

## When a comment earns its place

A comment stays only when a first-time reader could not write it from the adjacent code. If they could, delete it or replace it with the missing fact. A reviewer's confusion settles the question.

Do not comment every line. Do not use a paragraph to hide a shallow module, a bad name, or a special case that should be deleted.

## Interface comments

These are the abstraction. A reader should use the type or method without opening the body.

- A type comment says what the type is for, what one instance represents, and which limits a caller must respect.
- A method comment starts with caller-visible behavior, then states each argument and the result, side effects, observable failures, and preconditions.

If the interface comment has to explain the implementation, the module is shallow.

## Implementation comments

Most short methods need none. For a long method, put a high-level note before each major block and before a loop whose iteration is not obvious. Say what and why. The code already says how.

- Lower-level comments add precision the declaration lacks: units, whether an end index is included, what null means, who frees a resource, invariants. Write them as nouns.
- Higher-level comments give the one or two ideas that make the block intelligible.

## Where comments live

- Put each comment at the narrowest place that covers everything it describes. The farther it sits from the code, the more abstract it should be.
- Record each design decision once, at the obvious declaration, and point there from other sites. A broken pointer is obvious. A stale duplicate is not.
- Do not document one module's decisions inside another.
- If a future developer needs a fact, it belongs in the code. The commit log is a poor place to hide it.
- Before commit, read the diff for comment drift.

Adopt one comment convention and stick to it.

## Write the comment first

For a new module:

1. Write the type comment.
2. Write comments and signatures for the important public methods. Leave bodies empty.
3. Revise until the structure is easy to say.
4. Declare the important fields with their comments.
5. Fill in bodies. Add implementation comments only where a block needs a frame.
6. For anything discovered later, write its comment with the declaration.

A long comment that is still incomplete means the interface is too complex or the comment is cryptic. If the comment must mention the major implementation pieces, the method is shallow. A deep method has a small interface description and a lot of work behind it.

Revise the comment when coding reveals a better abstraction.

### The usual excuses

- "Good code documents itself." Signatures omit meaning, boundaries, and why the design exists. Forcing readers into the body destroys the abstraction and pushes the design toward many shallow methods.
- "There is no time." A postponed comment is a missing comment. The comment written while designing is the design check.
- "Comments rot." They rot when they are copied and stored far from the code. Keep one copy next to the code.
- "Comments I have seen were useless." Restating the next line teaches people to ignore comments. Adding the missing fact does not.

## Names

A name is a small abstraction. Shown only the name, a stranger's guess should be close.

- Pick the few words that highlight what matters. Two or three words is usually the limit before the name is unwieldy.
- Vague names invite the wrong assumption. Over-specific names lie.
- For a recurring concept, use one name everywhere, and do not use that name for anything else. When you need two, add a prefix (`src` / `dst`) rather than a second root.
- A boolean should read as a predicate.
- `i` and `j` are fine when the whole loop is visible and the meaning is the iteration itself. A short name reused for several meanings is how false assumptions return. Distance from the declaration calls for a longer name.

If you cannot find a precise name, the thing may be doing two jobs or have no crisp purpose. Split it or redefine it. One ambiguous name causes bugs because people trust the name and stop checking the value.

Readers decide. If review calls the names cryptic, lengthen them. Do not lengthen a loop index whose whole life is three lines.
