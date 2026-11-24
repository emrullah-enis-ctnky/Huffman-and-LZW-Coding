# Background part: notes, slide plan and likely questions

Group 8, INF605. This file supports `notebooks/background.ipynb`. It has three parts: the formulas to know by heart, a plan for the background slides, and questions the instructor might ask, with answers.

Sources: Gonzalez & Woods, *Digital Image Processing*, 4th ed., Chapter 8, Section 8.1 (the book); Shannon, "A mathematical theory of communication," *Bell System Technical Journal*, 1948; Huffman, *Proc. IRE*, 1952.

## 1. Formulas to know

| Name | Formula | In words |
| --- | --- | --- |
| Compression ratio | $C = b / b'$ | original bits divided by compressed bits |
| Relative redundancy | $R = 1 - 1/C$ | share of the original bits that was not needed |
| Self-information | $I = -\log_2 P$ | surprise of an event with probability $P$ |
| Entropy | $H = -\sum_j P_j \log_2 P_j$ | average surprise per symbol |
| Average code length | $L_{avg} = \sum_k P_k\, l_k$ | code-word lengths weighted by probability |
| Kraft inequality | $\sum_k 2^{-l_k} \le 1$ | a prefix-free code with these lengths exists |
| Shannon's first theorem | $H \le L_{avg} < H + 1$ | best one-word-per-symbol code, within one bit of $H$ |
| Blocks of $n$ symbols | $L_n / n < H + 1/n$ | gets closer to $H$, at the price of $2^{8n}$ words for pixels |

Reproduced from the book (notebook, Section 1): Example 8.1 gives $L_{avg} = 1.81$, $C = 4.42$, $R = 0.774$. The book prints $H = 1.6614$ in Example 8.2; with the rounded probabilities of its Table 8.1 we get $1.6637$.

## 2. Slide plan (about 1.5 minutes in total)

The whole talk is 8 minutes for five speakers, so background gets two slides. The depth stays in the notebook and in this file for questions.

**Slide 1: The problem**
- An image is 8 bits per pixel, but most of those bits are not needed. $C = b/b'$ and $R = 1 - 1/C$.
- Three kinds of redundancy: coding, spatial, irrelevant (lossy, not used by us).
- The yardstick: entropy $H$. One word per pixel value cannot average below $H$.
- Our four test images need between 5.9 and 7.9 bits per pixel at best with one word per pixel.

**Slide 2: Why simple methods are not enough**
- Fixed 8-bit code: ignores the histogram.
- Run-length coding: 128:1 on constant rows, but about 0.5 on real photos (the data doubles).
- Shannon–Fano (top-down): not always optimal (2.31 against 2.30 bits in our example).
- Hand-over: Huffman 1952 fixes the code optimally, bottom-up. LZW 1984 needs no probabilities and finds repeated patterns.

## 3. Likely questions and answers

**Why can a one-word-per-symbol code never beat the entropy?**
Write $q_k = 2^{-l_k}$. Then $L_{avg} - H = \sum_k P_k \log_2 (P_k / q_k)$. By the Gibbs inequality (the log-sum inequality) this is at least $-\log_2 \sum_k q_k$, and the Kraft inequality says $\sum_k q_k \le 1$, so it is at least 0.

**Why is there always a code with $L_{avg} < H + 1$?**
Choose $l_k = \lceil -\log_2 P_k \rceil$. These lengths satisfy Kraft, because $\sum 2^{-l_k} \le \sum P_k = 1$, so a prefix-free code exists. Also $l_k < -\log_2 P_k + 1$, so averaging gives $L_{avg} < H + 1$.

**What does the Kraft inequality mean, and why is it exactly the right condition?**
In a binary tree, a code word of length $l$ occupies a fraction $2^{-l}$ of all leaves below it. Prefix-free means the occupied regions do not overlap, so the fractions add up to at most 1. In the other direction, if the sum is at most 1, sort the lengths from short to long and give each word the next free place in the tree.

**When is $L_{avg}$ exactly equal to $H$?**
When every probability is a power of $1/2$ (for example $1/2, 1/4, 1/8, 1/8$). Then $-\log_2 P_k$ is a whole number and nothing needs rounding up.

**What is the entropy of a constant image? Of an image with all 256 values equally likely?**
A constant image has $H = 0$ (no surprise). A uniform histogram has $H = 8$ bits, so no coding redundancy exists. (The book's Fig. 8.1(b) has $H = 8$ and still compresses 128:1 with run lengths, because of spatial redundancy.)

**Does $H$ depend on where the pixels are?**
No. $H$ comes from the histogram only. If you shuffle the pixels, $H$ stays the same but the spatial redundancy is destroyed. This is why $H$ of single pixels is not the final limit for real images.

**Why can PNG beat the single-pixel entropy?**
PNG first predicts each pixel from its neighbours (its filters) and codes the differences, then applies DEFLATE (LZ77 plus Huffman coding). In our notebook the horizontal differences have lower entropy than the pixels (moon 6.71 to 5.51, desert 5.91 to 4.77, mandrill 7.36 to 6.35 bits). The texture mosaic gains almost nothing (7.91 to 7.82).

**Is the entropy of the differences a fair estimate?**
It is an estimate, not a bound. The mapping (pixel to difference) is reversible once the first column is stored, so no information is lost, but the first column costs a few bits and the difference range is $-255$ to $255$.

**What is the biggest compression ratio a one-word-per-pixel code can reach?**
$8/H$. For our images that is about 1.19 (moon), 1.35 (desert), 1.09 (mandrill) and 1.01 (texture mosaic). Real gains need neighbours or repeated patterns, not only a better code.

**Why not code blocks of pixels, since that gets closer to $H$?**
For 256 gray values, blocks of 2 pixels need $256^2 = 65{,}536$ code words and blocks of 3 need $16{,}777{,}216$. The table becomes too big, and it must be known by the decoder.

**Why does run-length coding fail on photographs?**
Neighbouring pixels are similar but rarely identical, so almost every pixel starts a new run. Each run costs 16 bits (value plus length) for a single pixel, which is twice the original 8 bits.

**Why is Shannon–Fano not optimal, and why is Huffman?**
Shannon–Fano decides each split by balancing probabilities in the current group, without looking ahead. Huffman builds from the bottom: the two least probable symbols can always be given the longest, equal-length words and made siblings. Merging them gives a smaller problem with one symbol fewer, which is solved the same way, so the final code is optimal.

**Which part of the pipeline is lossy?**
The quantizer. The book's model is mapper, then quantizer, then symbol coder. Our methods are lossless, so there is no quantizer and the decoded image is identical to the original.

**What is the difference between data and information?**
Data are the bits we store. Information is the content they carry. The same information can be stored with different amounts of data, and the unnecessary part is the redundancy.

## 4. Open points
- The book's printed $H = 1.6614$ (Example 8.2) differs slightly from our $1.6637$ for the rounded probabilities in its table.
- Check the reference for Shannon–Fano (Fano 1949) before it goes on a slide.
