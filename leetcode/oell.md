```python
class Solution:
    def oddEvenList(self, head: Optional[ListNode]) -> Optional[ListNode]:
        # we want to unzip the nodes and then attach the them
        # maybe we could use a dummy node for the odd and even ones
        # oh -> 1 -> 3 -> 5 -> 2 -> 4 -> None
        # eh ->
        # 5 -> None
        # None
        #    1 -> 2 -> 3 -> 4 -> 5
        #              o    e
        odd = ListNode()
        even = ListNode(next=head)
        oh = odd
        eh = even

        while even and even.next:
            # we want to set odd.next = even.next
            # then odd = odd.next
            # then even.next = odd.next
            # then even = even.next
            odd.next = even.next
            odd = odd.next
            even.next = odd.next
            even = even.next

        odd.next = eh.next

        return oh.next
```
