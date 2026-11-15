# Huffman & LZW Coding: Group 8 Briefing (Draft)

INF605 Digital Image Processing, Fall 2026. Draft of 15 Nov 2026.

## Start here

Our two papers solve the same problem in two different ways: **how to store data in fewer bits without losing anything**. Huffman (1952) gives shorter codes to common symbols. Welch (1984) gives short codes to repeated *chunks* of data.

This briefing is for everyone 8 before we start implementing. Each part goes over one step of the story in plain words, with small examples we can check by hand.

**How to read the labels.** Anything marked *(from the paper)* is stated in Huffman's or Welch's text, which we read. Anything marked *(background)* is general knowledge added for context# mostly with help from Claude.

**The one-minute version**

- Data wastes bits because some symbols are common and some patterns repeat. Compression recovers those wasted bits.
- Huffman's method turns symbol frequencies into the best possible set of short and long codes, one code per symbol.
- Welch's method (LZW) builds a phrasebook of repeated strings while reading the data, and replaces each phrase with its page number.
- Both are **lossless**: decoding gives back exactly the original, bit for bit.
- In our project we implement both for grayscale images, then measure how well they do against the image's entropy and against PNG.

## 1. Before 1952: wasted bits, and a proven limit

Before Huffman, people knew data could be shortened and knew the theoretical limit, but had no clean recipe for reaching it.

### The waste

A simple way to store text or pixels is a **fixed-length code**: every symbol gets the same number of bits. ASCII uses 8 bits for every character, and a grayscale pixel uses 8 bits for every value from 0 to 255.

That is wasteful when symbols are not equally common. In English, "e" and the space appear far more often than "z". Giving all of them 8 bits spends the same effort on the common and the rare. *(background)* Morse code already had the right instinct: the very common letter E is a single dot.

Welch makes the same point about ASCII files: nearly three quarters of the 256 possible byte values may never appear in a given file *(from the paper)*.

### Entropy: the speed limit of compression

In 1948 Claude Shannon asked how few bits per symbol are really needed *(background)*. His answer is a number called **entropy**. Start with one idea: a rare event is more *surprising* than a common one, and surprise can be measured in bits.

- An event with probability 1/2 carries 1 bit of surprise (a fair coin flip).
- Probability 1/4 carries 2 bits. Probability 1/8 carries 3 bits.
- In general, an event with probability $p$ carries $\log_2(1/p)$ bits.

Entropy is the **average surprise** over all symbols, weighted by how often each one occurs:

$$H = -\sum_{i} p_i \log_2 p_i$$

Read it as: for each symbol $i$, take its probability $p_i$, multiply by its surprise $\log_2(1/p_i)$ (the minus sign just flips the log of a fraction), and add everything up.

**Small example.** Two symbols with probabilities 0.9 and 0.1. The common one is cheap (0.9 × 0.15 bits), the rare one is expensive (0.1 × 3.32 bits). Together H ≈ 0.47 bits per symbol, far below the 1 bit a plain code spends.

The key fact: **no lossless code can use fewer than H bits per symbol on average**, if it codes symbols one at a time from these probabilities. Entropy is the floor we measure ourselves against. Huffman's paper cites Shannon's theory, and Welch's text describes the same idea as code length being about log2 of the symbol probability *(from the papers)*.

### What we had: close, but not best

Huffman names Shannon and Fano as having built coding procedures that approach this limit. He says their procedures are not optimum for a finite set of messages, and only come close as the number of messages grows without end *(from the paper)*. He also notes that Kraft had worked on the finite case, but that no definite procedure for building the best code was known to him.

*(Background)* Shannon-Fano coding sorts the symbols by probability and splits them into two groups of roughly equal total probability, then repeats inside each group. It works from the top down, and it sometimes lands on a code that is slightly longer than necessary. The open question was: what is the *best possible* code, and how do you build it?

### Why this matters for images

A grayscale image is a list of pixel values. Count how often each value occurs (a **histogram**), divide by the number of pixels, and you have the probabilities $p_i$. Entropy then tells you the minimum average bits per pixel that coding pixels one at a time can reach. A photo with many similar shades has lower entropy than 8 bits, so there is room to compress.

## 2. Huffman 1952: the best possible code per symbol

Huffman's paper gives a short, exact recipe that builds the shortest possible average code when every symbol gets its own code. The paper itself is only three pages of content.

### The goal in one line

Each symbol $i$ has a probability $P(i)$ and gets a code of length $L(i)$ bits. Huffman wants the **average code length** as small as possible *(his equations 1 and 2)*:

$$L_{av} = \sum_{i=1}^{N} P(i)\, L(i)$$

Read it as a weighted average. Multiply each symbol's code length by how often it occurs, then add. A symbol that is common and has a long code hurts a lot. A symbol that is rare and has a long code hurts very little. So the best codes give long codes only to rare symbols.

### The rule that keeps decoding possible

Codes have different lengths, so the decoder needs to know where one code ends and the next begins, with no separator. Huffman's restriction (b) solves this: **no code may be the beginning of another code** *(from the paper)*. Today this is called a **prefix-free** code.

- Bad example: A = 0 and B = 01. The bits "01" could be B, or A followed by something that starts with 1. The decoder cannot tell.
- Good example: A = 0, B = 10, C = 11. Reading left to right, there is only one way to split any bit string.

### The recipe

Huffman shows that an optimal code must satisfy a few rules: a more probable symbol never gets a longer code, and the two least probable symbols get codes of the same length that differ only in the last bit *(from the paper)*. From that he builds the code bottom-up:

1. List the symbols with their probabilities.
2. Take the two least probable symbols and **merge** them into one combined symbol. Its probability is the sum of the two.
3. Repeat step 2 on the new, shorter list until one symbol with probability 1 remains.
4. Walk back down the merges. At each merge, give one branch a 0 and the other a 1. A symbol's code is the sequence of 0s and 1s on the path to it.

The merges form a binary tree (people usually call it a Huffman tree). Rare symbols end up deep in the tree, so they get long codes. Common symbols stay near the top, so they get short codes.

### A worked example we can check by hand

Five symbols with these probabilities:

| Symbol | Probability | Plain fixed code |
| --- | --- | --- |
| A | 0.4 | 3 bits |
| B | 0.2 | 3 bits |
| C | 0.2 | 3 bits |
| D | 0.1 | 3 bits |
| E | 0.1 | 3 bits |

Merge steps:

1. The two smallest are D (0.1) and E (0.1). Merge them into DE (0.2).
2. The list is now A 0.4, B 0.2, C 0.2, DE 0.2. Merge C and DE into CDE (0.4).
3. The list is now A 0.4, B 0.2, CDE 0.4. Merge B and A into AB (0.6).
4. Merge AB (0.6) and CDE (0.4) into the root (1.0).

Assign 0 and 1 at each merge and read off the codes:

| Symbol | Code | Length L(i) | P(i) × L(i) |
| --- | --- | --- | --- |
| A | 00 | 2 | 0.8 |
| B | 01 | 2 | 0.4 |
| C | 10 | 2 | 0.4 |
| D | 110 | 3 | 0.3 |
| E | 111 | 3 | 0.3 |

Adding the last column gives **L_av = 2.2 bits per symbol**, compared with 3 bits for the fixed code. The entropy of this source is about 2.12 bits, so Huffman is very close to the limit.

To encode the message A D C A, write 00 110 10 00. That is 9 bits instead of 12.

There are ties in step 2 (B, C and DE all have 0.2), so different tie-breaks give different codes. Huffman points this out himself: the average length is the same either way *(from the paper)*. This matters for testing, because our codes may legitimately differ from another implementation's while having identical lengths.

### What the paper reports

Huffman's own worked example (his Table I and Table II) uses 13 messages and arrives at an average length of **3.42 digits per message**. He also shows how to extend the method to codes with more than two symbols, using a four-symbol example (his Table III). We can use the 13-message example as a result to reproduce.

### What it can and cannot do

- It is **optimal** among codes that give each symbol its own whole number of bits.
- *(Background)* Its average length always lands between H and H + 1 bits. The "+ 1" comes from the whole-bit rule: a symbol with probability 0.9 deserves about 0.15 bits but must get at least 1 bit. Later methods such as arithmetic coding remove that limit.
- It needs the probabilities first. For an image, that means counting a histogram before encoding.

## 3. 1952 to 1984: Huffman works, but real systems push back

Huffman's method became the standard way to code symbols by frequency. Over the next 30 years its practical weaknesses showed up, and a second family of methods appeared. The two papers are not parent and child. They are two different answers to the same question, written for different worlds.

### Weakness 1: it needs the statistics first

To build the tree you must know every symbol's probability. For a whole file that means **two passes**: one to count, one to encode. The code table must also travel with the data so the decoder can rebuild the tree, and that table costs bits. Welch lists exactly this problem *(from the paper)*: different file types (English text, object code, source code, system tables) have different statistics, so one fixed table does not fit all, and per-file tables add overhead and delay.

*(Background)* The fix was **adaptive Huffman coding**, where the encoder and decoder update the tree as they go. Faller (1973) and Gallager (1978) proposed it, and Knuth and Vitter refined it in the mid-1980s.

### Weakness 2: it only sees single symbols

Huffman's recipe codes one symbol at a time. It exploits that "e" is common, but not that "th" or the word "compression" keeps coming back. Welch calls these other kinds of redundancy *character repetition* (runs of the same symbol), *high-usage patterns* (frequent sequences) and *positional redundancy* (the same value at the same place in every record), and notes Huffman handles only the first kind well *(from the paper)*.

Coding pairs of bytes would help, but a table of all byte pairs has 64K entries (65,536), which Welch says is too costly *(from the paper)*.

### Weakness 3: decoding is slow in hardware

Huffman codes have variable lengths, so the decoder walks the tree one bit at a time. Welch argues that at disk speeds (he gives 30 million bits per second as an example) the decoder has to keep up, and a bit-by-bit tree walk is a bottleneck *(from the paper)*.

### Weakness 4: whole bits only

*(Background)* Because each code is a whole number of bits, Huffman wastes a little on very skewed sources. Arithmetic coding (late 1970s) removed that limit, but it is more complex and was slower at the time.

### A new idea: build a dictionary instead

In 1977 and 1978 Abraham Lempel and Jacob Ziv published two papers describing a different approach *(background; Welch cites both as references 5 and 6)*. Instead of asking how likely each symbol is, they replace **repeated strings** with short references to a dictionary that is built while reading the data. No probabilities are needed, and one pass is enough. The 1977 version points back into recent data (a sliding window); the 1978 version builds an explicit dictionary of phrases. Welch's method grows out of the 1978 version.

### The world changed too

- Computer systems stored large amounts of mixed data on disks and tapes, and storage was expensive. Welch frames the goal as **"transparent" compression**: the programmer never notices it, except that the disk seems bigger *(from the paper)*.
- A compressor sitting between the computer and a disk has to be fast and cheap to build. Algorithms needed to be simple enough for hardware.
- *(Background)* Compression also appeared in standards, for example fax machines around 1980 coded runs of black and white pixels with Huffman-style tables.

### Why 32 years?

The gap is not a mystery. Huffman (1952) solved the symbol-statistics problem so completely that there was little left to improve on that front. The next big idea, the dictionary approach, arrived only in 1977 to 1978. Welch (1984) then took that idea and made it practical for real computer systems. His paper is more an engineering paper than a theory paper.

Welch also gives a sense of what was achievable. For English text he reports a compression ratio of about 1.8 with Lempel-Ziv style methods, and he cites others who saw 1.5 with Huffman coding and up to 2.4 with more complex algorithms. He warns that these comparisons are not very reliable because the texts differ *(from the paper)*.

## 4. Welch 1984: LZW and the string table

Welch's method, LZW (Lempel-Ziv-Welch), compresses by building a phrasebook while it reads, and replacing each phrase it already knows with a short number. It needs no statistics in advance, reads the data once, and is simple enough to build in hardware.

### The big idea: a phrasebook

Imagine writing a long text. The first time you write "compression" you spell it out. You also note it in a phrasebook as entry number 300. The next time, you write just "300". The reader builds the same phrasebook as they read, so the numbers always make sense, and **the phrasebook itself never needs to be sent**.

Welch calls the phrasebook the **string table**. It maps strings of characters to code numbers. A typical size is 12-bit codes, so up to 4096 entries *(from the paper)*. The table starts with every single character already in it (for bytes, that is 256 entries). Everything after that is learned.

Two rules from the paper define the method:

- **Prefix property.** For every string in the table, the string minus its last character is also in the table. So the table can be stored as "entry = an earlier entry + one extra character".
- **Greedy parsing.** At each step, take the **longest string from the input that is already in the table**.

### Encoding, step by step

Keep a current string w (starts empty). Read the input one character K at a time:

1. If w + K is in the table, extend: w becomes w + K.
2. If it is not, output the code for w, add w + K to the table as a new entry, and restart with w = K.
3. At the end of the input, output the code for w.

**Worked example.** Input: A B A B A B A. Start table: A = 0, B = 1.

| Step | Read | Is w+K in table? | Output | New table entry |
| --- | --- | --- | --- | --- |
| 1 | B (w = A) | AB: no | 0 (for A) | 2 = AB |
| 2 | A (w = B) | BA: no | 1 (for B) | 3 = BA |
| 3 | B (w = A) | AB: yes | none | none (w = AB) |
| 4 | A (w = AB) | ABA: no | 2 (for AB) | 4 = ABA |
| 5 | B (w = A) | AB: yes | none | none (w = AB) |
| 6 | A (w = AB) | ABA: yes | none | none (w = ABA) |
| End | | | 4 (for ABA) | |

Output: **0, 1, 2, 4**. Seven characters became four codes. Notice that the encoder discovered AB and then ABA by itself, and used them as soon as they repeated.

### Decoding: rebuild the same table

The decoder starts with the same single-character table and reads codes. For each code it outputs the string, and it adds one new entry: the *previous string* plus the *first character of the current string*. This produces exactly the entries the encoder created, one step later.

Decode 0, 1, 2, 4:

1. Code 0 is A. Output A.
2. Code 1 is B. Output B. Add A + B = AB as entry 2.
3. Code 2 is AB. Output AB. Add B + A = BA as entry 3.
4. Code 4 is **not in the table yet** (it only goes up to 3).

Step 4 is the one tricky case Welch discusses, often called the **KwKwK case** *(from the paper)*. The encoder used an entry it created only a moment ago. The decoder can still work it out: the new string must be the previous string (AB) plus that string's own first character (A), giving ABA. Output ABA and add it as entry 4.

Full output: A, B, AB, ABA = ABABABA. This matches the input. Every project implementation of LZW must handle this case, and it is the usual source of decoder bugs.

### Why the savings appear

If codes are 12 bits and characters are 8 bits, a phrase pays off once it covers more than 1.5 characters on average. In the toy example, 4 codes × 12 bits = 48 bits against 7 × 8 = 56 bits. The gain is small because the input is short and the table is still empty at the start. Welch says exactly this: adaptive methods compress poorly at the start of a message while the table fills, so the message must be long enough to learn from *(from the paper)*. He notes that LZW reaches acceptably good compression on messages around ten thousand characters.

### Built for speed

Welch's main engineering point is that LZW is fast. The compressor must search the table for w + K at every step, so he suggests hashing (a way to look entries up almost directly). The decompressor reads its table directly by code number, so it needs no search at all *(from the paper)*. He claims a typical implementation runs in under three clock cycles per symbol.

### What the paper reports

Compression ratio here means the uncompressed bit count divided by the compressed bit count, so 2.0 means the data shrank to half *(from the paper)*. Welch's Table 2 gives:

| Data type | Ratio |
| --- | --- |
| English text | 1.8 |
| Cobol files | 2 to 6 |
| Floating-point arrays | 1.0 |
| Formatted scientific data | 2.1 |
| System log data | 2.6 |
| Program source code | 2.3 |
| Object code | 1.5 |

He states that these were measured with several Lempel-Ziv variants that agreed within a few percent, not with one single LZW program. Floating-point arrays look like random noise and compress poorly, and some even grow by about 10 percent. Expansion on unlucky data is possible for almost any adaptive method.

### Limits Welch admits

- Poor compression at the beginning of each message.
- A table of fixed size (4096 entries) stops learning once it is full. If the data changes character halfway, compression gets worse.
- Random-looking data can expand slightly.

## 5. After 1984: both ideas are still everywhere

Neither method was replaced. They got combined and built into the file formats we use daily. Everything in this section is *(background)*, so check it before citing it on a slide.

### Spreading out

- **LZW became a product.** Unix `compress` and the ARC archiver (mid 1980s) used it. GIF images (1987) and TIFF files use LZW, and PDF can use it too.
- **The patent.** Unisys held a US patent on the LZW algorithm (patent 4,558,302). When it started demanding licence fees for GIF software in the 1990s, many developers moved to alternatives. The patent expired in the early 2000s.
- **LZ77-based methods took over general use.** Because of the patent dispute and good results, the sliding-window family from Lempel and Ziv's 1977 paper became dominant for general-purpose compression.

### The combination: DEFLATE

The most widely used lossless method today is **DEFLATE**, used by zip, gzip and PNG. It works in two stages: an LZ77-style step first replaces repeated strings, and then a **Huffman** step codes whatever is left (literals and match lengths) with short and long codes. In other words, our two papers' ideas work together: dictionary matching removes repeats, Huffman removes uneven frequencies.

### Huffman inside lossy formats

Baseline **JPEG** is a lossy format, but its final stage is lossless: it uses Huffman tables to store the quantized coefficients compactly. The lossy part throws information away first. Huffman then packs what remains.

### Where things stand now

- Newer general-purpose compressors (for example zstd and Brotli) still build on LZ77-style matching and add faster, more precise entropy coders. Many have moved from Huffman to arithmetic-like coders (such as ANS) that avoid the whole-bit limit.
- For lossless images, **PNG** first applies a *prediction filter* (each pixel is predicted from its neighbours, and only the difference is stored) and then runs DEFLATE. Prediction matters because neighbouring pixels are strongly related, and plain Huffman or LZW on raw pixel values cannot see that relationship.
- Specialised lossless image formats (such as lossless WebP or JPEG-LS) go further with better prediction and context modelling.

The lesson for our project: Huffman and LZW are the *foundations*. They are not the state of the art, but they are the parts every modern compressor is built from, and PNG is a fair real-world yardstick to compare against.

## 6. Our project: what we actually do

We write our own Huffman and LZW encoder and decoder for grayscale images, and measure them against the image's entropy and against PNG. The assignment is paper 8 in the course sign-up sheet. The sentence there is: *encoder and decoder for both, with bit packing; compression ratio against image entropy; compare with PNG*.

### What the instructor asks for

| Part | When | What we deliver |
| --- | --- | --- |
| Progress report | Week 10 (about Nov 16 to 22) | 1 to 2 pages: what works, what is left, problems, plan to Week 14. Test images should load and the first step of the method should run. |
| Presentation (25%) | Week 13 | 8 minutes plus 2 for questions. Every member speaks. Slides as PDF on Google Classroom before the session. Last slide discloses our AI use. |
| Final project (40%) | Week 14 | A Jupyter notebook that runs top to bottom, requirements.txt, test images, a PDF report, and a live demo where anyone can be asked to explain any function. |

Week dates are an estimate from the sign-up deadline; the exact dates are not in the course files.

### The terms in the assignment, in plain words

- **Encoder and decoder.** The encoder turns an image into a compressed stream of bits. The decoder turns it back. We prove it is lossless by checking that decoded pixels equal the original pixels exactly.
- **Bit packing.** Codes are not whole bytes (Huffman codes might be 2 or 7 bits, LZW codes 9 to 12 bits). Files are stored in bytes, so we must glue the bits together and write them out as bytes. Without this step we would only be *counting* bits, not really compressing. It also means we have to count the extra bytes the file needs (for example the Huffman table or header), which is part of the honest compressed size.
- **Compression ratio.** Original size divided by compressed size, as Welch defines it. A ratio of 2.0 means half the size.
- **Comparison with entropy.** We compute the image's entropy H from its histogram. It is the floor for coding pixels one at a time. Huffman's average length should land between H and H + 1.
- **Comparison with PNG.** We encode the same image with `cv2.imencode` (the library function the course lists) and compare file sizes. PNG is for comparison only, and we are not allowed to use any library function that implements Huffman or LZW for our own coders.

### What the notebook must show

1. Our implementation, step by step, with short Markdown explanations.
2. At least one result from the papers, reproduced. Good candidates: Huffman's 13-message example (average length 3.42, his Tables I and II) and Welch's compression ratios by data type (his Table 2), which we can compare against our results on different kinds of images and data.
3. Experiments on our own images, including the effect of the method's main parameters.
4. A comparison against PNG through `cv2.imencode`.

## Glossary and references

### Glossary

| Term | Plain meaning |
| --- | --- |
| Bit | One 0 or 1. The smallest unit of data. |
| Symbol | One unit we code: a character, or for us a pixel value from 0 to 255. |
| Code (codeword) | The string of bits we assign to a symbol. |
| Lossless | Decoding returns exactly the original data. |
| Redundancy | Predictable or repeated structure in data. It is what compression removes. |
| Probability P(i) | How often symbol i occurs, as a fraction of the total. |
| Entropy H | The average information per symbol in bits. The floor for coding symbols one at a time. |
| Average code length L_av | The sum of each code's length times its probability. |
| Prefix-free code | No code is the start of another, so decoding is unambiguous. |
| Huffman tree | The binary tree built by repeatedly merging the two least probable symbols. |
| Static vs adaptive | Static: statistics are fixed first. Adaptive: the model updates while data is processed. |
| String table (dictionary) | LZW's phrasebook mapping strings to code numbers. |
| Greedy parsing | Always taking the longest string already in the table. |
| Compression ratio | Original size divided by compressed size. |
| Bit packing | Gluing variable-length codes into whole bytes for storage. |
| Histogram | Count of how many times each pixel value occurs. |
| DEFLATE | The lossless method behind zip, gzip and PNG: LZ77-style matching plus Huffman coding. |

### What we read and what we looked up

Both papers are in the repository folder `Research_Papers`. The Welch PDF is a scan with imperfect text recognition, so check formulas and table values against the page images.

- D. A. Huffman, "A Method for the Construction of Minimum-Redundancy Codes," *Proc. IRE*, vol. 40, no. 9, pp. 1098 to 1101, 1952. [doi.org/10.1109/JRPROC.1952.273898](https://doi.org/10.1109/JRPROC.1952.273898)
- T. A. Welch, "A Technique for High-Performance Data Compression," *Computer*, vol. 17, no. 6, pp. 8 to 19, June 1984. [doi.org/10.1109/MC.1984.1659158](https://doi.org/10.1109/MC.1984.1659158)
- INF605 Final Project Brief and Final Project Groups sheet (Fall 2026), from `Teachers_Materials`.
- Other facts about Unisys' LZW patent and the LZ78 lineage were checked against [FOLDOC's entry](https://www.irt.org/foldoc/Lempel-Ziv%20Welch%20compression.htm) and a [data compression book chapter](https://cmd.inp.nsk.su/old/cmd2/manuals/programming/compression/ch07/208-209.html) found by search.

### Background references to cite properly before the presentation

Statements marked *(background)* come from general knowledge, not from files we read. We should confirm each one against a primary source before it appears on a slide. Sources named in the papers themselves: C. E. Shannon, "A mathematical theory of communication," *Bell System Technical Journal*, 1948 (cited by Huffman). J. Ziv and A. Lempel, 1977 and 1978 (cited by Welch). Also the image compression chapter of Gonzalez and Woods, which the course uses.
