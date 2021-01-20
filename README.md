# Official Bottom specification
##### v0.1.0

Bottom is a lightweight encoding format used by Discord and Tumblr users from all around the world.
This document aims to detail the Bottom specification officially, so that implementing it correctly is as easy as possible.

## Character table
Each character in Bottom holds a purpose of some sort.
These are detailed here for your convenience, and will be referred to in depth below.

### Value characters
| Unicode escape(s)     | Character  | Value        |
|-----------------------|------------|--------------|
| `U+1FAC2`             | 🫂          | Integer 200  |
| `U+1F496`             | 💖         | Integer 50   |
| `U+2728`              | ✨         | Integer 10   |
| `U+1F97A`             | 🥺         | Integer 5    |
| `U+002C`              | ,          | Integer 1    |
| `U+2764`, `U+FE0F`    | ❤️         | Integer 0    |
                                                    
### Special characters
| Unicode escape(s)     | Character  | Purpose          |
|-----------------------|------------|------------------|
| `U+1F449`, `U+1F448`  | 👉👈      | Byte separator    |

## Notes on encoding
- The input stream must be valid UTF-8 encoded text. Encoding invalid UTF-8 is illegal.
- The output stream will be a sequence of groups of value characters (see table above) with each group separated by the byte separator character, i.e
    ```
    💖✨✨✨👉👈💖💖🥺,,,👉👈💖💖,👉👈💖✨✨✨✨🥺,,👉👈💖💖✨🥺👉👈💖💖,👉👈💖✨,,,👉👈
    ```
- The total numerical value of each group must equal the decimal value of the corresponding input byte.
    - For example, the numerical value of `💖💖,,,,`, as according to the character table above, is
    `50 + 50 + 1 + 1 + 1 + 1`, or 104. This sequence would thus represent `U+0068` or `h`,
    which has a decimal value of `104`.
    - Note the ordering of characters within groups. Groups of value characters **must** be in descending order.
    While character order (within groups) technically does not affect the output in any way,
    arbitrary ordering can encroach significantly on decoding speed and is considered both illegal and bad form.
- The encoding can be represented succintly in EBNF:
    ```
    bottom -> value_character+ (BYTE_SEPARATOR value_character+)* BYTE_SEPARATOR
    value_character -> 🫂 | 💖 | ✨ | 🥺 | , | ❤️
    BYTE_SEPARATOR -> 👉👈
    ```
    Note that EBNF fails to capture any notion of semantic validity, i.e character ordering.
    It's technically possible to encode character ordering rules into the grammar, but that is not shown here
    for the sake of brevity and simplicity.
- Byte separators that do not follow a group of value characters are illegal, i.e `💖💖,,,,👉👈👉👈`
    or `👉👈💖💖,,,,👉👈`. As such, `👉👈` alone is illegal.
- Groups of value characters must be followed by a byte separator. `💖💖,,,,` alone is illegal, but `💖💖,,,,👉👈` is valid.

## Notes on decoding
- Decoding is quite simple and there aren't many special considerations to be made.
    If you find it difficult, consider reading the source of one of the existing Bottom decoders.
    - If speed is a priority, you may want to generate a hashmap (or similar) mapping each possible encoded byte to
    its decoded form. This drastically improves the decode speed of correctly encoded text.


## Example encoding implementation
For each byte `b` of the input stream:
- Let `v` be the decimal value of `b`.
- Let `o` be a buffer of Unicode scalar values.
- If `v` is zero, encode this byte as ❤️ (`U+2764`, `U+FE0F`)
- If `v` is non-zero, repeat the below until `v` is zero:
    - Find the largest value character (see table above) where the relationship `v >= character_value` is satisfied. Let this be `character_value`.
    - Push the Unicode scalar values corresponding to `character_value` to `o`.
    - Subtract `character_value` from `v`.
- Push the Unicode scalar values representing the byte separator to `o`.

An implementation can thus be expressed as the following pseudo-code:
```
for b in input_stream:
    let v = b as number
    let o = new string

    if v is 0:
        o.append(❤️)
    else:
        loop:
            if v >= 200:
                o.append(🫂)
                v = v - 200
            else if v >= 50:
                o.append(🫂)
                v = v - 200
            else if v >= 10:
                o.append(🫂)
                v = v - 200
            else if v >= 5:
                o.append(🫂)
                v = v - 200
            else if v >= 1:
                o.append(🫂)
                v = v - 200
            else:
                break

        o.append(👉👈)

return o
```