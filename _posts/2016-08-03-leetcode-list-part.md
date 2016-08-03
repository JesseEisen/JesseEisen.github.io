---
title: LeetCode OJ ——List Prat
updated: 2016-08-03 18:00
---

> 本篇博客主要关于List的一些操作的题目，单链表的题目居多

## Swap Node

> Given a linked list, swap every two adjacent nodes and return its head.
>
> For example,
> Given 1->2->3->4, you should return the list as 2->1->4->3.
>
> Your algorithm should use only constant space. You may not modify the values in the list, only nodes itself can be changed.

这道题目的是让交换相邻的两个节点,但是不修改节点的值。初步的思路：遍历链表，两两交换。只要注意下不破坏链表的链(即链表不被断开)。这个过程不需要申请额外的空间，只是指针之间的调换。

C代码如下：

```c
struct ListNode* swapPairs(struct ListNode* head) {
     struct ListNode **p = &head;

     while(*p && (*p)->next){
          struct ListNode *t = (*p)->next;

          (*p)->next = t->next;
          t->next = *p;
          *p = t;

          p = &t->next->next;
     }

     return head;
}
```

对代码简单的解释一下：如果要修改节点，我们可以用二级指针来实现。 首先将第二个节点用临时节点保存下。然后将第一个节点的后继节点指向第三个节点，同时将第一个节点作为第二个节点的后继结点，最后将原来的第二个节点放置到第一个节点上。这边需要注意**使用二级指针才能修改，否则如果将p定义成一个普通的节点，那这么替换后，链表就乱了**。 接着将p跳到两步，重复上面的动作。

上面的解释也许读起来比较绕口，实在不理解可以手动画一些图来理解这个。

这个题目竟然有人想到用递归来实现，真的是owesome! 递归的做法看代码不一定能很明白的理解。这边的递归主要是解决了在两两替换时，保证了链表能够不乱。具体的实现我贴一下代码：

```c
struct ListNode* swapPairs(struct ListNode* head) {
	if(head == NULL || head->next == NULL)
		return head;

	struct ListNode * temp;
	temp = head->next;
	head->next = swapPairs(temp->next);
	temp->next = head;

	return temp;
```

`temp->next = head` 和 `return temp` 是两个比较关键的步骤。这个保证了顺序，此外使用递归，可以避免head的丢失。如果使用迭代的话，head很容易丢失，导致在最后返回链表时找不到第一个节点。 而递归则很完美的解决了这个问题，因为最终递归返回到第一层的时候，正好是最开始的两个。所以这能保证head不丢失。


## Remove Nth Node From End of List

> Given a linked list, remove the nth node from the end of list and return its head.

For Example:

```
Given linked list: 1->2->3->4->5, and n = 2.

After removing the second node from the end, the linked list becomes 1->2->3->5.
```

这道题最容易想到的思路是: 定义一个指针,移动到要删除的节点处,用它的后继来替换它即可。所以方法可以拆分成:遍历一遍，获得长度。再次遍历到要删除的那个节点，执行替换。所以实现是很简单的：

```c
struct ListNode* removeNthFromEnd(struct ListNode* head, int n) {
	if(head == NULL)
		return NULL;

	int len = 0;
	struct ListNode *p = head;

	while(p!=NULL){
		len++;
		p = p->next;
	}

	if(n > len)
		return NULL;

	int i = 0;
	struct ListNode **current = &head;
	while(*current != NULL){
		p = *current;
		if(i++ == (len-n)){
			*current = p ->next;
			free(p);
			break;
		}else{
			current = &p->next;
		}
	}

	return head;
}
```

这个解法多了一个统计长度的步骤。实际上可以做到一次遍历就能删除掉指定节点的。我们可以用两个指针`fast`,`slow`，先让这两个指针之间的间隔为`n`,接着两个指针同步往前遍历，当`fast`指针到达链表尾部时，`slow`便停在了我们要删除节点的前面。此时要删除指定节点，是非常的简单的。

```c
struct ListNode* removeNthFromEnd(struct ListNode* head, int n) {
	struct ListNode * header;
	struct ListNode * p;

	header = malloc(sizeof(struct ListNode));
	header->next = head;

	struct ListNode *fast = header;
	struct ListNode *slow = header;
	int temp = n;

	for(; fast != NULL; temp--){
		if(temp < 0)
			slow = slow->next;

		fast = fast->next;
	}

	if(slow->next == NULL)
		return NULL;

	p = slow->next;
	slow->next = slow->next->next;
	free(p); 

	return header->next;
}
```

我在实现的时候,加上了一个头指针,为的是在删除第一个节点的时候可以方便一点。


## Remove Linked List Elements

> Remove all elements from a linked list of integers that have value val.
>
> Example
> Given: 1 --> 2 --> 6 --> 3 --> 4 --> 5 --> 6, val = 6
> Return: 1 --> 2 --> 3 --> 4 --> 5

有了之前的两道题的基础,这个题目只需要将上面`Remove nth node from end of list` 的删除节点代码修改一下即可。思路就是：从头开始遍历链表，如果遇到了指定值的节点，执行删除，直至遍历到最后。

```c
struct ListNode* removeElements(struct ListNode* head, int val) {
	struct ListNode *pre;
	struct ListNode **current;

	current = &head;

	while(*current != NULL){
		pre = *current;
		if((*current)->val == val){
				*current = pre->next;
				free(pre);
		}else
			current = &pre->next;
		}

		return head;
}
```

注意还是需要使用到二级指针，如果不想利用二级指针，同样可以添加一个头指针，然后用一般的删除来实现,简要代码如下：

```c
pre->next = current->next;
free(current);
```
主要是设置一个前置节点,这样对于上出current节点比较有利。


(未完待续....) 
