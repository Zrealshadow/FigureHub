# cmu  15445  Project 2  B+Tree

[TOC]

### B+ Tree

A B+ tree is a self-balancing tree data structure that keeps data sorted and allows searches, sequential access, insertion and deletions in O(log(n)). It is optim8ized for disk-oriented DBMSs that read/write large blocks of data.

Properties of B+ tree:

- Perfectly balanced (i.e., every leaf node is at the same depth)
- Every inner node other than the root is at least half full ( M/2 - 1 <= num of keys <= M -1)
- Every inner node with k keys has k+1 non-null children ( the zero position of the key is empty)

Every node in a B+ tree contains an array of key/value pairs:

- Arrays at every node are sorted by keys
- value array for inner nodes contain pointers to other nodes(inner nodes or leaf nodes)
- leaf node can save two kinds of value: 
  - Record IDs: A pointer to the location of the tuple
  - Tuple Data: actual data

#### Insertion

1. Find correct leaf L.
2. Add new entry into L in sorted order:
   - if L has enough space, the operation done
   - Otherwise (number of keys == M), **split** L into two nodes L and L2, Redistribute entries evenly and copy up middle key.
   - Insert index entry pointing to L2 into parent of L.
3. split an inner node, redistribute entries evenly, but push up the middle key.

![cmu](https://raw.githubusercontent.com/Zrealshadow/FigureHub/main/CMU15445/proj2/B%2Btree_Insertation.png)



![cmu](https://raw.githubusercontent.com/Zrealshadow/FigureHub/main/CMU15445/proj2/B%2Btree_Insertation2.png)

#### Deletion

1. Find correct leaf L.
2. Remove the entry:
   - If the number of key is larger than the half full, the operation is done.
   - Otherwise, try to **redistribute**, borrowing  key from siblings whose number of key is larger than the half full.
   - If redistribute fails, merge L and siblings.
3. If merge occurred,  delete entry in parent pointing to L and recursively check parent node.



![cmu](https://raw.githubusercontent.com/Zrealshadow/FigureHub/main/CMU15445/proj2/B%2Btree_delete.jpg)

ABout B+ Tree: https://www.geeksforgeeks.org/introduction-of-b-tree/

---



### Project Details

I will introduce my thinking ~

Also explain my understanding about the API given by Andy. 



Firstly, we have to finish

- B+Tree Page (b_plus_tree_page.h / b_plus_tree_page.cpp)
- B+ Tree Internal Page (b_plus_tree_internal_page.h / b_plus_tree_internal_page.cpp)
- B+ Tree Leaf Page (b_plus_tree_leaf_page.h / b_plus_tree_leaf_page.cpp)

**BPlusTreePage** is the father class of **BPlussTreeInternalPage** and **BPlusTreeLeafPage**.

In **b_plus_tree_page.h**, it defines the common methods, for example IsLeafPage(), IsRootPage() to judge the type of this node, or GetSize(), SetSize() to access private attribute.



Each B+ tree Page corresponds to the content of memory page (it means that the B+ tree page is saved in data content of page fecthed from memory buffer pool)

So every time if you try to read or write a leaf/internal page, when you have an unique `page_id`. You have to do like this

```c++
 		auto page = buffer_pool_manager.Fetch(page_id);
		BPlusTreePage * node = reinterprete_cast<BPlusTreePage*> (page.getData());
```



In the implementation of the **BPlusTreePage**, we should be careful about the BPlusTreePage

```c++
int BPlusTreePage::GetMinSize() const {
  if(IsRootPage()){
    if(IsLeafPage()){
      return 1;
    }
    else return 2;
  }
  return max_size_ / 2;
}
```

this method is used to judge whether this node need merge and redistribution when deleting some k/v. here, we have to judge the type of the node. If the node is root page and Internal Page, node only contains one key, so the size is 2 (the first empty value also points to the leaf node). the node is leaf page, node only contains one tuple, so the minimum size is 1. If the node is not root page, so the minimum size is the max_size_ / 2; 



**Note: this `max_size_` is less than the size of the memory 1. This design is convenient for insert. For example, we can insert key firstly, sort the key and then judge whether to split it.**



In **b_plus_tree_page**, **BPlusTreeInternalPage** is defined and it is the sub class of **BPlusTreePage**. It store **n** indexed keys and **n+1** child pointers(page_id) within internal page. Pointer PAGE_ID(i) points to a subtree in which all keys **K** satisfy: **K(i) <= K < K(i+1)**. Since the number of keys dose not equal to number of child pointers, the first key always remians invalid. **That is to say, any search/ lookup should ignore the first key.**



`Lookup` is to search key in page. It can use linear search or binary search. Linear search is easy to implement. In binary search, be careful about the boarder problem.

Note:

- when the inserted key dose not exist in these page, the function should also return the key which is largest but less than inserted key. So the `comparator <= 0`
- If `GetSize == 1`, it means only contains the first empty k, whose index is 0. But the start of binary search is 1. Thus `return ValueAt(left - 1)`.

```c++
#define INDEX_TEMPLATE_ARGUMENTS template <typename KeyType, typename ValueType, typename KeyComparator>

INDEX_TEMPLATE_ARGUMENTS
ValueType B_PLUS_TREE_INTERNAL_PAGE_TYPE::Lookup(const KeyType &key, const KeyComparator &comparator){
  int left = 1, right = GetSize() - 1;
  while(left <= right){
    int mid = (left + right) / 2;
    if (comparator(KeyAt(mid), key) <= 0){
      left = mid + 1;
    }
    else right = mid - 1;
  }
  return ValueAt(left - 1);
}
```



`PopulateNewRoot` is to create a new root Internal page, it only called within `InsertIntoParent()(b_plus_tree.cpp)`. The situation is that the internal root page is full and have to split into two internal page. Thus the new root internal page only have one key, and two pointers.

```c++
INDEX_TEMPLATE_ARGUMENTS
  void B_PLUS_TREE_INTERNAL_PAGE_TYPE::PopulateNewRoot(const ValueType &old_value, const KeyType &new_key, const ValueType &new_value)
{
  array[0].second = old_value;
  array[1] = {new_key, new_value};
  SetSize(2);
}
```



`InsertNodeAfter`  is the Insert operation, it is always called with the `Lookup`. 

Insert new_key and new_value pair right after the pair with its value == old_value.

```c++
INDEX_TEMPLATE_ARGUMENTS
int B_PLUS_TREE_INTERNAL_PAGE_TYPE::InsertNodeAfter(const ValueType &old_value, const KeyType &new_key, const ValueType &new_value){
  int index = ValueIndex(old_value);
  IncreaseSize(1);
  for(int i = GetSize() - 1; i > index + 1; i--){
    array[i] = array[i - 1];
  }
  array[index + 1] = {new_key, new_value};
  return GetSize();
}
```





**SPLIT &  MERGE & REDISTRIBUTION**

**Note:**

**When we move the children nodes, do not forget to change the parentId of children nodes.**

**When we fetch one page from buffer_pool_manager, do not forget to Unpin the page when the operation is finished**.



`MoveHalfTo` and `CopyNFrom`  are used when internal pages are splited.

Be careful about how to choose the middle key's position( the value of `start_index`)

```c++
/*
 * Remove half of key & value pairs from this page to "recipient" page
 */
INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_INTERNAL_PAGE_TYPE::MoveHalfTo(BPlusTreeInternalPage *recipient,
                                                BufferPoolManager *buffer_pool_manager) {
    int size = GetSize() / 2;
    int start_index = (GetSize() + 1) / 2;
    recipient->CopyNFrom(array + start_index, size, buffer_pool_manager);
    IncreaseSize(-1 * size);
}

/* Copy entries into me, starting from {items} and copy {size} entries.
 * Since it is an internal page, for all entries (pages) moved, their parents page now changes to me.
 * So I need to 'adopt' them by changing their parent page id, which needs to be persisted with BufferPoolManger
 */
INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_INTERNAL_PAGE_TYPE::CopyNFrom(MappingType *items, int size, BufferPoolManager *buffer_pool_manager) {
  for(int i = 0, j = GetSize(); i < size; j++, i++){
    array[j] = items[i];
    page_id_t child_page_id = items[i].second;
    //DEBUG: no cast
    auto childDataPage = buffer_pool_manager->FetchPage(child_page_id);
    BPlusTreePage * child_page = reinterpret_cast<BPlusTreePage*>(childDataPage->GetData());
    child_page->SetParentPageId(GetPageId());
    buffer_pool_manager->UnpinPage(child_page_id, true);
  }
  IncreaseSize(size);
}
```



`MoveAllTo` is used when two internal pages are merged.

**Trick**: We used the first empty key to make it more convenient.

![cmu](https://raw.githubusercontent.com/Zrealshadow/FigureHub/main/CMU15445/proj2/B%2BtreeMerge.png)

```c++
/*
 * Remove all of key & value pairs from this page to "recipient" page.
 * The middle_key is the separation key you should get from the parent. You need
 * to make sure the middle key is added to the recipient to maintain the invariant.
 * You also need to use BufferPoolManager to persist changes to the parent page id for those
 * pages that are moved to the recipient
 */

INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_INTERNAL_PAGE_TYPE::MoveAllTo(BPlusTreeInternalPage *recipient, const KeyType &middle_key,
                                               BufferPoolManager *buffer_pool_manager) {
      auto *parentDataPage = buffer_pool_manager->FetchPage(GetParentPageId())->GetData();
      B_PLUS_TREE_INTERNAL_PAGE_TYPE *parentPage = reinterpret_cast<B_PLUS_TREE_INTERNAL_PAGE_TYPE *>(parentDataPage);
      int index = parentPage->ValueAt(GetPageId());
      KeyType tmp = parentPage->KeyAt(index);
      SetKeyAt(0, tmp);
      parentPage->Remove(index);
      buffer_pool_manager->UnpinPage(parentPage->GetPageId(), true);
      recipient->CopyNFrom(array, GetSize(), buffer_pool_manager);
      SetSize(0);
}
```





`MoveFirstToEndOf` and `MoveLastToFrontOf` are called when internal pages are redistributed.

`CopyLastFrom` and `CopyFirstFrom` are helper function.

**Note:** 

- Do not forget to change the key of parent node
- Do not forget to change the parent id of moved page.

```c++
INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_INTERNAL_PAGE_TYPE::MoveFirstToEndOf(BPlusTreeInternalPage *recipient, const KeyType &middle_key,
                                                      BufferPoolManager *buffer_pool_manager) {
        auto *parentDataPage = buffer_pool_manager->FetchPage(GetParentPageId())->GetData();
        B_PLUS_TREE_INTERNAL_PAGE_TYPE *parentPage = reinterpret_cast<B_PLUS_TREE_INTERNAL_PAGE_TYPE *>(parentDataPage);
        int index = parentPage->ValueAt(GetPageId());
        KeyType tmp = parentPage->KeyAt(index);
        parentPage->SetKeyAt(index, KeyAt(1));
        buffer_pool_manager->UnpinPage(GetParentPageId(), true);
        MappingType moveKV = {tmp, ValueAt(0)};
        SetValueAt(0, ValueAt(1));
        Remove(1);
        recipient->CopyLastFrom(moveKV, buffer_pool_manager);
        IncreaseSize(-1);
}

/* Append an entry at the end.
 * Since it is an internal page, the moved entry(page)'s parent needs to be updated.
 * So I need to 'adopt' it by changing its parent page id, which needs to be persisted with BufferPoolManger
 */
INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_INTERNAL_PAGE_TYPE::CopyLastFrom(const MappingType &pair, BufferPoolManager *buffer_pool_manager) {
      auto * childDataPage = buffer_pool_manager->FetchPage(pair.second)->GetData();
      B_PLUS_TREE_INTERNAL_PAGE_TYPE *childPage = reinterpret_cast<B_PLUS_TREE_INTERNAL_PAGE_TYPE *>(childDataPage);
      childPage->SetParentPageId(GetPageId());
      buffer_pool_manager->UnpinPage(pair.second,true);
      array[GetSize()] = pair;
      IncreaseSize(1);
}

/*
 * Remove the last key & value pair from this page to head of "recipient" page.
 * You need to handle the original dummy key properly, e.g. updating recipient’s array to position the middle_key at the
 * right place.
 * You also need to use BufferPoolManager to persist changes to the parent page id for those pages that are
 * moved to the recipient
 */
INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_INTERNAL_PAGE_TYPE::MoveLastToFrontOf(BPlusTreeInternalPage *recipient, const KeyType &middle_key,
                                                       BufferPoolManager *buffer_pool_manager) {
      // auto * parentDataPage = buffer_pool_manager->FetchPage(GetParentPageId())->GetData();
      // B_PLUS_TREE_INTERNAL_PAGE_TYPE *parentPage = reinterpret_cast<B_PLUS_TREE_INTERNAL_PAGE_TYPE *> (parentDataPage);
      // KeyType middlekey = parentPage->KeyAt(parentPage->ValueIndex(GetPageId()) + 1);
      // buffer_pool_manager->UnpinPage(GetParentPageId(), false);
      MappingType moveKV = array[GetSize() - 1];
      IncreaseSize(-1);
      recipient->CopyFirstFrom(moveKV, buffer_pool_manager);
}

/* Append an entry at the beginning.
 * Since it is an internal page, the moved entry(page)'s parent needs to be updated.
 * So I need to 'adopt' it by changing its parent page id, which needs to be persisted with BufferPoolManger
 */
INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_INTERNAL_PAGE_TYPE::CopyFirstFrom(const MappingType &pair, BufferPoolManager *buffer_pool_manager) {
    auto * parentDataPage = buffer_pool_manager->FetchPage(GetParentPageId())->GetData();
    B_PLUS_TREE_INTERNAL_PAGE_TYPE * parentPage = reinterpret_cast<B_PLUS_TREE_INTERNAL_PAGE_TYPE *>(parentDataPage);
    int index = parentPage->ValueIndex(GetPageId());
    SetKeyAt(0, parentPage->KeyAt(index));
    parentPage->SetKeyAt(0, pair.first);
    buffer_pool_manager->UnpinPage(parentPage->GetPageId(), true);
    for(int i = GetSize() - 1; i >= 0; i--){
      array[i + 1] = array[i];
    }
    SetValueAt(0, pair.second);

    auto * childDataPage = buffer_pool_manager->FetchPage(pair.second)->GetData();
    B_PLUS_TREE_INTERNAL_PAGE_TYPE * childPage = reinterpret_cast<B_PLUS_TREE_INTERNAL_PAGE_TYPE*>(childDataPage);
    childPage->SetParentPageId(GetPageId());
    buffer_pool_manager->UnpinPage(childPage->GetPageId(), true);
}               
```





In `b_plus_tree_leaf_page.h`, it defines **BPlusTreeLeafPage**,  it store indexed key and record id togather within leaf page. **Only support unique key**.

Compare to **BPlusTreeInternalPage**, **BPlusTreeLeafPage** is much easier. There is no need to empty the first position.

There is some points

Important Method `KeyIndex`, helper function to find the first index i so that array[i].first > = key. It is like `Lookup` in **BPlusTreeInternalPage**.

But everytime before calling `KeyIndex`, we can do a pre-check to improve efficiency, since the keys conatained in leaf page is sorted. If the searched key is less then the key at left boarder or larger then the key at right boarder. The key does not exist in leaf Page. No need to use binary search.

```c++
if(GetSize() == 0 || comparator(key, KeyAt(0)) < 0 || comparator(key, KeyAt(GetSize()-1)) > 0)
    return false;
```



For the implementation of  **BPlusTreeLeafPage** and **BPlusTreeInternalPage**, we only have to think about how to finish these methods, no need to think about when and where to call this methods. They are the basic component of **BPlusTree**.







































