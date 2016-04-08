---
layout: post
title:  "Longest Common subsequence with non-repeating characters"
date:   2016-04-07 15:12:30
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

```

#include <map>
#include <utility>
using namespace std;
class Solution {
public:
    int lengthOfLongestSubstring(string s) {
        int mlen = 0, start = 0;
        map<char, int> m;
        while(start < s.size()) {
            int i = start;
            while(m.find(s[i]) == m.end() && i < s.size()) {
                m[s[i]] = i;
                i++;
            }
            if(m.size() > mlen)
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



[leetcode]:                 https://leetcode.com/problems/longest-substring-without-repeating-characters/
