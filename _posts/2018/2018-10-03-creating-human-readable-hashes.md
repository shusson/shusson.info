# Creating human readable hashes

__03/10/2018__

![hashes](https://imgs.xkcd.com/comics/password_strength.png)

When displaying hospital data to users it's important to try to protect the privacy of patients. The nature of the data means there will always be identifiable data but we can do simple things to improve privacy like not displaying personal info like names. The easiest way to avoid this is to simply replace names with a pseudo-id which is a one way hash of the internal hospital id. However hashes are not easy to remember, and make it hard for users to reference or remember patients. We can get around this by transforming the pseudo-id hash into a human readable string.

There are some considerations to made about the desired output.

- easy to remember
- non-offensive
- easy to pronounce
- flexible for non-english speakers

Luckily this is not a new problem and there is prior work on this, although in slightly different domains.

- https://en.wikipedia.org/wiki/PGP_word_list
- https://en.wikipedia.org/wiki/NATO_phonetic_alphabet
- https://en.wikipedia.org/wiki/What3words
- https://github.com/singpolyma/mnemonicode

For our solution we are going to combine the simplicity of the PGP algorithm with better words from mnemonicode.

Here's our word list:

```json
{
    "00": {
        "even": "academy",
        "odd": "adam"
    },
    "01": {
        "even": "address",
        "odd": "admiral"
    },
    "02": {
        "even": "adrian",
        "odd": "agenda"
    },
    "03": {
        "even": "alabama",
        "odd": "aladdin"
    },
    ...
}
```

And here's how we can convert the hash to a pseudonym:

```typescript

import words from "./words.json";

function convert(hash: string): string {
    if (!hash) {
        return hash;
    }

    // break the hex string representation of the hash up into bytes
    const keys = hash.toUpperCase().match(/.{2}/g);

    if (!keys) {
        return hash;
    }

    // apply the PGP algorithm using the modified word list.
    const pseudonym = keys
        .map((k, i) => {
            const parity = i % 2 ? "odd" : "even";
            return (words as any)[k][parity];
        })
        .join("-");

    return pseudonym;
}
```

Examples of a conversion:

`5704d149-b4f8-44a4-9a6d-01790ba16e3c`
=>
`career-carrot-rondo-solo-william-wizard-alabama-hunter-pencil-sierra-strange-present-wisdom-roof-mammal-puma`

To make it easy to remember we can only show the first n words of the pseudonym, but keep in mind that the fewer words you show the more likely there will be duplicates.
