+++
title = "Hiding code inside identifier"
date = "2025-02-21"
keywords = ["javascript","variation","selector","unicode","hack","hiding","identifier"]

[extra]
repo_view = false
comment = false
+++

# Unicode
[Unicode](https://www.unicode.org/versions/Unicode16.0.0/UnicodeStandard-16.0.pdf) maps codepoints to characters. E.g., U+0041 -> A. While some characters map directly to one codepoint, some scripts, e.g., Devanagari, require multiple codepoints per character. To accommodate all of the world's writing systems, Unicode includes code points supporting the combination of characters or fine variations of characters in different scripts.

<!-- There are many cool [codepoints](https://github.com/Codepoints/awesome-codepoints).  -->

To create variations of a character without assigning each variation a new code point, Unicode introduced [Variation Selectors](https://en.wikipedia.org/wiki/Variation_Selectors_Supplement) (VS-1 to VS-256). Variation selectors modify the appearance of preceding characters but have no visual representation themselves. One use case is to force some characters to be in text or emoji style.

```javascript
console.log(String.fromCodePoint(0x2615));         // ☕ Default (may appear as text or emoji)
console.log(String.fromCodePoint(0x2615, 0xFE0E)); // ☕︎ Forces text style
console.log(String.fromCodePoint(0x2615, 0xFE0F)); // ☕️ Forces emoji style
```
Variation selectors are supposed to be preserved even if their meaning is unknown to a system in order for Unicode to be backward compatible. Unicode does not seem to define a maximum of variation selectors for one base character either.

# Encoding into Variation Selectors
Given the properties of variation selectors, we could add arbitrary data to a base character by mapping the data to the different variation selectors. Most systems will carry this unknown data around without modifying it. 

Since we have exactly 256 variation selectors, we can assign a byte value to each selector and use this as a way to encode arbitrary data after each Unicode character. Variation selectors exist in two codepoint ranges: 16 in ``U+FE00`` to ``U+FE0F`` and 240 in ``U+E0100`` to ``U+E01EF``. If we combine both ranges when mapping to byte values, we are going to end up with a mapping like this:

<center>

| Variation Selector | Code Point  | Byte    |
|--------------------|------------|-----------------------|
| VS-1              | U+FE00      | 0x00      |
| VS-2              | U+FE01      | 0x01      |
| ....              | ....        | ....                |
| VS-15             | U+FE0E      | 0x0E       |
| VS-16             | U+FE0F      | 0x0F       |
| VS-17             | U+E0100     | 0x10 |
| VS-18             | U+E0101     | 0x11 |
| ...               | ...         | ...                   |
| VS-256            | U+E01EF     | 0xFF |

</center>
<center>
Mapping Code Points to Bytes
</center>

Using this mapping we can encode arbitrary bytes using the different variation selectors. If the variation selectors have no meaning for the base character, there will be no visual indication of our appended data.

# Embedding hidden data in strings
Storing arbitrary data after a character is an interesting property that could have many applications. Maybe the first idea that comes to mind is to use it to send hidden messages like [StegCloack](https://github.com/KuroLabs/stegcloak) is doing. 

However, I wanted to test how variation selectors behave inside programming languages that support Unicode. We are going to use JavaScript for this to encode a message into a character.

First, we need to encode a hidden message by appending variation selectors to a base character.

```javascript
function encode(baseChar, stringData) {
    let result = baseChar;

    for (let char of stringData) {
        let byte = char.charCodeAt(0);

        result += (byte < 16) 
            ? String.fromCodePoint(0xFE00 + byte) 
            : String.fromCodePoint(0xE0100 + (byte - 16));
    }

    return result;
}
```

The ``baseChar`` can be a Unicode character like "a", "b", or "😀󠅜󠅟󠅜" etc. As long as the variation selectors don't modify the base character it's fine.

```javascript
let emoji = encode("😀", "hello hidden world!");
console.log(emoji);
```

The resulting ``emoji`` has the string ``"hello hidden world!"`` appended, encoded as variation selectors. You cannot see them. But if you e.g., copy the smiley, you will copy the variation selectors too.

Now, let’s decode the hidden message again:

```javascript
function decode(input) {
    let result = [];
    let startedDecoding = false;

    for (let char of input) {
        const codePoint = char.codePointAt(0);
        let byte = null;

        if (codePoint >= 0xFE00 && codePoint <= 0xFE0F) {
            byte = codePoint - 0xFE00;
        } else if (codePoint >= 0xE0100 && codePoint <= 0xE01EF) {
            byte = codePoint - 0xE0100 + 16;
        }

        if (byte !== null) {
            result.push(byte);
            startedDecoding = true;
        } else if (startedDecoding) {
            break; // Stop decoding after encountering a non-variation selector
        }
    }

    return result;
}
```

The ``decode()`` function will skip the first base character and then decode variation selectors as long as there is no non-variation-selector. The output is a byte array which we convert back into a string with ``String.FromCharCode()``:

```javascript
let bytes = decode(emoji);
console.log(String.fromCharCode(...bytes));
```

When we print the output, we get the original message: ``"hello hidden world!"``. Instead of encoding a random message, we can also encode JavaScript code and ``eval()`` it:

```javascript
let emoji = encode("😀", "console.log(\"hello hidden world!\")");
let bytes = decode(emoji);
eval(String.fromCharCode(...bytes))
```

This is cool because we can hide now arbitrary amounts of code inside single characters and at runtime we can decode and execute it again. 

# Embedding hidden data in identifier
JavaScript does not only support Unicode in Strings but also for identifiers.
Let’s choose a random character where we hide the JavaScript and print it:

```javascript
let a = encode("a", "console.log(\"hello hidden world!\")");
console.log(String.fromCharCode(...a));
```

Now we can copy the symbol ``a󠅓󠅟󠅞󠅣󠅟󠅜󠅕󠄞󠅜󠅟󠅗󠄘󠄒󠅘󠅕󠅜󠅜󠅟󠄐󠅘󠅙󠅔󠅔󠅕󠅞󠄐󠅧󠅟󠅢󠅜󠅔󠄑󠄒󠄙`` from our console with Unicode support (you can copy it from here too) and use it as an identifier inside our JavaScript code. To decode the hidden script, we need to get the name of the identifier. There seems to be no direct way to do this, but, we can do it by placing the identifier inside an object and accessing the first key via ``Object.keys()``. Now we can get the identifier name, decode it, and execute it.

```javascript
let obj = { a󠅓󠅟󠅞󠅣󠅟󠅜󠅕󠄞󠅜󠅟󠅗󠄘󠄒󠅘󠅕󠅜󠅜󠅟󠄐󠅘󠅙󠅔󠅔󠅕󠅞󠄐󠅧󠅟󠅢󠅜󠅔󠄑󠄒󠄙 : 1 };
let varName = Object.keys(obj)[0];
let bytes = decode(varName);
eval(String.fromCharCode(...bytes)); // prints: "hello hidden world!"
```

This will print ``"hello hidden world!"`` which means we can hide arbitrary amount of JavaScript inside identifiers. This should work for other script languages too that allow to access variable names and allow Unicode in identifiers. Hiding code inside characters of string literals should work for any language that supports Unicode too.

