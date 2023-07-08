---
layout: post
title: "On Dan Boneh's coursera Cryptography I,II courses"
date: 2023-07-08
categories: [Tech]
---

# Week 1.

The description of the programming assignemnt from week 1 describes a problem where a single key was used multiple times with the OTP (One Time Pad) algorithm.
This is bad as by intercepting multiple messsages we can quickly recover the original key used.

They key points to know here is that 

- `E(k, m_1) = k ^ m_1`.
- By intercepting two or more messages we can xor the messages together to receive `k ^ m_1 ^ k ^ m_2 => m_1 ^ m_2`. 

What we're left with is the xor of the two messages, from this point onwards it gets a bit tricky.
There are multipla ways we can choose from this point. For example, we know that the plaintext is in english thus we could make guesses on what the contents of one of the message is
and xor our guess with the `m_1 ^ m_2` to see if our guess was correct or wrong. We would need to go through the most common words of the English language
and make good guesses to continiously adjust the key based on the deciphered plaintext.

Another possible solution, which from my point of view is also easier, is that we can make use of the fact, that when a character from the alphabet [a-z][A-Z] is xored
with a space `' '` then the character capitalization gets reversed (i.e lowercase => uppercase, uppercase => lowercase).

However, xoring just two ciphertext to test for this property gives us a low chance of guessing right. To increase our chances we test a quadruple at a time.
By testing multiple ciphertexts, if at the given index a space was xored gives us pretty good chances of guessing parts of the key correctly and the rest can be
guessed by reading through the decrypted ciphertexts. 

The solution for this problem can be found [here](https://github.com/Despire/coursera-crypto/blob/main/assignment-1/main.go)


