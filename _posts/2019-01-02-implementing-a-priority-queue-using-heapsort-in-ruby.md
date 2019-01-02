---
layout: post
title:  "Implementing a Priority Queue Using Heapsort in Ruby"
date:   2019-01-02 17:07:38 -0500
categories:
  - priority queue
  - heapsort
  - algorithms
  - min-heap
  - ruby
---
Heaps are a tree data structure that can be implemented using an array. In this case a heap will be a binary min-heap, which means each parent node will be less than the value of each of, at most, two child nodes.

```
    1
   / \
  3   5
 / \
8   4
```

The above structure can be represented in an array:

```ruby
[nil, 1, 3, 5, 8, 4]
```

Note the first element of the array (i = 0) is `nil` to simplify the implementation of the queue (which will be explained below). This element does not represent actual data.

## Traversing Array Representation of a Binary Heap

To find the index of the left child of a index `i` of parent node in the array use `2 * i`.

The index of the right child in the array is `(2 * i) + 1`.

To find the index of parent for a child of index `j` in the array is `j / 2`.

The above math works out if the root of the heap starts in the second container of the array, i.e., `i = 1`.

## Initializing the Heap

In the ruby implementation the heap in our priority queue is initialized with a dummy `nil` value in the first container:

```ruby
class PriorityQueue
  def initialize
    @heap = [nil]
  end
end
```

## Inserting Values

In heapsort the latest value is inserted in the next open leaf node going left to right. For example if `3` is inserted into the following heap:

```
    1
   / \
  4   5
 /
8
```

It would result in:

```
    1
   / \
  4   5
 / \
8   3
```

In the array representation this value would simply be inserted at the end:

```ruby
[nil, 1, 4, 5, 8]
# insert 3
[nil, 1, 4, 5, 8, 3]
```

Now the heap has to be reordered so each parent is less than its children. This is accomplished be comparing the last inserted value to its parent:

```
    1
   / \
  4   5
 / \
8   3
```

If it is less than its parent then those values are swapped:

```
    1
   / \
  3   5
 / \
8   4
```

This is done recursively up the tree until the it reaches the root or a parent with a value that is less than it.


In ruby this can be implemented like so:

```ruby
class PriorityQueue
  def initialize
    @heap = [nil]
  end

  def insert(item)
    @heap << item
    bubble_up(last_index)
    self
  end

  private

  def bubble_up(index)
    parent_index = index / 2

    return if parent_index.zero?

    child = @heap[index]
    parent = @heap[parent_index]

    if parent > child
      swap(index, parent_index)
      bubble_up(parent_index)
    end
  end

  def last_index
    @heap.length - 1
  end

  def swap(index_a, index_b)
    @heap[index_a], @heap[index_b] = @heap[index_b], @heap[index_a]
  end
end
```

## Extracting the Minimum Value

The root node of the heap represents the minimum value in the queue. Once this value is removed from the queue the next minimum value must occupy the root position.

Starting with this initial heap:

```
    1
   / \
  3   5
 / \
8   9
```

The root node and the last right-most node are swapped:

```
    9
   / \
  3   5
 / \
8   1
```

Then the last right-most node is removed:

```
    9
   / \
  3   5
 /
8
```

Then the heap is reordered by comparing the root note to its smallest child and swapping them if the root is greater than that child:

```
    3
   / \
  9   5
 /
8
```

The the newly swapped child from the last operation is then compared to its smallest child and then again swapped if it is greater than that child. This is done recursively downward until the bottom-most node is reached or the child is greater than the parent:

```
    3
   / \
  8   5
 /
9
```

This can be implemented in ruby by extending the `PriorityQueue` class from above. Here the method `next` is added which returns the minimum value in the queue and reorders the heap:

```ruby
class PriorityQueue
  ...

  def next
    minimum = @heap[1]
    swap(1, last_index)
    @heap.pop
    bubble_down(1)
    minimum
  end

  private

  ...

  def bubble_down(index)
    left_child_index = index * 2
    right_child_index = index * 2 + 1

    return if left_child_index > last_index

    lesser_child_index = determine_lesser_child(left_child_index, right_child_index)

    if @heap[index] > @heap[lesser_child_index]
      swap(index, lesser_child_index)
      bubble_down(lesser_child_index)
    end
  end

  def determine_lesser_child(left_child_index, right_child_index)
    return left_child_index if right_child_index > last_index

    if @heap[left_child_index] < @heap[right_child_index]
      left_child_index
    else
      right_child_index
    end
  end

  ...
end
```

## Usage

Here is an example of how this class would be used:

```ruby
pq = PriorityQueue.new
=> #<PriorityQueue:0x00007ffa21145050 @heap=[nil]>
pq.insert(100)
=> #<PriorityQueue:0x00007ffa21145050 @heap=[nil, 100]>
pq.insert(1)
=> #<PriorityQueue:0x00007ffa21145050 @heap=[nil, 1, 100]>
pq.insert(50)
=> #<PriorityQueue:0x00007ffa21145050 @heap=[nil, 1, 100, 50]>
pq.next
=> 1
pq.next
=> 50
pq.next
=> 100
pq.next
=> nil
```
