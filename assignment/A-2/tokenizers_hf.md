# Tokenizers

Fast State-of-the-art tokenizers, optimized for both research and
production

[🤗 Tokenizers](https://github.com/huggingface/tokenizers) provides an
implementation of today's most used tokenizers, with a focus on
performance and versatility. These tokenizers are also used in [🤗 Transformers](https://github.com/huggingface/transformers).

# Main features:

- Train new vocabularies and tokenize, using today's most used tokenizers.
- Extremely fast (both training and tokenization), thanks to the Rust implementation. Takes less than 20 seconds to tokenize a GB of text on a server's CPU.
- Easy to use, but also extremely versatile.
- Designed for both research and production.
- Full alignment tracking. Even with destructive normalization, it's always possible to get the part of the original sentence that corresponds to any token.
- Does all the pre-processing: Truncation, Padding, add the special tokens your model needs.

# Quicktour

Let's have a quick look at the 🤗 Tokenizers library features. The
library provides an implementation of today's most used tokenizers that
is both easy to use and blazing fast.

## Build a tokenizer from scratch

To illustrate how fast the 🤗 Tokenizers library is, let's train a new
tokenizer on [wikitext-103](https://www.salesforce.com/blog/the-wikitext-long-term-dependency-language-modeling-dataset/)
(516M of text) in just a few seconds. First things first, you will need
to download this dataset and unzip it with:

``` bash
wget https://s3.amazonaws.com/research.metamind.io/wikitext/wikitext-103-raw-v1.zip
unzip wikitext-103-raw-v1.zip
```

### Training the tokenizer

In this tour, we will build and train a Byte-Pair Encoding (BPE)
tokenizer. For more information about the different type of tokenizers,
check out this [guide](https://huggingface.co/transformers/tokenizer_summary.html) in
the 🤗 Transformers documentation. Here, training the tokenizer means it
will learn merge rules by:

-   Start with all the characters present in the training corpus as
    tokens.
-   Identify the most common pair of tokens and merge it into one token.
-   Repeat until the vocabulary (e.g., the number of tokens) has reached
    the size we want.

The main API of the library is the `class` `Tokenizer`, here is how
we instantiate one with a BPE model:

```python
from tokenizers import Tokenizer
from tokenizers.models import BPE
tokenizer = Tokenizer(BPE(unk_token="[UNK]"))
```

```rust
use tokenizers::models::bpe::BPE;
let mut tokenizer: TokenizerImpl = TokenizerImpl::new(
    BPE::builder()
        .unk_token("[UNK]".to_string())
        .build()
        .unwrap(),
);
```

```js
{ Tokenizer } = require('tokenizers')
{ BPE } = require('tokenizers')
tokenizer = new Tokenizer(BPE.init({}, [], { unkToken: '[UNK]' }))
```

To train our tokenizer on the wikitext files, we will need to
instantiate a [trainer]{.title-ref}, in this case a
`BpeTrainer`

```python
from tokenizers.trainers import BpeTrainer
trainer = BpeTrainer(special_tokens=["[UNK]", "[CLS]", "[SEP]", "[PAD]", "[MASK]"])
```

```rust
use tokenizers::models::bpe::BpeTrainer;
let mut trainer = BpeTrainer::builder()
    .special_tokens(vec![
        AddedToken::from("[UNK]", true),
        AddedToken::from("[CLS]", true),
        AddedToken::from("[SEP]", true),
        AddedToken::from("[PAD]", true),
        AddedToken::from("[MASK]", true),
    ])
    .build();
```

```js
{ bpeTrainer } = require('tokenizers')
trainer = bpeTrainer({
ecialTokens: ['[UNK]', '[CLS]', '[SEP]', '[PAD]', '[MASK]'],
```

We can set the training arguments like `vocab_size` or `min_frequency` (here
left at their default values of 30,000 and 0) but the most important
part is to give the `special_tokens` we
plan to use later on (they are not used at all during training) so that
they get inserted in the vocabulary.

The order in which you write the special tokens list matters: here `"[UNK]"` will get the ID 0,
`"[CLS]"` will get the ID 1 and so forth.

We could train our tokenizer right now, but it wouldn't be optimal.
Without a pre-tokenizer that will split our inputs into words, we might
get tokens that overlap several words: for instance we could get an
`"it is"` token since those two words
often appear next to each other. Using a pre-tokenizer will ensure no
token is bigger than a word returned by the pre-tokenizer. Here we want
to train a subword BPE tokenizer, and we will use the easiest
pre-tokenizer possible by splitting on whitespace.

```python
from tokenizers.pre_tokenizers import Whitespace
tokenizer.pre_tokenizer = Whitespace()
```

```rust
use tokenizers::pre_tokenizers::whitespace::Whitespace;
tokenizer.with_pre_tokenizer(Some(Whitespace {}));
```

```js
{ whitespacePreTokenizer } = require('tokenizers')
nizer.setPreTokenizer(whitespacePreTokenizer())
```

Now, we can just call the `Tokenizer.train` method with any list of files we want to use:

```python
files = [f"data/wikitext-103-raw/wiki.{split}.raw" for split in ["test", "train", "valid"]]
tokenizer.train(files, trainer)
```

```rust
let files = vec![
    "data/wikitext-103-raw/wiki.train.raw".into(),
    "data/wikitext-103-raw/wiki.test.raw".into(),
    "data/wikitext-103-raw/wiki.valid.raw".into(),
];
tokenizer.train_from_files(&mut trainer, files)?;
```

```js
files = ['test', 'train', 'valid'].map((split) => `data/wikitext-103-raw/wiki.${split}.raw`)
nizer.train(files, trainer)
```

This should only take a few seconds to train our tokenizer on the full
wikitext dataset! To save the tokenizer in one file that contains all
its configuration and vocabulary, just use the
`Tokenizer.save` method:

```python
tokenizer.save("data/tokenizer-wiki.json")
```

```rust
tokenizer.save("data/tokenizer-wiki.json", false)?;
```

```js
nizer.save('data/tokenizer-wiki.json')
```

and you can reload your tokenizer from that file with the
`Tokenizer.from_file`
`classmethod`:

```python
tokenizer = Tokenizer.from_file("data/tokenizer-wiki.json")
```

```rust
let mut tokenizer = Tokenizer::from_file("data/tokenizer-wiki.json")?;
```

```js
tokenizer = Tokenizer.fromFile('data/tokenizer-wiki.json')
```

### Using the tokenizer

Now that we have trained a tokenizer, we can use it on any text we want
with the `Tokenizer.encode` method:

```python
output = tokenizer.encode("Hello, y'all! How are you 😁 ?")
```

```rust
let output = tokenizer.encode("Hello, y'all! How are you 😁 ?", true)?;
```

```js
output = await tokenizer.encode("Hello, y'all! How are you 😁 ?")
```

This applied the full pipeline of the tokenizer on the text, returning
an `Encoding` object. To learn more
about this pipeline, and how to apply (or customize) parts of it, check out [this page](https://github.com/huggingface/tokenizers/blob/main/docs/source-doc-builder/pipeline.mdx).

This `Encoding` object then has all the
attributes you need for your deep learning model (or other). The
`tokens` attribute contains the
segmentation of your text in tokens:

```python
print(output.tokens)
# ["Hello", ",", "y", "'", "all", "!", "How", "are", "you", "[UNK]", "?"]
```

```rust
println!("{:?}", output.get_tokens());
// ["Hello", ",", "y", "'", "all", "!", "How", "are", "you", "[UNK]", "?",]
```

```js
ole.log(output.getTokens())
"Hello", ",", "y", "'", "all", "!", "How", "are", "you", "[UNK]", "?"]
```

Similarly, the `ids` attribute will
contain the index of each of those tokens in the tokenizer's
vocabulary:

```python
print(output.ids)
# [27253, 16, 93, 11, 5097, 5, 7961, 5112, 6218, 0, 35]
```

```rust
println!("{:?}", output.get_ids());
// [27253, 16, 93, 11, 5097, 5, 7961, 5112, 6218, 0, 35]
```

```js
ole.log(output.getIds())
27253, 16, 93, 11, 5097, 5, 7961, 5112, 6218, 0, 35]
```

An important feature of the 🤗 Tokenizers library is that it comes with
full alignment tracking, meaning you can always get the part of your
original sentence that corresponds to a given token. Those are stored in
the `offsets` attribute of our
`Encoding` object. For instance, let's
assume we would want to find back what caused the
`"[UNK]"` token to appear, which is the
token at index 9 in the list, we can just ask for the offset at the
index:

```python
print(output.offsets[9])
# (26, 27)
```

```rust
println!("{:?}", output.get_offsets()[9]);
// (26, 30)
```

```js
offsets = output.getOffsets()
ole.log(offsets[9])
26, 27)
```

and those are the indices that correspond to the emoji in the original
sentence:

```python
sentence = "Hello, y'all! How are you 😁 ?"
sentence[26:27]
# "😁"
```

```rust
let sentence = "Hello, y'all! How are you 😁 ?";
println!("{}", &sentence[26..30]);
// "😁"
```

```js
{ slice } = require('tokenizers')
sentence = "Hello, y'all! How are you 😁 ?"
[start, end] = offsets[9]
ole.log(slice(sentence, start, end))
😁"
```

### Post-processing

We might want our tokenizer to automatically add special tokens, like
`"[CLS]"` or `"[SEP]"`. To do this, we use a post-processor.
`TemplateProcessing` is the most
commonly used, you just have to specify a template for the processing of
single sentences and pairs of sentences, along with the special tokens
and their IDs.

When we built our tokenizer, we set `"[CLS]"` and `"[SEP]"` in positions 1
and 2 of our list of special tokens, so this should be their IDs. To
double-check, we can use the `Tokenizer.token_to_id` method:

```python
tokenizer.token_to_id("[SEP]")
# 2
```

```rust
println!("{}", tokenizer.token_to_id("[SEP]").unwrap());
// 2
```

```js
ole.log(tokenizer.tokenToId('[SEP]'))
```

Here is how we can set the post-processing to give us the traditional
BERT inputs:

```python
from tokenizers.processors import TemplateProcessing
tokenizer.post_processor = TemplateProcessing(
    single="[CLS] $A [SEP]",
    pair="[CLS] $A [SEP] $B:1 [SEP]:1",
    special_tokens=[
        ("[CLS]", tokenizer.token_to_id("[CLS]")),
        ("[SEP]", tokenizer.token_to_id("[SEP]")),
    ],
)
```

```rust
use tokenizers::processors::template::TemplateProcessing;
let special_tokens = vec![
    ("[CLS]", tokenizer.token_to_id("[CLS]").unwrap()),
    ("[SEP]", tokenizer.token_to_id("[SEP]").unwrap()),
];
tokenizer.with_post_processor(Some(
    TemplateProcessing::builder()
        .try_single("[CLS] $A [SEP]")
        .unwrap()
        .try_pair("[CLS] $A [SEP] $B:1 [SEP]:1")
        .unwrap()
        .special_tokens(special_tokens)
        .build()?,
));
```

```js
{ templateProcessing } = require('tokenizers')
nizer.setPostProcessor(
mplateProcessing('[CLS] $A [SEP]', '[CLS] $A [SEP] $B:1 [SEP]:1', [
['[CLS]', tokenizer.tokenToId('[CLS]')],
['[SEP]', tokenizer.tokenToId('[SEP]')],
,
```

Let's go over this snippet of code in more details. First we specify
the template for single sentences: those should have the form
`"[CLS] $A [SEP]"` where
`$A` represents our sentence.

Then, we specify the template for sentence pairs, which should have the
form `"[CLS] $A [SEP] $B [SEP]"` where
`$A` represents the first sentence and
`$B` the second one. The
`:1` added in the template represent the `type IDs` we want for each part of our input: it defaults
to 0 for everything (which is why we don't have
`$A:0`) and here we set it to 1 for the
tokens of the second sentence and the last `"[SEP]"` token.

Lastly, we specify the special tokens we used and their IDs in our
tokenizer's vocabulary.

To check out this worked properly, let's try to encode the same
sentence as before:

```python
output = tokenizer.encode("Hello, y'all! How are you 😁 ?")
print(output.tokens)
# ["[CLS]", "Hello", ",", "y", "'", "all", "!", "How", "are", "you", "[UNK]", "?", "[SEP]"]
```

```rust
let output = tokenizer.encode("Hello, y'all! How are you 😁 ?", true)?;
println!("{:?}", output.get_tokens());
// ["[CLS]", "Hello", ",", "y", "'", "all", "!", "How", "are", "you", "[UNK]", "?", "[SEP]"]
```

```js
output = await tokenizer.encode("Hello, y'all! How are you 😁 ?")
ole.log(output.getTokens())
"[CLS]", "Hello", ",", "y", "'", "all", "!", "How", "are", "you", "[UNK]", "?", "[SEP]"]
```

To check the results on a pair of sentences, we just pass the two
sentences to `Tokenizer.encode`:

```python
output = tokenizer.encode("Hello, y'all!", "How are you 😁 ?")
print(output.tokens)
# ["[CLS]", "Hello", ",", "y", "'", "all", "!", "[SEP]", "How", "are", "you", "[UNK]", "?", "[SEP]"]
```

```rust
let output = tokenizer.encode(("Hello, y'all!", "How are you 😁 ?"), true)?;
println!("{:?}", output.get_tokens());
// ["[CLS]", "Hello", ",", "y", "'", "all", "!", "[SEP]", "How", "are", "you", "[UNK]", "?", "[SEP]"]
```

```js
output = await tokenizer.encode("Hello, y'all!", 'How are you 😁 ?')
ole.log(output.getTokens())
"[CLS]", "Hello", ",", "y", "'", "all", "!", "[SEP]", "How", "are", "you", "[UNK]", "?", "[SEP]"]
```

You can then check the type IDs attributed to each token is correct with

```python
print(output.type_ids)
# [0, 0, 0, 0, 0, 0, 0, 0, 1, 1, 1, 1, 1, 1]
```

```rust
println!("{:?}", output.get_type_ids());
// [0, 0, 0, 0, 0, 0, 0, 0, 1, 1, 1, 1, 1, 1]
```

```js
ole.log(output.getTypeIds())
0, 0, 0, 0, 0, 0, 0, 0, 1, 1, 1, 1, 1, 1]
```

If you save your tokenizer with `Tokenizer.save`, the post-processor will be saved along.

### Encoding multiple sentences in a batch

To get the full speed of the 🤗 Tokenizers library, it's best to
process your texts by batches by using the
`Tokenizer.encode_batch` method:

```python
output = tokenizer.encode_batch(["Hello, y'all!", "How are you 😁 ?"])
```

```rust
let output = tokenizer.encode_batch(vec!["Hello, y'all!", "How are you 😁 ?"], true)?;
```

```js
output = await tokenizer.encodeBatch(["Hello, y'all!", 'How are you 😁 ?'])
```

The output is then a list of `Encoding`
objects like the ones we saw before. You can process together as many
texts as you like, as long as it fits in memory.

To process a batch of sentences pairs, pass two lists to the
`Tokenizer.encode_batch` method: the
list of sentences A and the list of sentences B:

```python
output = tokenizer.encode_batch(
    [["Hello, y'all!", "How are you 😁 ?"], ["Hello to you too!", "I'm fine, thank you!"]]
)
```

```rust
let output = tokenizer.encode_batch(
    vec![
        ("Hello, y'all!", "How are you 😁 ?"),
        ("Hello to you too!", "I'm fine, thank you!"),
    ],
    true,
)?;
```

```js
ar output = await tokenizer.encodeBatch(
   [["Hello, y'all!", "How are you 😁 ?"], ["Hello to you too!", "I'm fine, thank you!"]]
;
```

When encoding multiple sentences, you can automatically pad the outputs
to the longest sentence present by using
`Tokenizer.enable_padding`, with the
`pad_token` and its ID (which we can
double-check the id for the padding token with
`Tokenizer.token_to_id` like before):

```python
tokenizer.enable_padding(pad_id=3, pad_token="[PAD]")
```

```rust
use tokenizers::PaddingParams;
tokenizer.with_padding(Some(PaddingParams {
    pad_id: 3,
    pad_token: "[PAD]".to_string(),
    ..PaddingParams::default()
}));
```

```js
nizer.setPadding({ padId: 3, padToken: '[PAD]' })
```

We can set the `direction` of the padding
(defaults to the right) or a given `length` if we want to pad every sample to that specific number (here
we leave it unset to pad to the size of the longest text).

```python
output = tokenizer.encode_batch(["Hello, y'all!", "How are you 😁 ?"])
print(output[1].tokens)
# ["[CLS]", "How", "are", "you", "[UNK]", "?", "[SEP]", "[PAD]"]
```

```rust
let output = tokenizer.encode_batch(vec!["Hello, y'all!", "How are you 😁 ?"], true)?;
println!("{:?}", output[1].get_tokens());
// ["[CLS]", "How", "are", "you", "[UNK]", "?", "[SEP]", "[PAD]"]
```

```js
output = await tokenizer.encodeBatch(["Hello, y'all!", 'How are you 😁 ?'])
ole.log(output[1].getTokens())
"[CLS]", "How", "are", "you", "[UNK]", "?", "[SEP]", "[PAD]"]
```

In this case, the `attention mask` generated by the
tokenizer takes the padding into account:

```python
print(output[1].attention_mask)
# [1, 1, 1, 1, 1, 1, 1, 0]
```

```rust
println!("{:?}", output[1].get_attention_mask());
// [1, 1, 1, 1, 1, 1, 1, 0]
```

```js
ole.log(output[1].getAttentionMask())
1, 1, 1, 1, 1, 1, 1, 0]
```

## Pretrained

### Using a pretrained tokenizer

You can load any tokenizer from the Hugging Face Hub as long as a
`tokenizer.json` file is available in the repository.

```python
from tokenizers import Tokenizer

tokenizer = Tokenizer.from_pretrained("bert-base-uncased")
```

### Importing a pretrained tokenizer from legacy vocabulary files

You can also import a pretrained tokenizer directly in, as long as you
have its vocabulary file. For instance, here is how to import the
classic pretrained BERT tokenizer:

```python
from tokenizers import BertWordPieceTokenizer

tokenizer = BertWordPieceTokenizer("bert-base-uncased-vocab.txt", lowercase=True)
```

as long as you have downloaded the file `bert-base-uncased-vocab.txt` with

```bash
wget https://s3.amazonaws.com/models.huggingface.co/bert/bert-base-uncased-vocab.txt
```

# Installation

🤗 Tokenizers is tested on Python 3.5+.

You should install 🤗 Tokenizers in a [virtual environment](https://docs.python.org/3/library/venv.html). If you're
unfamiliar with Python virtual environments, check out the [user
guide](https://packaging.python.org/guides/installing-using-pip-and-virtual-environments/).
Create a virtual environment with the version of Python you're going to
use and activate it.

## Installation with pip

🤗 Tokenizers can be installed using pip as follows:

```bash
pip install tokenizers
```

## Installation from sources

To use this method, you need to have the Rust language installed. You
can follow [the official
guide](https://www.rust-lang.org/learn/get-started) for more
information.

If you are using a unix based OS, the installation should be as simple
as running:

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

Or you can easily update it with the following command:

```bash
rustup update
```

Once rust is installed, we can start retrieving the sources for 🤗
Tokenizers:

```bash
git clone https://github.com/huggingface/tokenizers
```

Then we go into the python bindings folder:

```bash
cd tokenizers/bindings/python
```

At this point you should have your [virtual environment]() already
activated. In order to compile 🤗 Tokenizers, you need to:

```bash
pip install -e .
```

## Crates.io

🤗 Tokenizers is available on [crates.io](https://crates.io/crates/tokenizers).

You just need to add it to your `Cargo.toml`:

```bash
cargo add tokenizers
```

## Installation with npm

You can simply install 🤗 Tokenizers with npm using:

```bash
npm install tokenizers
```

# The tokenization pipeline

When calling `Tokenizer.encode` or
`Tokenizer.encode_batch`, the input
text(s) go through the following pipeline:

-   `normalization`
-   `pre-tokenization`
-   `model`
-   `post-processing`

We'll see in details what happens during each of those steps in detail,
as well as when you want to `decode ` some token ids, and how the 🤗 Tokenizers library allows you
to customize each of those steps to your needs. If you're already
familiar with those steps and want to learn by seeing some code, jump to
`our BERT from scratch example `.

For the examples that require a `Tokenizer` we will use the tokenizer we trained in the
`quicktour`, which you can load with:

```python
from tokenizers import Tokenizer
tokenizer = Tokenizer.from_file("data/tokenizer-wiki.json")
```

```rust
use tokenizers::Tokenizer;
let mut tokenizer = Tokenizer::from_file("data/tokenizer-wiki.json")?;
```

```js
let { Tokenizer } = require("tokenizers");
let tokenizer = Tokenizer.fromFile("data/tokenizer-wiki.json");
```

## Normalization

Normalization is, in a nutshell, a set of operations you apply to a raw
string to make it less random or "cleaner". Common operations include
stripping whitespace, removing accented characters or lowercasing all
text. If you're familiar with [Unicode
normalization](https://unicode.org/reports/tr15), it is also a very
common normalization operation applied in most tokenizers.

Each normalization operation is represented in the 🤗 Tokenizers library
by a `Normalizer`, and you can combine
several of those by using a `normalizers.Sequence`. Here is a normalizer applying NFD Unicode normalization
and removing accents as an example:

```python
from tokenizers import normalizers
from tokenizers.normalizers import NFD, StripAccents
normalizer = normalizers.Sequence([NFD(), StripAccents()])
```

```rust
use tokenizers::normalizers::{
    strip::StripAccents, unicode::NFD, utils::Sequence as NormalizerSequence,
};
let normalizer = NormalizerSequence::new(vec![NFD.into(), StripAccents.into()]);
```

```js
let { sequenceNormalizer, nfdNormalizer, stripAccentsNormalizer } = require("tokenizers");
let normalizer = sequenceNormalizer([nfdNormalizer(), stripAccentsNormalizer()]);
```

You can manually test that normalizer by applying it to any string:

```python
normalizer.normalize_str("Héllò hôw are ü?")
# "Hello how are u?"
```

```rust
use tokenizers::{NormalizedString, Normalizer};
let mut normalized = NormalizedString::from("Héllò hôw are ü?");
normalizer.normalize(&mut normalized)?;
println!("{}", normalized.get());
// "Hello how are u?"
```

```js
let normalized = normalizer.normalizeString("Héllò hôw are ü?")
// "Hello how are u?"
```

When building a `Tokenizer`, you can
customize its normalizer by just changing the corresponding attribute:

```python
tokenizer.normalizer = normalizer
```

```rust
tokenizer.with_normalizer(Some(normalizer));
```

```js
tokenizer.setNormalizer(normalizer)
```

Of course, if you change the way a tokenizer applies normalization, you
should probably retrain it from scratch afterward.

## Pre-Tokenization

Pre-tokenization is the act of splitting a text into smaller objects
that give an upper bound to what your tokens will be at the end of
training. A good way to think of this is that the pre-tokenizer will
split your text into "words" and then, your final tokens will be parts
of those words.

An easy way to pre-tokenize inputs is to split on spaces and
punctuations, which is done by the
`pre_tokenizers.Whitespace`
pre-tokenizer:

```python
from tokenizers.pre_tokenizers import Whitespace
pre_tokenizer = Whitespace()
pre_tokenizer.pre_tokenize_str("Hello! How are you? I'm fine, thank you.")
# [("Hello", (0, 5)), ("!", (5, 6)), ("How", (7, 10)), ("are", (11, 14)), ("you", (15, 18)),
#  ("?", (18, 19)), ("I", (20, 21)), ("'", (21, 22)), ('m', (22, 23)), ("fine", (24, 28)),
#  (",", (28, 29)), ("thank", (30, 35)), ("you", (36, 39)), (".", (39, 40))]
```

```rust
use tokenizers::pre_tokenizers::whitespace::Whitespace;
use tokenizers::{OffsetReferential, OffsetType, PreTokenizedString, PreTokenizer};
let pre_tokenizer = Whitespace {};
let mut pre_tokenized = PreTokenizedString::from("Hello! How are you? I'm fine, thank you.");
pre_tokenizer.pre_tokenize(&mut pre_tokenized)?;
println!(
    "{:?}",
    pre_tokenized.get_splits(OffsetReferential::Original, OffsetType::Byte)
);
// [("Hello", (0, 5), None), ("!", (5, 6), None), ("How", (7, 10), None),
//  ("are", (11, 14), None), ("you", (15, 18), None), ("?", (18, 19), None),
//  ("I", (20, 21), None), ("\'", (21, 22), None), ("m", (22, 23), None),
//  ("fine", (24, 28), None), (",", (28, 29), None), ("thank", (30, 35), None),
//  ("you", (36, 39), None), (".", (39, 40), None)]
```

```js
let { whitespacePreTokenizer } = require("tokenizers");
var preTokenizer = whitespacePreTokenizer();
var preTokenized = preTokenizer.preTokenizeString("Hello! How are you? I'm fine, thank you.");
```

The output is a list of tuples, with each tuple containing one word and
its span in the original sentence (which is used to determine the final
`offsets` of our `Encoding`). Note that splitting on
punctuation will split contractions like `"I'm"` in this example.

You can combine together any `PreTokenizer` together. For instance, here is a pre-tokenizer that will
split on space, punctuation and digits, separating numbers in their
individual digits:

```python
from tokenizers import pre_tokenizers
from tokenizers.pre_tokenizers import Digits
pre_tokenizer = pre_tokenizers.Sequence([Whitespace(), Digits(individual_digits=True)])
pre_tokenizer.pre_tokenize_str("Call 911!")
# [("Call", (0, 4)), ("9", (5, 6)), ("1", (6, 7)), ("1", (7, 8)), ("!", (8, 9))]
```

```rust
use tokenizers::pre_tokenizers::{digits::Digits, sequence::Sequence};
let pre_tokenizer = Sequence::new(vec![Whitespace {}.into(), Digits::new(true).into()]);
let mut pre_tokenized = PreTokenizedString::from("Call 911!");
pre_tokenizer.pre_tokenize(&mut pre_tokenized)?;
println!(
    "{:?}",
    pre_tokenized.get_splits(OffsetReferential::Original, OffsetType::Byte)
);
```

```js
let { sequencePreTokenizer, digitsPreTokenizer } = require("tokenizers");
var preTokenizer = sequencePreTokenizer([whitespacePreTokenizer(), digitsPreTokenizer(true)]);
var preTokenized = preTokenizer.preTokenizeString("Call 911!");
```

As we saw in the `quicktour`, you can
customize the pre-tokenizer of a `Tokenizer` by just changing the corresponding attribute:

```python
tokenizer.pre_tokenizer = pre_tokenizer
```

```rust
tokenizer.with_pre_tokenizer(Some(pre_tokenizer));
```

```js
tokenizer.setPreTokenizer(preTokenizer)
```

Of course, if you change the way the pre-tokenizer, you should probably
retrain your tokenizer from scratch afterward.

## Model

Once the input texts are normalized and pre-tokenized, the
`Tokenizer` applies the model on the
pre-tokens. This is the part of the pipeline that needs training on your
corpus (or that has been trained if you are using a pretrained
tokenizer).

The role of the model is to split your "words" into tokens, using the
rules it has learned. It's also responsible for mapping those tokens to
their corresponding IDs in the vocabulary of the model.

This model is passed along when initializing the
`Tokenizer` so you already know how to
customize this part. Currently, the 🤗 Tokenizers library supports:

-   `models.BPE`
-   `models.Unigram`
-   `models.WordLevel`
-   `models.WordPiece`

For more details about each model and its behavior, you can check
[here](components#models)

## Post-Processing

Post-processing is the last step of the tokenization pipeline, to
perform any additional transformation to the
`Encoding` before it's returned, like
adding potential special tokens.

As we saw in the quick tour, we can customize the post processor of a
`Tokenizer` by setting the
corresponding attribute. For instance, here is how we can post-process
to make the inputs suitable for the BERT model:

```python
from tokenizers.processors import TemplateProcessing
tokenizer.post_processor = TemplateProcessing(
    single="[CLS] $A [SEP]",
    pair="[CLS] $A [SEP] $B:1 [SEP]:1",
    special_tokens=[("[CLS]", 1), ("[SEP]", 2)],
)
```

```rust
use tokenizers::processors::template::TemplateProcessing;
tokenizer.with_post_processor(Some(
    TemplateProcessing::builder()
        .try_single("[CLS] $A [SEP]")
        .unwrap()
        .try_pair("[CLS] $A [SEP] $B:1 [SEP]:1")
        .unwrap()
        .special_tokens(vec![("[CLS]", 1), ("[SEP]", 2)])
        .build()
        .unwrap(),
));
```

```js
let { templateProcessing } = require("tokenizers");
tokenizer.setPostProcessor(templateProcessing(
    "[CLS] $A [SEP]",
    "[CLS] $A [SEP] $B:1 [SEP]:1",
    [["[CLS]", 1], ["[SEP]", 2]]
));
```

Note that contrarily to the pre-tokenizer or the normalizer, you don't
need to retrain a tokenizer after changing its post-processor.

## All together: a BERT tokenizer from scratch

Let's put all those pieces together to build a BERT tokenizer. First,
BERT relies on WordPiece, so we instantiate a new
`Tokenizer` with this model:

```python
from tokenizers import Tokenizer
from tokenizers.models import WordPiece
bert_tokenizer = Tokenizer(WordPiece(unk_token="[UNK]"))
```

```rust
use tokenizers::models::wordpiece::WordPiece;
use tokenizers::Tokenizer;
let mut bert_tokenizer = Tokenizer::new(
    WordPiece::builder()
        .unk_token("[UNK]".to_string())
        .build()
        .unwrap(),
);
```

```js
let { Tokenizer } = require("tokenizers");
let { WordPiece } = require("tokenizers");
let bertTokenizer = new Tokenizer(WordPiece.init({}, { unkToken: "[UNK]" }));
```

Then we know that BERT preprocesses texts by removing accents and
lowercasing. We also use a unicode normalizer:

```python
from tokenizers import normalizers
from tokenizers.normalizers import NFD, Lowercase, StripAccents
bert_tokenizer.normalizer = normalizers.Sequence([NFD(), Lowercase(), StripAccents()])
```

```rust
use tokenizers::normalizers::utils::Sequence as NormalizerSequence;
use tokenizers::normalizers::{strip::StripAccents, unicode::NFD, utils::Lowercase};
bert_tokenizer.with_normalizer(Some(NormalizerSequence::new(vec![
    NFD.into(),
    Lowercase.into(),
    StripAccents.into(),
])));
```

```js
let { sequenceNormalizer, lowercaseNormalizer, nfdNormalizer, stripAccentsNormalizer }
    = require("tokenizers");
bertTokenizer.setNormalizer(sequenceNormalizer([
    nfdNormalizer(), lowercaseNormalizer(), stripAccentsNormalizer()
]))
```

The pre-tokenizer is just splitting on whitespace and punctuation:

```python
from tokenizers.pre_tokenizers import Whitespace
bert_tokenizer.pre_tokenizer = Whitespace()
```

```rust
use tokenizers::pre_tokenizers::whitespace::Whitespace;
bert_tokenizer.with_pre_tokenizer(Some(Whitespace {}));
```

```js
let { whitespacePreTokenizer } = require("tokenizers");
bertTokenizer.setPreTokenizer(whitespacePreTokenizer());
```

And the post-processing uses the template we saw in the previous
section:

```python
from tokenizers.processors import TemplateProcessing
bert_tokenizer.post_processor = TemplateProcessing(
    single="[CLS] $A [SEP]",
    pair="[CLS] $A [SEP] $B:1 [SEP]:1",
    special_tokens=[
        ("[CLS]", 1),
        ("[SEP]", 2),
    ],
)
```

```rust
use tokenizers::processors::template::TemplateProcessing;
bert_tokenizer.with_post_processor(Some(
    TemplateProcessing::builder()
        .try_single("[CLS] $A [SEP]")
        .unwrap()
        .try_pair("[CLS] $A [SEP] $B:1 [SEP]:1")
        .unwrap()
        .special_tokens(vec![("[CLS]", 1), ("[SEP]", 2)])
        .build()
        .unwrap(),
));
```

```js
let { templateProcessing } = require("tokenizers");
bertTokenizer.setPostProcessor(templateProcessing(
    "[CLS] $A [SEP]",
    "[CLS] $A [SEP] $B:1 [SEP]:1",
    [["[CLS]", 1], ["[SEP]", 2]]
));
```

We can use this tokenizer and train on it on wikitext like in the
`quicktour`:

```python
from tokenizers.trainers import WordPieceTrainer
trainer = WordPieceTrainer(vocab_size=30522, special_tokens=["[UNK]", "[CLS]", "[SEP]", "[PAD]", "[MASK]"])
files = [f"data/wikitext-103-raw/wiki.{split}.raw" for split in ["test", "train", "valid"]]
bert_tokenizer.train(files, trainer)
bert_tokenizer.save("data/bert-wiki.json")
```

```rust
use tokenizers::models::{wordpiece::WordPieceTrainer, TrainerWrapper};
let mut trainer: TrainerWrapper = WordPieceTrainer::builder()
    .vocab_size(30_522)
    .special_tokens(vec![
        AddedToken::from("[UNK]", true),
        AddedToken::from("[CLS]", true),
        AddedToken::from("[SEP]", true),
        AddedToken::from("[PAD]", true),
        AddedToken::from("[MASK]", true),
    ])
    .build()
    .into();
let files = vec![
    "data/wikitext-103-raw/wiki.train.raw".into(),
    "data/wikitext-103-raw/wiki.test.raw".into(),
    "data/wikitext-103-raw/wiki.valid.raw".into(),
];
bert_tokenizer.train_from_files(&mut trainer, files)?;
bert_tokenizer.save("data/bert-wiki.json", false)?;
```

```js
let { wordPieceTrainer } = require("tokenizers");
let trainer = wordPieceTrainer({
    vocabSize: 30522,
    specialTokens: ["[UNK]", "[CLS]", "[SEP]", "[PAD]", "[MASK]"]
});
let files = ["test", "train", "valid"].map(split => `data/wikitext-103-raw/wiki.${split}.raw`);
bertTokenizer.train(files, trainer);
bertTokenizer.save("data/bert-wiki.json")
```

## Decoding

On top of encoding the input texts, a `Tokenizer` also has an API for decoding, that is converting IDs
generated by your model back to a text. This is done by the methods
`Tokenizer.decode` (for one predicted text) and `Tokenizer.decode_batch` (for a batch of predictions).

The `decoder` will first convert the IDs back to tokens
(using the tokenizer's vocabulary) and remove all special tokens, then
join those tokens with spaces:

```python
output = tokenizer.encode("Hello, y'all! How are you 😁 ?")
print(output.ids)
# [1, 27253, 16, 93, 11, 5097, 5, 7961, 5112, 6218, 0, 35, 2]
tokenizer.decode([1, 27253, 16, 93, 11, 5097, 5, 7961, 5112, 6218, 0, 35, 2])
# "Hello , y ' all ! How are you ?"
```

```rust
let output = tokenizer.encode("Hello, y'all! How are you 😁 ?", true)?;
println!("{:?}", output.get_ids());
// [1, 27253, 16, 93, 11, 5097, 5, 7961, 5112, 6218, 0, 35, 2]
let decoded = tokenizer.decode(
    &[1, 27253, 16, 93, 11, 5097, 5, 7961, 5112, 6218, 0, 35, 2],
    true,
)?;
println!("{decoded}");
// "Hello , y ' all ! How are you ?"
```

```js
let output = await tokenizer.encode("Hello, y'all! How are you 😁 ?");
console.log(output.getIds());
// [1, 27253, 16, 93, 11, 5097, 5, 7961, 5112, 6218, 0, 35, 2]
let decoded = await tokenizer.decode([1, 27253, 16, 93, 11, 5097, 5, 7961, 5112, 6218, 0, 35, 2], true);
// "Hello , y ' all ! How are you ?"
```

If you used a model that added special characters to represent subtokens
of a given "word" (like the `"##"` in
WordPiece) you will need to customize the `decoder` to treat
them properly. If we take our previous `bert_tokenizer` for instance the
default decoding will give:

```python
output = bert_tokenizer.encode("Welcome to the 🤗 Tokenizers library.")
print(output.tokens)
# ["[CLS]", "welcome", "to", "the", "[UNK]", "tok", "##eni", "##zer", "##s", "library", ".", "[SEP]"]
bert_tokenizer.decode(output.ids)
# "welcome to the tok ##eni ##zer ##s library ."
```

```rust
let output = bert_tokenizer.encode("Welcome to the 🤗 Tokenizers library.", true)?;
println!("{:?}", output.get_tokens());
// ["[CLS]", "welcome", "to", "the", "[UNK]", "tok", "##eni", "##zer", "##s", "library", ".", "[SEP]"]
let decoded = bert_tokenizer.decode(output.get_ids(), true)?;
println!("{decoded}");
// "welcome to the tok ##eni ##zer ##s library ."
```

```js
let output = await bertTokenizer.encode("Welcome to the 🤗 Tokenizers library.");
console.log(output.getTokens());
// ["[CLS]", "welcome", "to", "the", "[UNK]", "tok", "##eni", "##zer", "##s", "library", ".", "[SEP]"]
var decoded = await bertTokenizer.decode(output.getIds(), true);
// "welcome to the tok ##eni ##zer ##s library ."
```

But by changing it to a proper decoder, we get:

```python
from tokenizers import decoders
bert_tokenizer.decoder = decoders.WordPiece()
bert_tokenizer.decode(output.ids)
# "welcome to the tokenizers library."
```

```rust
use tokenizers::decoders::wordpiece::WordPiece as WordPieceDecoder;
bert_tokenizer.with_decoder(Some(WordPieceDecoder::default()));
let decoded = bert_tokenizer.decode(output.get_ids(), true)?;
// "welcome to the tokenizers library."
```

```js
let { wordPieceDecoder } = require("tokenizers");
bertTokenizer.setDecoder(wordPieceDecoder());
var decoded = await bertTokenizer.decode(output.getIds(), true);
// "welcome to the tokenizers library."
```

# Components

When building a Tokenizer, you can attach various types of components to
this Tokenizer in order to customize its behavior. This page lists most
provided components.

## Normalizers

A `Normalizer` is in charge of pre-processing the input string in order
to normalize it as relevant for a given use case. Some common examples
of normalization are the Unicode normalization algorithms (NFD, NFKD,
NFC & NFKC), lowercasing etc... The specificity of `tokenizers` is that
we keep track of the alignment while normalizing. This is essential to
allow mapping from the generated tokens back to the input text.

The `Normalizer` is optional.

| Name | Description | Example |
| :--- | :--- | :--- |
| NFD | NFD unicode normalization |  |
| NFKD | NFKD unicode normalization |  |
| NFC | NFC unicode normalization |  |
| NFKC | NFKC unicode normalization |  |
| Lowercase | Replaces all uppercase to lowercase | Input: `HELLO ὈΔΥΣΣΕΎΣ`  Output: `hello`ὀδυσσεύς`  |
| Strip | Removes all whitespace characters on the specified sides (left, right or both) of the input | Input: `"`hi`"`  Output: `"hi"`  |
| StripAccents | Removes all accent symbols in unicode (to be used with NFD for consistency) | Input: `é`  Output: `e`  |
| Replace | Replaces a custom string or regexp and changes it with given content | `Replace("a", "e")` will behave like this:  Input: `"banana"`  Output: `"benene"`  |
| BertNormalizer | Provides an implementation of the Normalizer used in the original BERT. Options that can be set are:  clean_text handle_chinese_chars strip_accents lowercase   |  |
| Sequence | Composes multiple normalizers that will run in the provided order | `Sequence([NFKC(), Lowercase()])` |

| Name | Description | Example |
| :--- | :--- | :--- |
| NFD | NFD unicode normalization |  |
| NFKD | NFKD unicode normalization |  |
| NFC | NFC unicode normalization |  |
| NFKC | NFKC unicode normalization |  |
| Lowercase | Replaces all uppercase to lowercase | Input: `HELLO ὈΔΥΣΣΕΎΣ`  Output: `hello`ὀδυσσεύς`  |
| Strip | Removes all whitespace characters on the specified sides (left, right or both) of the input | Input: `"`hi`"`  Output: `"hi"`  |
| StripAccents | Removes all accent symbols in unicode (to be used with NFD for consistency) | Input: `é`  Output: `e`  |
| Replace | Replaces a custom string or regexp and changes it with given content | `Replace("a", "e")` will behave like this:  Input: `"banana"`  Output: `"benene"`  |
| BertNormalizer | Provides an implementation of the Normalizer used in the original BERT. Options that can be set are:  clean_text handle_chinese_chars strip_accents lowercase   |  |
| Sequence | Composes multiple normalizers that will run in the provided order | `Sequence::new(vec![NFKC, Lowercase])` |

| Name | Description | Example |
| :--- | :--- | :--- |
| NFD | NFD unicode normalization |  |
| NFKD | NFKD unicode normalization |  |
| NFC | NFC unicode normalization |  |
| NFKC | NFKC unicode normalization |  |
| Lowercase | Replaces all uppercase to lowercase | Input: `HELLO ὈΔΥΣΣΕΎΣ`  Output: `hello`ὀδυσσεύς`  |
| Strip | Removes all whitespace characters on the specified sides (left, right or both) of the input | Input: `"`hi`"`  Output: `"hi"`  |
| StripAccents | Removes all accent symbols in unicode (to be used with NFD for consistency) | Input: `é`  Output: `e`  |
| Replace | Replaces a custom string or regexp and changes it with given content | `Replace("a", "e")` will behave like this:  Input: `"banana"`  Output: `"benene"`  |
| BertNormalizer | Provides an implementation of the Normalizer used in the original BERT. Options that can be set are:  cleanText handleChineseChars stripAccents lowercase   |  |
| Sequence | Composes multiple normalizers that will run in the provided order | |

## Pre-tokenizers

The `PreTokenizer` takes care of splitting the input according to a set
of rules. This pre-processing lets you ensure that the underlying
`Model` does not build tokens across multiple "splits". For example if
you don't want to have whitespaces inside a token, then you can have a
`PreTokenizer` that splits on these whitespaces.

You can easily combine multiple `PreTokenizer` together using a
`Sequence` (see below). The `PreTokenizer` is also allowed to modify the
string, just like a `Normalizer` does. This is necessary to allow some
complicated algorithms that require to split before normalizing (e.g.
the ByteLevel)

| Name | Description | Example |
| :--- | :--- | :--- |
| ByteLevel | Splits on whitespaces while remapping all the bytes to a set of visible characters. This technique as been introduced by OpenAI with GPT-2 and has some more or less nice properties:  Since it maps on bytes, a tokenizer using this only requires **256** characters as initial alphabet (the number of values a byte can have), as opposed to the 130,000+ Unicode characters. A consequence of the previous point is that it is absolutely unnecessary to have an unknown token using this since we can represent anything with 256 tokens (Youhou!! 🎉🎉) For non ascii characters, it gets completely unreadable, but it works nonetheless!  | Input: `"Hello my friend, how are you?"`  Output: `"Hello", "Ġmy", Ġfriend", ",", "Ġhow", "Ġare", "Ġyou", "?"`  |
| Whitespace | Splits on word boundaries (using the following regular expression: `\w+&#124;[^\w\s]+` | Input: `"Hello there!"`  Output: `"Hello", "there", "!"`  |
| WhitespaceSplit | Splits on any whitespace character | Input: `"Hello there!"`  Output: `"Hello", "there!"`  |
| Punctuation | Will isolate all punctuation characters | Input: `"Hello?"`  Output: `"Hello", "?"`  |
| Metaspace | Splits on whitespaces and replaces them with a special char “▁” (U+2581) | Input: `"Hello there"`  Output: `"Hello", "▁there"`  |
| CharDelimiterSplit | Splits on a given character | Example with `x`:  Input: `"Helloxthere"`  Output: `"Hello", "there"`  |
| Digits | Splits the numbers from any other characters. | Input: `"Hello123there"`   Output: ``"Hello", "123", "there"``  |
| Split | Versatile pre-tokenizer that splits on provided pattern and according to provided behavior. The pattern can be inverted if necessary.  pattern should be either a custom string or regexp. behavior should be one of: removedisolatedmerged_with_previousmerged_with_nextcontiguous invert should be a boolean flag.  | Example with pattern = ` `, behavior = `"isolated"`, invert = `False`:  Input: `"Hello, how are you?"`  Output: `"Hello,", " ", "how", " ", "are", " ", "you?"` |
| Sequence | Lets you compose multiple `PreTokenizer` that will be run in the given order | `Sequence([Punctuation(), WhitespaceSplit()])` |

| Name | Description | Example |
| :--- | :--- | :--- |
| ByteLevel | Splits on whitespaces while remapping all the bytes to a set of visible characters. This technique as been introduced by OpenAI with GPT-2 and has some more or less nice properties:  Since it maps on bytes, a tokenizer using this only requires **256** characters as initial alphabet (the number of values a byte can have), as opposed to the 130,000+ Unicode characters. A consequence of the previous point is that it is absolutely unnecessary to have an unknown token using this since we can represent anything with 256 tokens (Youhou!! 🎉🎉) For non ascii characters, it gets completely unreadable, but it works nonetheless!  | Input: `"Hello my friend, how are you?"`  Output: `"Hello", "Ġmy", Ġfriend", ",", "Ġhow", "Ġare", "Ġyou", "?"`  |
| Whitespace | Splits on word boundaries (using the following regular expression: `\w+&#124;[^\w\s]+` | Input: `"Hello there!"`  Output: `"Hello", "there", "!"`  |
| WhitespaceSplit | Splits on any whitespace character | Input: `"Hello there!"`  Output: `"Hello", "there!"`  |
| Punctuation | Will isolate all punctuation characters | Input: `"Hello?"`  Output: `"Hello", "?"`  |
| Metaspace | Splits on whitespaces and replaces them with a special char “▁” (U+2581) | Input: `"Hello there"`  Output: `"Hello", "▁there"`  |
| CharDelimiterSplit | Splits on a given character | Example with `x`:  Input: `"Helloxthere"`  Output: `"Hello", "there"`  |
| Digits | Splits the numbers from any other characters. | Input: `"Hello123there"`   Output: ``"Hello", "123", "there"``  |
| Split | Versatile pre-tokenizer that splits on provided pattern and according to provided behavior. The pattern can be inverted if necessary.  pattern should be either a custom string or regexp. behavior should be one of: RemovedIsolatedMergedWithPreviousMergedWithNextContiguous invert should be a boolean flag.  | Example with pattern = ` `, behavior = `"isolated"`, invert = `False`:  Input: `"Hello, how are you?"`  Output: `"Hello,", " ", "how", " ", "are", " ", "you?"` |
| Sequence | Lets you compose multiple `PreTokenizer` that will be run in the given order | `Sequence::new(vec![Punctuation, WhitespaceSplit])` |

| Name | Description | Example |
| :--- | :--- | :--- |
| ByteLevel | Splits on whitespaces while remapping all the bytes to a set of visible characters. This technique as been introduced by OpenAI with GPT-2 and has some more or less nice properties:  Since it maps on bytes, a tokenizer using this only requires **256** characters as initial alphabet (the number of values a byte can have), as opposed to the 130,000+ Unicode characters. A consequence of the previous point is that it is absolutely unnecessary to have an unknown token using this since we can represent anything with 256 tokens (Youhou!! 🎉🎉) For non ascii characters, it gets completely unreadable, but it works nonetheless!  | Input: `"Hello my friend, how are you?"`  Output: `"Hello", "Ġmy", Ġfriend", ",", "Ġhow", "Ġare", "Ġyou", "?"`  |
| Whitespace | Splits on word boundaries (using the following regular expression: `\w+&#124;[^\w\s]+` | Input: `"Hello there!"`  Output: `"Hello", "there", "!"`  |
| WhitespaceSplit | Splits on any whitespace character | Input: `"Hello there!"`  Output: `"Hello", "there!"`  |
| Punctuation | Will isolate all punctuation characters | Input: `"Hello?"`  Output: `"Hello", "?"`  |
| Metaspace | Splits on whitespaces and replaces them with a special char “▁” (U+2581) | Input: `"Hello there"`  Output: `"Hello", "▁there"`  |
| CharDelimiterSplit | Splits on a given character | Example with `x`:  Input: `"Helloxthere"`  Output: `"Hello", "there"`  |
| Digits | Splits the numbers from any other characters. | Input: `"Hello123there"`   Output: ``"Hello", "123", "there"``  |
| Split | Versatile pre-tokenizer that splits on provided pattern and according to provided behavior. The pattern can be inverted if necessary.  pattern should be either a custom string or regexp. behavior should be one of: removedisolatedmergedWithPreviousmergedWithNextcontiguous invert should be a boolean flag.  | Example with pattern = ` `, behavior = `"isolated"`, invert = `False`:  Input: `"Hello, how are you?"`  Output: `"Hello,", " ", "how", " ", "are", " ", "you?"` |
| Sequence | Lets you compose multiple `PreTokenizer` that will be run in the given order | |

## Models

Models are the core algorithms used to actually tokenize, and therefore,
they are the only mandatory component of a Tokenizer.

| Name | Description |
| :--- | :--- |
| WordLevel | This is the “classic” tokenization algorithm. It let’s you simply map words to IDs without anything fancy. This has the advantage of being really simple to use and understand, but it requires extremely large vocabularies for a good coverage. Using this `Model` requires the use of a `PreTokenizer`. No choice will be made by this model directly, it simply maps input tokens to IDs.  |
| BPE | One of the most popular subword tokenization algorithm. The Byte-Pair-Encoding works by starting with characters, while merging those that are the most frequently seen together, thus creating new tokens. It then works iteratively to build new tokens out of the most frequent pairs it sees in a corpus. BPE is able to build words it has never seen by using multiple subword tokens, and thus requires smaller vocabularies, with less chances of having “unk” (unknown) tokens.  |
| WordPiece | This is a subword tokenization algorithm quite similar to BPE, used mainly by Google in models like BERT. It uses a greedy algorithm, that tries to build long words first, splitting in multiple tokens when entire words don’t exist in the vocabulary. This is different from BPE that starts from characters, building bigger tokens as possible. It uses the famous `##` prefix to identify tokens that are part of a word (ie not starting a word).  |
| Unigram | Unigram is also a subword tokenization algorithm, and works by trying to identify the best set of subword tokens to maximize the probability for a given sentence. This is different from BPE in the way that this is not deterministic based on a set of rules applied sequentially. Instead Unigram will be able to compute multiple ways of tokenizing, while choosing the most probable one. |

## Post-Processors

After the whole pipeline, we sometimes want to insert some special
tokens before feed a tokenized string into a model like "[CLS] My
horse is amazing [SEP]". The `PostProcessor` is the component doing
just that.

| Name | Description | Example |
| :--- | :--- | :--- |
| TemplateProcessing | Let’s you easily template the post processing, adding special tokens, and specifying the `type_id` for each sequence/special token. The template is given two strings representing the single sequence and the pair of sequences, as well as a set of special tokens to use. | Example, when specifying a template with these values:   single: `"[CLS] $A [SEP]"`   pair: `"[CLS] $A [SEP] $B [SEP]"`   special tokens:  `"[CLS]"` `"[SEP]"`     Input: `("I like this", "but not this")`  Output: `"[CLS] I like this [SEP] but not this [SEP]"` |

## Decoders

The Decoder knows how to go from the IDs used by the Tokenizer, back to
a readable piece of text. Some `Normalizer` and `PreTokenizer` use
special characters or identifiers that need to be reverted for example.

| Name | Description |
| :--- | :--- |
| ByteLevel | Reverts the ByteLevel PreTokenizer. This PreTokenizer encodes at the byte-level, using a set of visible Unicode characters to represent each byte, so we need a Decoder to revert this process and get something readable again. |
| Metaspace | Reverts the Metaspace PreTokenizer. This PreTokenizer uses a special identifier `▁` to identify whitespaces, and so this Decoder helps with decoding these. |
| WordPiece | Reverts the WordPiece Model. This model uses a special identifier `##` for continuing subwords, and so this Decoder helps with decoding these. |

# Training from memory

In the [Quicktour](quicktour), we saw how to build and train a
tokenizer using text files, but we can actually use any Python Iterator.
In this section we'll see a few different ways of training our
tokenizer.

For all the examples listed below, we'll use the same [Tokenizer](/docs/tokenizers/v0.22.2/en/api/tokenizer#tokenizers.Tokenizer) and
`Trainer`, built as
following:

```python
from tokenizers import Tokenizer, decoders, models, normalizers, pre_tokenizers, trainers
tokenizer = Tokenizer(models.Unigram())
tokenizer.normalizer = normalizers.NFKC()
tokenizer.pre_tokenizer = pre_tokenizers.ByteLevel()
tokenizer.decoder = decoders.ByteLevel()
trainer = trainers.UnigramTrainer(
    vocab_size=20000,
    initial_alphabet=pre_tokenizers.ByteLevel.alphabet(),
    special_tokens=["", "", ""],
)
```

This tokenizer is based on the [Unigram](/docs/tokenizers/v0.22.2/en/api/models#tokenizers.models.Unigram) model. It
takes care of normalizing the input using the NFKC Unicode normalization
method, and uses a [ByteLevel](/docs/tokenizers/v0.22.2/en/api/pre-tokenizers#tokenizers.pre_tokenizers.ByteLevel) pre-tokenizer with the corresponding decoder.

For more information on the components used here, you can check
[here](components).

## The most basic way

As you probably guessed already, the easiest way to train our tokenizer
is by using a `List`{.interpreted-text role="obj"}:

```python
# First few lines of the "Zen of Python" https://www.python.org/dev/peps/pep-0020/
data = [
    "Beautiful is better than ugly."
    "Explicit is better than implicit."
    "Simple is better than complex."
    "Complex is better than complicated."
    "Flat is better than nested."
    "Sparse is better than dense."
    "Readability counts."
]
tokenizer.train_from_iterator(data, trainer=trainer)
```

Easy, right? You can use anything working as an iterator here, be it a
`List`{.interpreted-text role="obj"}, `Tuple`{.interpreted-text
role="obj"}, or a `np.Array`{.interpreted-text role="obj"}. Anything
works as long as it provides strings.

## Using the 🤗 Datasets library

An awesome way to access one of the many datasets that exist out there
is by using the 🤗 Datasets library. For more information about it, you
should check [the official documentation
here](https://huggingface.co/docs/datasets/).

Let's start by loading our dataset:

```python
import datasets
dataset = datasets.load_dataset("wikitext", "wikitext-103-raw-v1", split="train+test+validation")
```

The next step is to build an iterator over this dataset. The easiest way
to do this is probably by using a generator:

```python
def batch_iterator(batch_size=1000):
    # Only keep the text column to avoid decoding the rest of the columns unnecessarily
    tok_dataset = dataset.select_columns("text")
    for batch in tok_dataset.iter(batch_size):  # type: ignore[attr-defined]
        yield batch["text"]
```

As you can see here, for improved efficiency we can actually provide a
batch of examples used to train, instead of iterating over them one by
one. By doing so, we can expect performances very similar to those we
got while training directly from files.

With our iterator ready, we just need to launch the training. In order
to improve the look of our progress bars, we can specify the total
length of the dataset:

```python
tokenizer.train_from_iterator(batch_iterator(), trainer=trainer, length=len(dataset))  # type: ignore[arg-type]
```

And that's it!

## Using gzip files

Since gzip files in Python can be used as iterators, it is extremely
simple to train on such files:

```python
import gzip
with gzip.open("data/my-file.0.gz", "rt") as f:
    tokenizer.train_from_iterator(f, trainer=trainer)
```

Now if we wanted to train from multiple gzip files, it wouldn't be much
harder:

```python
files = ["data/my-file.0.gz", "data/my-file.1.gz", "data/my-file.2.gz"]
def gzip_iterator():
    for path in files:
        with gzip.open(path, "rt") as f:
            for line in f:
                yield line
tokenizer.train_from_iterator(gzip_iterator(), trainer=trainer)
```

And voilà!

