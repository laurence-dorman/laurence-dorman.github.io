---
layout: post
title: Comparing String Searching Algorithms 🔤🔎
description: Comparing the efficiency of common string searching algorithms | C++
date: 2021-01-11
---

>**Disclaimer: This is university coursework. The code is not intended to be used for any purpose other than to demonstrate the algorithms.**

## Table of contents
{:.no_toc}
* TOC
{:toc}

&nbsp;

***

&nbsp;

## [Introduction](#introduction)

Let's use a common problem to compare the efficiency of different string searching algorithms.

&nbsp;

***

&nbsp;

## [1. The Problem](#1-the-problem)

To find and store all instances of a particular combination of nucleotides within a large [nucleic acid sequence](https://en.wikipedia.org/wiki/Nucleic_acid_sequence#/media/File:AMY1gene.png) (DNA).

&nbsp;

***

&nbsp;

## [2. Data Structures](#2-data-structures)

I will be using `std::vector` as the storage data structure for the returned location values of the found pattern.

### Why `std::vector`?
  1. Items can be pushed to the end of the vector in constant time, O(1).
  2. Memory is dynamically allocated, so the vector can grow as needed.
  3. Memory is contiguous, therefore cache hits are more likely when iterating through the vector.

&nbsp;

***

&nbsp;

## [3. The Algorithms](#3-the-algorithms)

I will be comparing the following algorithms:
  1. [Boyer-Moore-Horspool algorithm](#boyer-moore-horspool-algorithm).
  2. [Rabin-Karp algorithm](#rabin-karp-algorithm).

### Boyer-Moore-Horspool algorithm
- Pre-processes the pattern to produce lookup tables that the algorithm will later use in order to skip ahead, saving time. I use arrays for the lookup tables.

- Why arrays? 
  1. Arrays are contiguous in memory, it's likely the next element to be accessed is already in the CPU cache.
  2. Element access is constant time, O(1).

- Returns a `std::vector`, storing the positions that the pattern was found at.

  ~~~cpp
  /** Boyer-Moore multi search skip algorithm - returns vector of positions of pat found in text
  - Inputs
  - const string& pat - the pattern to search for
  - const string& text - the text to search in

  - Returns
  - vector<Position> pos_vec - vector of positions that the pattern was found at
  */
  vector<Position> search_BM(const string& pat, const string& text) {

    // (Position is an alias for a long long int)
  	const Position pat_len = pat.size();
  	const Position text_len = text.size();
  	vector<Position> pos_vec; // stores positions of found patterns.

  	bool in_pattern[256] = { false }; // tracks which characters are in the pattern.
  	Position skip[256]; // lookup table for skipping ahead.

  	for (int i = 0; i < 256; i++) { // characters will initially skip forwards by the length of the pattern
  		skip[i] = pat_len;
  	}

  	for (char c : pat) 
  	{ // Set in_pattern entry to true if that character is in pat
  		in_pattern[int(c)] = true;
  	}

  	for (Position i = 0; i < pat_len; i++) { // Setup skip array for each character
  		skip[int(pat[i])] = (pat_len - 1) - i; 
  	}

    // Loop until we have reached the end of the text
  	for (Position i = 0; i <= text_len - pat_len; ++i) { 
  		Position j;
  		Position s = skip[int(text[i + pat_len - 1])];
  		if (s != 0) { // Skip forwards by value taken from skip array
  			i += s - 1; // skip forwards
  			continue;
  		}
  		if (!in_pattern[int(text[i + pat_len - 1])]) { // if current char isnt in pattern, skip forwards
  			i += pat_len - 1; // skip forwards
  			continue;
  		}
  		for (j = 0; j < pat_len; j++) { // Previous 2 if statements have failed, check all current characters against pattern
  			if (text[i + j] != pat[j]) // if any of these checks dont match, break out of current for loop
  				break; // doesn't match here
  		}
  		if (j == pat_len) { // if we exit the last for loop, with j equal to pat_len, that means we checked every char and succeeded, so we found the pattern
  			pos_vec.push_back(i); // matched here
  		}
  	}
  	return pos_vec;
  }
  ~~~

  #### Hypothetical time complexity

  >Where m is pattern length, and n is the text length:
    - Worst case: **0(n*m)**. 
    - Best case: **0(n/m)**.
    - Average case: **0(n)**.
    
  - Good average case time complexity, excellent best case time complexity.

&nbsp;

***

&nbsp;

### Rabin-Karp algorithm
- Pre-processes the pattern and the first substring of the text using a (very basic) hash function which sums the integer values of each character in the string.

  ~~~cpp
  unsigned int hash(const string& s) {
      unsigned int total = 0;
  	  for (char c : s) {
  		  total += int(c);
  	  }
  	  return total;
  }
  ~~~

- This hashing function is about as basic as it gets, and has a high likelihood of collisions - but it's good enough for this example and fairly straight forward to implement.

- When the algorithm finds a hash match, it then compares the pattern to the substring to confirm it's a match.

- "Rolls" the hash forward by subtracting the first character of the substring, and adding the next character of the text.

- Returns a `std::vector` of positions that the pattern was found at.

  ~~~cpp
  /** Rabin-Karp multi search algorithm - returns vector of positions of pat found in text
  - Inputs
  - const string& pat - the pattern to search for
  - const string& text - the text to search in

  - Returns
  - vector<Position> pos_vec - vector of positions that the pattern was found at
  */
  vector<Position> search_RK(const string& pat, const string& text) {

  	const Position pat_len = pat.size();
  	const Position text_len = text.size();
  	vector<Position> pos_vec; // stores positions of found patterns.
  
  	unsigned int patHash = hash(pat); // Store hash of pat to be used for comparison against currentHash
  	unsigned int currentHash = hash(text.substr(0, pat_len)); // Store hash of starting substring
  
  	// Loop until we have reached the end of the text	
    for (Position i = 0; i <= text_len - pat_len; i++) {		
  		Position j; // long long indexing variable
  		if (currentHash == patHash) { // Check hashes
  			for (j = 0; j < pat_len; j++) { // Check all current characters against pattern to confirm match
  				if (text[i + j] != pat[j]) // if any of these checks dont match, break out of current for loop
  					break; // doesn't match here	
  			}													   
  			if (j == pat_len) { // if we exit the previous for loop, with j equal to pat_len, that means we checked every char and succeeded, so we found the pattern
  				pos_vec.push_back(i); // matched here
  			}
  		}														   
  		currentHash += (int)text[pat_len + i] - (int)text[i]; // roll the hash forward
  	}															   
  	return pos_vec;
  }
  ~~~

  #### Hypothetical time complexity

  > Where **m** is pattern length, and **n** is the text length:
    - Worst case: **0(n*m)**.
    - Best case: **0(n+m)**.
    - Average case: **0(n+m)**.

    - I would expect that since my hash function is basic, collisions are more likely and therefore the average case is closer to the worst case.
    - Very consistent, independent of input.

&nbsp;

***

&nbsp;

## [4. Testing](#4-testing)

I will be testing both algorithms using the following test cases:
  - **Random DNA sequence**, incrementally increasing the length of the pattern (M) each iteration
  - **Boyer-Moore-H worst case**, DNA sequence that meets Boyer-Moore-Horspool's worst case (not able to skip ahead)
  - **Rabin-Karp worst case**, DNA sequence that meets Rabin-Karp's worst case (hash collision at every substring)

### Generating the test cases

I will be using the following function to generate random DNA sequences:

  ~~~cpp
  /** Generates a random DNA sequence of length l
  - Inputs
  - int l - length of sequence to generate

  - Returns
  - string s - random DNA sequence of length l
  */

  string generateDNA(int l) {
    string s = "";

    for (int i = 0; i < l, i++) {
        int r = rand () % 4;
        switch (r)
        {
        case 0:
            s += 'A';
            break;
        case 1:
            s += 'C';
            break;
        case 2:
            s += 'G';
            break;
        case 3:
            s += 'T';
            break;
        }
    }
    std::random_shuffle(s.begin(), s.end());
    return s;
  }
  ~~~

&nbsp;

***

&nbsp;

### Test Case 1: Random DNA sequence, incrementally increasing the length of the pattern (M) each iteration

> N (text length) = 2,000,000. M 1:32

- Both algorithms are searching the same text, a new pattern of same size is generated each iteration to increase consistency. Both algorithms generally run faster with a longer pattern. For Rabin-Karp, after pattern length > 3 the results stabilise and pattern length doesn’t impact results much.

- Boyer-Moore-H - much more inconsident than Rabin-Karp, time on average is decreasing as pattern length increases. Since there are only 4 possible characters in the DNA sequence, the skip table is not as effective as it would be in a larger alphabet.

#### How do the results change when text length is increased?

> N (text length) = 3,000,000. M 1:32

- Boyer-Moore median changed by 1.5x (6ms > 9ms)
- Rabin-Karp median changed by 1.5x (4ms > 6ms)

- Since the median time for both algorithms increased with respect to the text length, this shows that the time complexity is linear for the average case as expected.

- This test case shows that for a large (randomly generated) text, Rabin-Karp (4ms median) is more consistent and on average faster than Boyer-Moore-Horspool (6ms median).

&nbsp;

***

&nbsp;

### Test Case 2: Boyer-Moore-H worst case

- BMH Median: 5ms
- RK Median: 1ms

- Due to the nature of the test case, the lookup table is not able to skip ahead and the time complexity is **O(n*m)** for Boyer-Moore-Horspool. However, Rabin-Karp is still able to roll the hash forward without collisions and the time complexity is **O(n+m)**.

&nbsp;

***

&nbsp;

### Test Case 3: Rabin-Karp worst case

- Pattern input: “ACCGTACGTACGTACGTACGTACGTACGTAC**TG**”. – hash collisions will always happen (both hash values are equal) and it will have to check every char until the 2nd last.

- Rabin-Karp algorithm will be comparing this pattern with the hash collision from substring: “ACCGTACGTACGTACGTACGTACGTACGTAC**GT**” every iteration, thus slowing it down immensely.

- BMH Median: 3ms
- RK Median: 5ms

- Due to the hash collisions at every substring, the time complexity is **O(n*m)** for Rabin-Karp. However, Boyer-Moore-Horspool is still able to skip ahead.

&nbsp;

***

&nbsp;

# [Conclusion](#conclusion)

For this particular problem I would use my Rabin-Karp string searching algorithm because:
  1. It is far more consistent for this particular problem.
  2. It has better average case time complexity for this problem.
  3. It has less memory overhead (no lookup tables). Rabin-Karp only needs to store the hash values of the pattern and the substring.
  4. Test case 1 shows that Rabin-Karp performed better on average for a random DNA sequence, which would be similar to a real world scenario.