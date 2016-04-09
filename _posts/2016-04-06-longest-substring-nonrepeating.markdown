---
layout: post
title:  "Longest Common subsequence with non-repeating characters"
date:   2016-04-06 15:12:30
categories: blog
tags: problems algorithms
---
Problem is to find the longest subsequence of a given string with non-repeating characters.
(DO NOT confuse with finding the unique elements in a string)

Found the problem [here][leetcode].  

Solution
===

Skipping the brute force solution which is simply to iterate over all possible subsequences with unique elements and counting the length of the maximum one. 

The second idea that came to mind was to keep a map of all the unique elements seen so far, and whenver a non-unique
element is encountered, move the start of the subsequence to one more than the previous position of this non-repeating
element. Here is the complete solution:

```cpp

#include <map>
#include <utility>

using namespace std;

class Solution {
public:

    int lengthOfLongestSubstring(string s) {
        int mlen = 0, start = 0;
        map<char, int> m;
        while (start < s.size()) {
            int i = start;
            while (m.find(s[i]) == m.end() && i < s.size()) {
                m[s[i]] = i;
                i++;
            }
            if (m.size() > mlen)
                mlen = m.size();
            
            if (i == s.size())
              break;
            
            // Got a repeated value!
            start = m[s[i]] + 1;
            m.clear();
        }
        
        return mlen;
    }
};

```

Unfortunately even this times out on a very large input as worst case time is still O(n^2).

It so happens that there is simple linear time solution for this one. The real insight is that to calculate whether the
current character under consideration is part of the current subsequence candidate as the answer, we just need to know
the index of the previous occurence of this character in the string, which we can store in the loop. Then, if current
char's position - length of current subsequence candidate is greater than previous index of this char, we know this
character has not been considered so far in this particular subsequence candidate and hence its length can simply be
increased by one. Otherwise we update the subsequence candidate again.

Here is the complete code:

```cpp

#include <map>
#include <utility>
#include <utility>

using namespace std;

class Solution {
public:
    int lengthOfLongestSubstring(string s) {
        if (s.size() == 0) return 0;
        int mlen = 1;
        int curlen = 1;
        map<char, int> m;
        for (int i=0; i<s.size(); i++) {
            m[s[i]] = -1;
        }
        m[s[0]] = 0;
        int prev = -1;
        for(int i=1; i<s.size(); i++) {
            prev = m[s[i]];
           
            // This char is not part of current subsequence
            if (i-curlen > prev) {
                curlen ++;
            } else {
                curlen = i - prev;
            }
            
            m[s[i]] = i;
            
            if (curlen > mlen)
                mlen = curlen;
        }
        return mlen;
    }
};

```


[leetcode]:                 https://leetcode.com/problems/longest-substring-without-repeating-characters/
