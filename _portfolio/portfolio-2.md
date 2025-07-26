---
title: "Hybrid Deep Learning for Authorship Attribution: Multi‑Branch Neural Network with Stylometric Features"
excerpt: "A proof-of-concept hybrid neural network for authorship attribution, integrating BiLSTM-based word embeddings, character-level CNNs, and handcrafted stylometric features. This multi-branch model captures nuanced writing styles and provides a scalable framework for advanced authorship analysis."
collection: portfolio
---

## Project Summary

This project started with a simple goal: figure out whether two snippets of text were written by the same person. That’s a classic authorship attribution problem, and while it’s easy to throw a big neural network at it and see what happens, I wanted to experiment with something a little more interesting: combining deep learning with old‑school stylometry. It’s one thing to let a model learn patterns on its own, but there’s still value in manually‑engineered features like sentence length, punctuation habits, and lexical diversity, which researchers have relied on for decades. This was meant to be a proof of concept, a stepping stone toward a larger system, but one that actually combined the best of both approaches.

Before diving in, I also spent time reviewing prior work in authorship attribution. There’s been a long history of researchers using simple but powerful statistical techniques like character n‑grams, function‑word frequencies, and sentence‑level heuristics to identify authors. On the other end of the spectrum, recent neural models rely almost entirely on learned representations. What interested me most was that the two worlds rarely intersect in practical implementations. So, I wanted to explore what a blended approach might look like: could we design a network that uses the flexibility of deep learning without discarding the interpretability of traditional features?

I started by cleaning up the dataset. The data was made up of text snippets, each paired with a binary label: same author or not. Preprocessing was pretty straightforward — lowercase everything, strip out extraneous characters, and then tokenize. I split the tokenization into two levels: words and characters. Words tell you something about an author’s vocabulary and syntax, while characters reveal lower‑level quirks like spelling patterns or punctuation use. On top of that, I built a set of handcrafted features — things like average words per sentence, lexical diversity, and relative use of commas, semicolons, and colons. Those were run through StandardScaler so they’d play nicely with the neural network later.

Here’s what the function for extracting those stylometric features looks like:
```
def tokenize_and_extract_features(text):
    tokens = word_tokenize(text)
    sent_count = max(len(nltk.sent_tokenize(text)), 1)
    avg_words_per_sentence = len(tokens) / sent_count
    lexical_diversity = len(set(tokens)) / len(tokens) if tokens else 0
    return [avg_words_per_sentence, lexical_diversity, text.count(',') / sent_count,
            text.count(';') / sent_count, text.count(':') / sent_count]
```

Once the features were ready, I set up tokenizers for the word and character inputs using Keras, with limits on vocabulary size and sequence length to keep everything consistent. At that point, I had three clean sets of features representing three levels of writing: semantic, orthographic, and stylistic.

This multi‑level representation also meant that each branch of the network had a clear purpose. The character branch didn’t need to capture semantic meaning, because that was handled by the word embeddings. The stylometric branch didn’t need to do deep pattern recognition, because the neural layers could. Instead, each one specialized in what it was best at. Having these distinct branches working in parallel gave the model more ways to “look at” a text and spot patterns that might not be obvious at any single level of analysis.

The model itself came together as a multi‑branch neural network. Each input type gets its own little pipeline. The word branch uses an embedding layer followed by a Bidirectional LSTM, which is great for picking up on context and sequential dependencies. The character branch is a convolutional network, since CNNs are good at spotting smaller patterns like repetitive letter combinations or punctuation quirks. The stylometric features go through a simple dense layer to make them usable alongside the learned representations.

Here’s a snippet from the architecture:
```
# word branch
word_input = Input(shape=(max_len,))
word_emb = Embedding(max_words, 128)(word_input)
word_bi_lstm = Bidirectional(LSTM(64, return_sequences=True))(word_emb)
word_pool = GlobalMaxPooling1D()(word_bi_lstm)

# char branch
char_input = Input(shape=(max_chars,))
char_emb = Embedding(100, 64)(char_input)
char_conv = Conv1D(64, 5, activation='relu')(char_emb)
char_pool = GlobalMaxPooling1D()(char_conv)

# stylometry branch
style_input = Input(shape=(stylometry_train.shape[1],))
style_dense = Dense(32, activation='relu')(style_input)

# merge and finish
merged = concatenate([word_pool, char_pool, style_dense])
dense1 = Dense(128, activation='relu')(merged)
drop = Dropout(0.5)(dense1)
output = Dense(1, activation='sigmoid')(drop)
model = Model(inputs=[word_input, char_input, style_input], outputs=output)
```
After merging the three branches, I ran everything through dense layers with dropout and a single sigmoid neuron for the binary prediction.

Training was relatively standard. I compiled with Adam and binary cross‑entropy, added class weights to deal with some label imbalance, and set up early stopping to prevent overfitting. The training loop looked like this:

```
model.compile(optimizer='adam', loss='binary_crossentropy', metrics=['accuracy'])
early_stop = EarlyStopping(monitor='val_loss', patience=3, restore_best_weights=True)
model.fit([X_train_w, X_train_c, X_train_s], y_train_split,
          validation_data=([X_val_w, X_val_c, X_val_s], y_val),
          epochs=15, batch_size=64, callbacks=[early_stop])
```

I also paid close attention to overfitting throughout the training process. Because the dataset contained multiple snippets from the same authors, there was a real risk of the model memorizing individual quirks from the training set rather than learning generalizable patterns. I mitigated this not just with early stopping, but also by testing different validation splits to make sure the model wasn’t just getting lucky with familiar authors. While this wasn’t a perfect fix, it gave me more confidence that the performance gains were real and not just artifacts of the data split.

In terms of results, the model ended up hitting a validation accuracy of around 87-88%. For a proof of concept using relatively short text snippets, I’m happy with that. It did best on longer, more distinct passages (which makes sense — the more text you give it, the more patterns it can find). Shorter, generic snippets were trickier, which is exactly where the stylometric features helped most.

Challenges? Plenty. Balancing the handcrafted features with the learned ones was harder than expected. If the stylometry branch carried too much weight, the model leaned too heavily on shallow metrics like sentence length and punctuation, which doesn’t generalize well. If it carried too little, those features were basically ignored. Getting that balance right took a lot of tuning. The class imbalance was another headache. I handled it with weighting, but oversampling or synthetic data might help in the future.

There’s plenty of room for this model to grow. Using pre‑trained embeddings like GloVe or FastText on the word branch could give it a stronger semantic foundation, and adding extra convolutional layers to the character branch might help it pick up on even finer stylistic patterns. It would also be interesting to ensemble this hybrid approach with a more traditional classifier to see if the combined strengths push the performance past the 90% range.

Even so, this project accomplished exactly what it set out to do: create a functioning proof of concept for hybrid authorship attribution. It’s modular, easy to interpret, and designed to be expanded into something more robust. More than anything, it showed how traditional NLP features can still work hand‑in‑hand with deep learning instead of being left behind.
