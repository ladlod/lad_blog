# 微软promote iq后端面经
## 一面（2021.8.13）
### 1.自我介绍、项目介绍
### 2.反问，了解对方工作方向
### 3.手撕代码
```Validate Parentheses (string contains chars '(', ')', '[', ']', '{', '}')	s = "()[]{}"```
>> easy难度 直接使用三个数字标注括号栈，做的时候没有考虑到'({)}'这种情况，在面试官指导下修改为使用一个真实栈，ac

```Valid Anagram of 2 strings	s = "anagram", t = "nagaram"```
>> easy难度 直接使用一个map比较两个str是否相同，ac

```
Given n integers {a1, a2, ... an} and a target sum k. Find an integer sequence {b1, b2, ... bn} such that
	1)	bi <= ai
	2)	b1 + b2 + ... + bn = k
	3)	Minimize max(bi)
Example:
	Input: a[] = [2,1,5,6,2,3], k = 13
	Solution: b[] = [2,1,3,3,2,2] max(bi) = 3
	Output: 3
solution:
	1. if sum(a) < k return 0
	2. sum(a) - k
	[1,2,2,3,5,6]
```
>> 贪心，数组从大到小排序，将第一个数字减到与第二个数字相等判断是否满足，不满足再将前两个数字减到与第三个数字相等...一开始思路正确了，写代码的时候后续每次做减法只减了最近的数字，没有一次ac，面试官提醒之后改正ac
### 面试总结
    hr反馈面试没过，面评是三道题目虽然最终都ac了，但边界情况等考虑不足，easy题也没有一次ac。反思日常刷题时确实不注意边界条件，经常写完大体代码就拿leetcode的测试用例做边界测试再补充边界条件，没有一次ac的习惯，很容易影响面试发挥。