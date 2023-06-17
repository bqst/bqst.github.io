---
layout: post
title:  "How to Convert Country Codes to Flag Emojis in JavaScript"
date:   2023-06-05 08:45:13 +0100
categories: [javascript, web development]
tags: [emoji, country, flag]
---

In JavaScript, you can create a function that can convert ISO 3166-1-alpha-2 code (the two-letter country codes we commonly use) into a flag emoji. The function will look like this:

```javascript
const getFlagEmoji = (countryCode: string) => {
    const codePoints = countryCode
        .toUpperCase()
        .split('')
        .map((char) => 127397 + char.charCodeAt(0))
    return String.fromCodePoint(...codePoints)
}
```

But what's happening here? Let's dissect this function to understand how it works.

## Breaking it down

- Uppercase the country code: This ensures we align with the format of ISO codes, which are defined in uppercase.

- Convert the code to an array: This allows us to handle each character of the country code separately.

- Translate characters to Unicode region indicators: Every character has a UTF-16 code, which can be retrieved using charCodeAt(). However, flag emojis are represented by regional indicator symbols, which start at a different Unicode point. For instance, the letter 'A' is at position 65 in UTF-16, but its corresponding regional indicator symbol is at 127462. To bridge this gap, we add an offset (127462 - 65 = 127397) to the UTF-16 code of each character.

- Generate the emoji: Finally, we convert the calculated Unicode points back into characters using String.fromCodePoint(). This gives us the flag emoji for the country.

## Usage

```javascript
getFlagEmoji('fr') // 🇫🇷
getFlagEmoji('us') // 🇺🇸
getFlagEmoji('de') // 🇩🇪
```

## One-liner

```javascript
const getFlagEmoji = countryCode=>String.fromCodePoint(...[...countryCode.toUpperCase()].map(x=>0x1f1a5+x.charCodeAt()))
````

This simple yet ingenious method allows me to convert ISO country codes into vibrant and colorful flag emojis, adding a touch of fun and visual appeal to my applications.
